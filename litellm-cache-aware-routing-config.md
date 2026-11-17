# LiteLLM 双集群主备与会话缓存亲和完整配置

新会话优先使用集群 A。当前 Pod 无法完成请求时，先尝试本集群其他 Pod，再切换集群 B，最后使用公网模型。成功切换后，通过会话重新绑定继续使用新目标。客户端自动维护会话 ID，每轮仍发送完整对话历史。

**本方案需要完成端到端验收后上线。** 配置依据 2026 年 11 月 17 日的官方文档与仓库提交 `49b2058c1a7` 核对，该提交用于说明代码依据，不是已验证镜像。本文提供部署文件、环境模板、客户端和验收步骤，尚未在真实集群执行。

## 能力范围

| 场景 | 预期行为 |
|---|---|
| 新会话，A 正常 | 选择 A，再选择一个本地 Pod |
| 同一会话后续请求 | 保持已绑定的集群与 Pod |
| 单 Pod 失败，替代 Pod 成功 | 更新到新 Pod，原 Pod 恢复后保持新绑定 |
| A 无法完成请求，B 成功 | 更新到 B，A 恢复后保持 B |
| A、B 都不可用，公网成功 | 更新到公网，后续保持公网绑定 |
| 新目标也失败 | 在剩余健康候选中继续选择，允许回到已恢复的原目标 |
| 会话绑定过期或丢失 | 重新选择，可能回到 A |

“A 主 B 备”是新会话的优先级。故障期间新建并绑定 B 的会话，A 恢复后也继续使用 B。A 处于共享 cooldown 时，新会话会避开 A。

重新绑定要求原 deployment 已进入 cooldown 或被移除，替代响应成功完成，成功回调处理时原目标仍不可用。并发请求可能短暂访问不同目标。TTL 到期、Redis 不可用、状态丢失或配置变化时，不能保证保持原选择。若需要跨这些情况仍严格固定目标，业务层需要持久化会话路由。

流式响应已经输出后断流，不能保证透明切换并续接原文本。调用方需要处理失败，并决定是否重发完整请求。会话亲和只提高前缀缓存复用机会，不保证缓存命中或固定的 TTFT 收益。

## 部署拓扑和前提

| 位置 | Namespace | 组件 | 对外模型名 |
|---|---|---|---|
| 入口 C | `llm-edge` | LiteLLM C、独立 Redis | `chat` |
| GPU 集群 A | `llm-local` | LiteLLM A、独立 Redis | `local-model` |
| GPU 集群 B | `llm-local` | LiteLLM B、独立 Redis | `local-model` |
| A 和 B 内部 | `model-serving` | 已部署的三个 vLLM Pod | 实际 served model 名 |

C 经 HTTPS 访问 A、B 的代理入口，集群内代理只访问本地 Pod。跨集群无需互通 Pod 网络。C 可以与 A 或 B 共用集群，但使用独立 namespace 和 Redis。同一层的所有代理副本及 worker 共享该层 Redis，三层状态空间相互隔离。

本文假设 vLLM 已按 StatefulSet 部署，名称为 `vllm`，`spec.serviceName` 为 `vllm-headless`，Pod 标签包含 `app: vllm`，服务端口为 8000。每个 Pod 都是完整的独立推理副本。一个分布式模型若由多个协同 Pod 构成，应注册实例服务端点，不能逐计算 Pod 注册。

普通 Deployment 的 Pod 名不稳定，需要另行维护 EndpointSlice 到 LiteLLM deployment 的增删，本文不提供该控制器。模型镜像、权重、GPU 数量和并行参数由现有 vLLM 部署负责。确认当前 vLLM 版本已启用前缀缓存。

本地需要 kubectl、envsubst、Python 3，以及具有 PyYAML 和 OpenAI SDK 的 Python 环境。集群需要默认 StorageClass、兼容下述注解的 nginx Ingress controller、已创建的 TLS Secret。故障验收需要 CNI 支持 NetworkPolicy。

## 固定版本和环境

将随附配置包解压到一个工作目录，或按本文文件名保存代码块。所有示例域名、context、镜像和模型占位值都要替换。LiteLLM 镜像必须包含 `session_affinity_repin_on_failover`，并通过后文普通模型组 order 与会话重新绑定的组合验收。仅启动未报错不足以证明参数生效。三处使用相同的固定镜像 digest。

```bash
export CTX_A='your-gpu-cluster-a-context'
export CTX_B='your-gpu-cluster-b-context'
export CTX_C='your-entry-cluster-context'
export LITELLM_IMAGE='your-registry/litellm@sha256:REPLACE_WITH_VERIFIED_DIGEST'
export REDIS_IMAGE='your-registry/redis@sha256:REPLACE_WITH_VERIFIED_DIGEST'
```

Redis 选择已核验的 7 或更新版本并固定 digest。本文不下载或执行远端安装脚本。

## vLLM 稳定 Pod 地址

Headless Service 配合 StatefulSet 提供逐 Pod DNS 地址。若已有对应 Service，核对 selector、serviceName 和端口后复用。

**`vllm-headless.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vllm-headless
  namespace: model-serving
spec:
  clusterIP: None
  selector:
    app: vllm
  ports:
    - name: http
      port: 8000
      targetPort: 8000
```

```bash
kubectl --context "$CTX_A" apply -f vllm-headless.yaml
kubectl --context "$CTX_B" apply -f vllm-headless.yaml
kubectl --context "$CTX_A" -n model-serving get statefulset vllm -o jsonpath='{.spec.serviceName}{"\n"}'
kubectl --context "$CTX_B" -n model-serving get statefulset vllm -o jsonpath='{.spec.serviceName}{"\n"}'
```

逐个调用 Pod 的 `/v1/models` 和实际推理接口，确认三个实例可独立服务。后续 `openai/` 后面的模型名必须与返回的 served model 名一致。

## A 和 B 的 LiteLLM 配置

每个 Pod 注册为独立 deployment，并保持 `model_info.id` 稳定。这里不启用 `kubernetes_pod_discovery`，因为其重试不会记住成功的新 Pod。逐 Pod 注册才能分别冷却和重新绑定。A、B 复用下面的 ConfigMap，通过各自环境变量传入模型和地址。

**`local-config.yaml`**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: litellm-config
data:
  config.yaml: |
    model_list:
      - model_name: local-model
        litellm_params:
          model: os.environ/VLLM_LITELLM_MODEL
          api_base: os.environ/VLLM_POD_0_API_BASE
          api_key: os.environ/VLLM_API_KEY
          timeout: 45
          stream_timeout: 120
        model_info:
          id: vllm-0
      - model_name: local-model
        litellm_params:
          model: os.environ/VLLM_LITELLM_MODEL
          api_base: os.environ/VLLM_POD_1_API_BASE
          api_key: os.environ/VLLM_API_KEY
          timeout: 45
          stream_timeout: 120
        model_info:
          id: vllm-1
      - model_name: local-model
        litellm_params:
          model: os.environ/VLLM_LITELLM_MODEL
          api_base: os.environ/VLLM_POD_2_API_BASE
          api_key: os.environ/VLLM_API_KEY
          timeout: 45
          stream_timeout: 120
        model_info:
          id: vllm-2
    router_settings:
      routing_strategy: simple-shuffle
      optional_pre_call_checks:
        - session_affinity
      session_affinity_repin_on_failover: true
      deployment_affinity_ttl_seconds: 86400
      num_retries: 2
      allowed_fails: 0
      cooldown_time: 900
      timeout: 180
      redis_host: os.environ/REDIS_HOST
      redis_port: os.environ/REDIS_PORT
      redis_password: os.environ/REDIS_PASSWORD
    general_settings:
      master_key: os.environ/LITELLM_MASTER_KEY
      coordination_redis:
        host: os.environ/REDIS_HOST
        port: os.environ/REDIS_PORT
        password: os.environ/REDIS_PASSWORD
```

**`local-a.env.example`**

```dotenv
REDIS_HOST=redis
REDIS_PORT=6379
VLLM_LITELLM_MODEL=openai/REPLACE_WITH_ACTUAL_SERVED_MODEL
VLLM_POD_0_API_BASE=http://vllm-0.vllm-headless.model-serving.svc.cluster.local:8000/v1
VLLM_POD_1_API_BASE=http://vllm-1.vllm-headless.model-serving.svc.cluster.local:8000/v1
VLLM_POD_2_API_BASE=http://vllm-2.vllm-headless.model-serving.svc.cluster.local:8000/v1
```

**`local-b.env.example`**

```dotenv
REDIS_HOST=redis
REDIS_PORT=6379
VLLM_LITELLM_MODEL=openai/REPLACE_WITH_ACTUAL_SERVED_MODEL
VLLM_POD_0_API_BASE=http://vllm-0.vllm-headless.model-serving.svc.cluster.local:8000/v1
VLLM_POD_1_API_BASE=http://vllm-1.vllm-headless.model-serving.svc.cluster.local:8000/v1
VLLM_POD_2_API_BASE=http://vllm-2.vllm-headless.model-serving.svc.cluster.local:8000/v1
```

A、B 的本地 URL 可以相同，因为在各自集群内部解析。后续统一复制为 `local-a.env`、`local-b.env`，并填写真实的 served model 名和 Pod URL。

## 入口 C 的 LiteLLM 配置

三个目标都使用 `model_name: chat`，属于同一个模型组。`order` 决定新会话优先级，会话亲和先收窄候选，成功重新绑定后可以继续选择 B。不要额外改成 routing_groups 的 priority 策略，它有不同的亲和性处理。

此处不配置跨模型组 fallbacks。跨模型组 fallback 不能被当成原模型组的会话绑定迁移。

**`edge-config.yaml`**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: litellm-config
data:
  config.yaml: |
    model_list:
      - model_name: chat
        litellm_params:
          model: openai/local-model
          api_base: os.environ/CLUSTER_A_API_BASE
          api_key: os.environ/CLUSTER_A_PROXY_KEY
          order: 1
          timeout: 210
          stream_timeout: 240
        model_info:
          id: cluster-a
      - model_name: chat
        litellm_params:
          model: openai/local-model
          api_base: os.environ/CLUSTER_B_API_BASE
          api_key: os.environ/CLUSTER_B_PROXY_KEY
          order: 2
          timeout: 210
          stream_timeout: 240
        model_info:
          id: cluster-b
      - model_name: chat
        litellm_params:
          model: os.environ/PUBLIC_LITELLM_MODEL
          api_base: os.environ/PUBLIC_API_BASE
          api_key: os.environ/PUBLIC_API_KEY
          order: 3
          timeout: 120
          stream_timeout: 120
        model_info:
          id: public-fallback
    router_settings:
      routing_strategy: simple-shuffle
      optional_pre_call_checks:
        - session_affinity
      session_affinity_repin_on_failover: true
      deployment_affinity_ttl_seconds: 86400
      num_retries: 0
      allowed_fails: 0
      cooldown_time: 1800
      timeout: 600
      redis_host: os.environ/REDIS_HOST
      redis_port: os.environ/REDIS_PORT
      redis_password: os.environ/REDIS_PASSWORD
    general_settings:
      master_key: os.environ/LITELLM_MASTER_KEY
      forward_client_headers_to_llm_api: true
      coordination_redis:
        host: os.environ/REDIS_HOST
        port: os.environ/REDIS_PORT
        password: os.environ/REDIS_PASSWORD
```

**`edge.env.example`**

```dotenv
REDIS_HOST=redis
REDIS_PORT=6379
CLUSTER_A_API_BASE=https://llm-a.example.com/v1
CLUSTER_B_API_BASE=https://llm-b.example.com/v1
PUBLIC_LITELLM_MODEL=dashscope/qwen-plus-latest
PUBLIC_API_BASE=https://dashscope.aliyuncs.com/compatible-mode/v1
```

后续统一复制为 `edge.env` 并修改。A、B 地址包含 `/v1`，与两个本地代理提供的 `local-model` 对应。公网示例使用 DashScope 提供方维护的模型别名，部署时按账号区域核对地址、模型可用性和费用。公网模型不一定与本地模型同权重或同上下文长度。若要求同模型兜底，替换成实际支持的对应模型。

公网切换会发送本轮完整请求内容，业务需事先允许这一数据流。

## Secret 文件

三个示例分别复制为同名但去掉 `.example` 的文件。值保持单行，不加引号。入口 `CLUSTER_A_PROXY_KEY` 等于 A 的 `LITELLM_MASTER_KEY`，B 同理。C 的 master key 用于本文管理和验收，不下发普通用户。

Redis 密码在此模板中使用随机十六进制字符串，避免配置文件转义问题。可用 `openssl rand -hex 32` 生成，代理密钥使用 `sk-` 加随机值。实际 vLLM 密钥填入 `VLLM_API_KEY`。

**`local-a.secrets.env.example`**

```dotenv
LITELLM_MASTER_KEY=sk-REPLACE_WITH_A_PROXY_KEY
REDIS_PASSWORD=REPLACE_WITH_A_REDIS_PASSWORD
VLLM_API_KEY=REPLACE_WITH_A_VLLM_API_KEY
```

**`local-b.secrets.env.example`**

```dotenv
LITELLM_MASTER_KEY=sk-REPLACE_WITH_B_PROXY_KEY
REDIS_PASSWORD=REPLACE_WITH_B_REDIS_PASSWORD
VLLM_API_KEY=REPLACE_WITH_B_VLLM_API_KEY
```

**`edge.secrets.env.example`**

```dotenv
LITELLM_MASTER_KEY=sk-REPLACE_WITH_C_PROXY_KEY
REDIS_PASSWORD=REPLACE_WITH_C_REDIS_PASSWORD
CLUSTER_A_PROXY_KEY=sk-REPLACE_WITH_A_PROXY_KEY
CLUSTER_B_PROXY_KEY=sk-REPLACE_WITH_B_PROXY_KEY
PUBLIC_API_KEY=REPLACE_WITH_PUBLIC_API_KEY
```

```bash
cp -n local-a.env.example local-a.env
cp -n local-b.env.example local-b.env
cp -n edge.env.example edge.env
cp -n local-a.secrets.env.example local-a.secrets.env
cp -n local-b.secrets.env.example local-b.secrets.env
cp -n edge.secrets.env.example edge.secrets.env
chmod 600 local-a.secrets.env local-b.secrets.env edge.secrets.env
```

再填写实际值，确认没有残留 REPLACE 占位。密钥文件不提交、不粘贴到工单和日志。配置包附带 `.gitignore` 排除这些文件。

本方案不依赖数据库。如需虚拟密钥、用户权限和预算管理，另行配置数据库及对应能力。

## 每层独立的 Redis

以下模板分别部署到三处。基础配置采用单实例、PVC、AOF 每秒同步，有单点故障和最近写入丢失风险。生产可替换为高可用 Redis，并同步修改 router 与 coordination Redis 配置。本文模板不配置 Redis TLS，若实际服务需要 TLS，应补充对应连接参数后验证。

**`redis.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
spec:
  clusterIP: None
  selector:
    app: routing-redis
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: routing-redis
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis-headless
  replicas: 1
  selector:
    matchLabels:
      app: routing-redis
  template:
    metadata:
      labels:
        app: routing-redis
    spec:
      containers:
        - name: redis
          image: ${REDIS_IMAGE}
          command:
            - sh
            - -ec
            - |
              umask 077
              printf '%s\n' 'bind 0.0.0.0' 'port 6379' 'dir /data' 'appendonly yes' 'appendfsync everysec' "requirepass $REDIS_PASSWORD" > /tmp/redis.conf
              exec redis-server /tmp/redis.conf
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: litellm-secrets
                  key: REDIS_PASSWORD
            - name: REDISCLI_AUTH
              valueFrom:
                secretKeyRef:
                  name: litellm-secrets
                  key: REDIS_PASSWORD
          ports:
            - name: redis
              containerPort: 6379
          readinessProbe:
            exec:
              command: [redis-cli, ping]
            periodSeconds: 5
          livenessProbe:
            exec:
              command: [redis-cli, ping]
            initialDelaySeconds: 20
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 1Gi
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 1Gi
```

## 三处共用的代理工作负载

每层两个副本，每副本两个 worker。配置挂载自 ConfigMap，普通环境来自 `litellm-env`，凭据来自 `litellm-secrets`。资源值是初始值，需要按实际并发调整。生产还应配置副本跨节点分布，避免两个副本落在同一故障域。

**`proxy.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: litellm-proxy
spec:
  selector:
    app: routing-litellm
  ports:
    - name: http
      port: 4000
      targetPort: http
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: litellm-proxy
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: routing-litellm
  template:
    metadata:
      labels:
        app: routing-litellm
    spec:
      terminationGracePeriodSeconds: 660
      containers:
        - name: litellm
          image: ${LITELLM_IMAGE}
          command: [litellm]
          args:
            - --config
            - /config/config.yaml
            - --port
            - "4000"
            - --num_workers
            - "2"
          envFrom:
            - configMapRef:
                name: litellm-env
            - secretRef:
                name: litellm-secrets
          ports:
            - name: http
              containerPort: 4000
          startupProbe:
            httpGet:
              path: /health/liveness
              port: http
            periodSeconds: 5
            failureThreshold: 60
          livenessProbe:
            httpGet:
              path: /health/liveness
              port: http
            periodSeconds: 10
            timeoutSeconds: 3
          readinessProbe:
            httpGet:
              path: /health/readiness
              port: http
            periodSeconds: 5
            timeoutSeconds: 3
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: "2"
              memory: 4Gi
          volumeMounts:
            - name: config
              mountPath: /config
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: litellm-config
```

探针检查代理存活和就绪，不证明后端推理成功。配置更新后显式滚动重启，使各 worker 使用相同配置。

## Ingress 和 TLS

三个入口复用模板。TLS Secret 提前创建在各自 namespace。A、B 的入口只允许 C 所在网络或经过授权的调用方访问，可通过现有网关或防火墙限制。已有统一网关时使用等价配置。

**`ingress.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: litellm-proxy
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "660"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "660"
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - ${PROXY_HOST}
      secretName: ${TLS_SECRET}
  rules:
    - host: ${PROXY_HOST}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: litellm-proxy
                port:
                  number: 4000
```

不启用 Cookie 黏性。会话 ID 和共享 Redis 负责代理之间的会话一致性。这里的 read timeout 是相邻读取超时，不是完整流式输出的总时长上限。

## 部署顺序

先完成环境文件和 Secret 文件，再创建 namespace、配置和 Redis，最后启动代理与入口。下面显式指定 context，不修改全局当前 context。

```bash
kubectl --context "$CTX_A" create namespace llm-local --dry-run=client -o yaml | kubectl --context "$CTX_A" apply -f -
kubectl --context "$CTX_B" create namespace llm-local --dry-run=client -o yaml | kubectl --context "$CTX_B" apply -f -
kubectl --context "$CTX_C" create namespace llm-edge --dry-run=client -o yaml | kubectl --context "$CTX_C" apply -f -

kubectl --context "$CTX_A" -n llm-local create secret generic litellm-secrets --from-env-file=local-a.secrets.env --dry-run=client -o yaml | kubectl --context "$CTX_A" -n llm-local apply -f -
kubectl --context "$CTX_B" -n llm-local create secret generic litellm-secrets --from-env-file=local-b.secrets.env --dry-run=client -o yaml | kubectl --context "$CTX_B" -n llm-local apply -f -
kubectl --context "$CTX_C" -n llm-edge create secret generic litellm-secrets --from-env-file=edge.secrets.env --dry-run=client -o yaml | kubectl --context "$CTX_C" -n llm-edge apply -f -

kubectl --context "$CTX_A" -n llm-local create configmap litellm-env --from-env-file=local-a.env --dry-run=client -o yaml | kubectl --context "$CTX_A" -n llm-local apply -f -
kubectl --context "$CTX_B" -n llm-local create configmap litellm-env --from-env-file=local-b.env --dry-run=client -o yaml | kubectl --context "$CTX_B" -n llm-local apply -f -
kubectl --context "$CTX_C" -n llm-edge create configmap litellm-env --from-env-file=edge.env --dry-run=client -o yaml | kubectl --context "$CTX_C" -n llm-edge apply -f -

kubectl --context "$CTX_A" -n llm-local apply -f local-config.yaml
kubectl --context "$CTX_B" -n llm-local apply -f local-config.yaml
kubectl --context "$CTX_C" -n llm-edge apply -f edge-config.yaml

envsubst '${REDIS_IMAGE}' < redis.yaml > redis.rendered.yaml
envsubst '${LITELLM_IMAGE}' < proxy.yaml > proxy.rendered.yaml

kubectl --context "$CTX_A" -n llm-local apply -f redis.rendered.yaml
kubectl --context "$CTX_B" -n llm-local apply -f redis.rendered.yaml
kubectl --context "$CTX_C" -n llm-edge apply -f redis.rendered.yaml

kubectl --context "$CTX_A" -n llm-local rollout status statefulset/redis --timeout=300s
kubectl --context "$CTX_B" -n llm-local rollout status statefulset/redis --timeout=300s
kubectl --context "$CTX_C" -n llm-edge rollout status statefulset/redis --timeout=300s

kubectl --context "$CTX_A" -n llm-local apply -f proxy.rendered.yaml
kubectl --context "$CTX_B" -n llm-local apply -f proxy.rendered.yaml
kubectl --context "$CTX_C" -n llm-edge apply -f proxy.rendered.yaml

kubectl --context "$CTX_A" -n llm-local rollout status deployment/litellm-proxy --timeout=300s
kubectl --context "$CTX_B" -n llm-local rollout status deployment/litellm-proxy --timeout=300s
kubectl --context "$CTX_C" -n llm-edge rollout status deployment/litellm-proxy --timeout=300s
```

只替换指定镜像变量，保留 Redis 容器运行时的 `$REDIS_PASSWORD`。镜像占位值必须替换为真实 digest。

```bash
export PROXY_HOST='llm-a.example.com'
export TLS_SECRET='llm-a-tls'
envsubst '${PROXY_HOST} ${TLS_SECRET}' < ingress.yaml > ingress-a.rendered.yaml
kubectl --context "$CTX_A" -n llm-local apply -f ingress-a.rendered.yaml

export PROXY_HOST='llm-b.example.com'
export TLS_SECRET='llm-b-tls'
envsubst '${PROXY_HOST} ${TLS_SECRET}' < ingress.yaml > ingress-b.rendered.yaml
kubectl --context "$CTX_B" -n llm-local apply -f ingress-b.rendered.yaml

export PROXY_HOST='llm.example.com'
export TLS_SECRET='llm-edge-tls'
envsubst '${PROXY_HOST} ${TLS_SECRET}' < ingress.yaml > ingress-c.rendered.yaml
kubectl --context "$CTX_C" -n llm-edge apply -f ingress-c.rendered.yaml
```

## 超时和重新绑定

| 参数 | 集群内 A 和 B | 入口 C |
|---|---|---|
| num_retries | 2 | 0 |
| allowed_fails | 0 | 0 |
| cooldown_time | 900 秒 | 1800 秒 |
| 单次 timeout | 45 秒 | A/B 210 秒，公网 120 秒 |
| Router timeout | 180 秒 | 600 秒 |
| stream_timeout | 120 秒 | A/B 240 秒，公网 120 秒 |
| 会话空闲 TTL | 86400 秒 | 86400 秒 |

本地增加最多两次重试，入口通过 order 进入后续层级。`allowed_fails: 0` 使符合冷却条件的首次失败即可触发 cooldown。较长 cooldown 为替代请求完成和回调预留时间，但也让新会话避开该目标更久。

若成功回调执行时原目标的 cooldown 已结束，可能无法重新绑定。尤其长流式输出可能超过 cooldown，需要业务侧限制完整请求时长，让 cooldown 大于最长故障切换链和成功响应完成所需时间。按实际镜像核对 timeout 与 stream_timeout 的作用阶段，不能将 SDK、Router 和 Ingress 的超时简单当作完整流式请求总时长。

这些值是起始配置，需按实际模型首 Token 延迟和输出长度调整。若单次推理经常超过 45 秒，应先调整预算，再做故障验收。

## 自动维护会话的客户端

每段对话生成 UUID，所有轮次携带相同 `x-litellm-session-id`，入口向下游转发。不要同时发送另一个不同的 `x-litellm-trace-id`。

下面只处理非流式普通文本。调用成功后才更新对话，失败不会提前追加用户消息。生产应把 Conversation 保存到业务存储，以便跨进程和重启恢复。工具调用、多模态需要保存和重放完整的相应消息结构。

**`client.py`**

```python
import os
from dataclasses import dataclass
from typing import Final
from uuid import uuid4

from openai import OpenAI
from openai.types.chat import ChatCompletionMessageParam


@dataclass(frozen=True, slots=True)
class Conversation:
    session_id: str
    messages: tuple[ChatCompletionMessageParam, ...] = ()


def new_conversation() -> Conversation:
    return Conversation(session_id=str(uuid4()))


def turn(client: OpenAI, conversation: Conversation, text: str) -> tuple[Conversation, str]:
    user_message: Final[ChatCompletionMessageParam] = {"role": "user", "content": text}
    request_messages: Final = (*conversation.messages, user_message)
    response: Final = client.chat.completions.create(
        model="chat",
        messages=request_messages,
        extra_headers={"x-litellm-session-id": conversation.session_id},
    )
    answer: Final = response.choices[0].message.content or ""
    assistant_message: Final[ChatCompletionMessageParam] = {"role": "assistant", "content": answer}
    updated: Final = Conversation(
        session_id=conversation.session_id,
        messages=(*request_messages, assistant_message),
    )
    return updated, answer


def main() -> None:
    with OpenAI(
        base_url=os.environ["LITELLM_BASE_URL"],
        api_key=os.environ["LITELLM_API_KEY"],
        max_retries=0,
        timeout=650,
    ) as client:
        initial: Final = new_conversation()
        first, answer_one = turn(client, initial, "请记住本次讨论编号是 routing-demo。")
        second, answer_two = turn(client, first, "刚才讨论的编号是什么？")
        print(second.session_id)
        print(answer_one)
        print(answer_two)


if __name__ == "__main__":
    main()
```

```bash
export LITELLM_BASE_URL='https://llm.example.com/v1'
read -r -s LITELLM_API_KEY
export LITELLM_API_KEY
python3 client.py
```

read 等待输入时粘贴 C 的验收密钥，再按回车。SDK 自身自动重试关闭，主要由 Router 控制重试。

## 观察真实绑定

入口响应头可观察入口 deployment，但不能仅凭外层头判断内部 Pod。下面直接读取各层专用 Redis 中对应测试会话的记录，再结合实际推理结果判断。

**`inspect-session.sh`**

```bash
set -euo pipefail
context=${1:?context is required}
namespace=${2:?namespace is required}
session=${3:?session ID is required}

while IFS= read -r key
do
  if [ -n "$key" ]
  then
    printf '%s\n' "$key"
    kubectl --context "$context" -n "$namespace" exec redis-0 -- redis-cli --raw GET "$key"
    kubectl --context "$context" -n "$namespace" exec redis-0 -- redis-cli TTL "$key"
  fi
done < <(kubectl --context "$context" -n "$namespace" exec redis-0 -- redis-cli --scan --pattern "deployment_affinity:v1:session:*:$session")
```

键格式属于版本相关诊断接口，目标镜像不同时按实现调整查询模式。脚本利用容器内 REDISCLI_AUTH，不把 Redis 密码放入命令参数。只在隔离验收环境使用。

```bash
export SESSION_ID="$(python3 -c 'from uuid import uuid4; print(uuid4())')"
export EDGE_URL='https://llm.example.com'
read -r -s EDGE_KEY
export EDGE_KEY

curl --fail-with-body --max-time 650 -sS -D response.headers -o response.json \
  "$EDGE_URL/v1/chat/completions" \
  -H "Authorization: Bearer $EDGE_KEY" \
  -H 'Content-Type: application/json' \
  -H "x-litellm-session-id: $SESSION_ID" \
  -d '{"model":"chat","messages":[{"role":"user","content":"请简短回答 routing-demo。"}]}'

bash inspect-session.sh "$CTX_C" llm-edge "$SESSION_ID"
bash inspect-session.sh "$CTX_A" llm-local "$SESSION_ID"
```

成功回调完成后，C 应显示 cluster-a，A 应显示某个 vllm Pod。串行重复相同会话，应保持绑定。未看到记录时先排查会话头转发、该层 Redis、版本支持和回调。HTTP 200 或回答正确不能单独证明亲和性。

## 故障注入和恢复验收

下述 NetworkPolicy 会真实阻断流量，只用于专门的测试部署。先确认 CNI 支持策略，且没有其他允许规则使隔离无效。删除策略恢复访问，不需要重启模型。

### 单 Pod 切换

先从 A 的绑定确认实际 Pod，再设置 FAILED_POD，不能随意选一个未接收当前会话的 Pod。

**`fail-pod.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: routing-test-block-pod
  namespace: model-serving
spec:
  podSelector:
    matchLabels:
      statefulset.kubernetes.io/pod-name: ${FAILED_POD}
  policyTypes: [Ingress]
  ingress: []
```

```bash
export FAILED_POD='vllm-1'
envsubst '${FAILED_POD}' < fail-pod.yaml > fail-pod.rendered.yaml
kubectl --context "$CTX_A" apply -f fail-pod.rendered.yaml
```

重复上面的同一会话请求，要求成功完成，C 仍绑定 A，A 更新到另一个 Pod。检查更新后解除隔离。

```bash
kubectl --context "$CTX_A" -n model-serving delete networkpolicy routing-test-block-pod
```

等待至少 900 秒，让原 Pod 的 cooldown 结束，再请求相同会话。要求仍使用新 Pod。仅在解除隔离后立刻请求，不能证明恢复后不回切，因为 cooldown 可能尚未结束。等待期间不要继续触发原 Pod 的失败计数。

### 集群 A 切换到 B

下述策略隔离 A 的 LiteLLM 入口，验证 C 无法访问 A 的场景。它不代表覆盖全部 GPU 故障类型，还需测试 A 的模型副本全部不可用、但 A 代理仍能访问的情况。

**`fail-local-proxy.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: routing-test-block-local-proxy
spec:
  podSelector:
    matchLabels:
      app: routing-litellm
  policyTypes: [Ingress]
  ingress: []
```

```bash
kubectl --context "$CTX_A" -n llm-local apply -f fail-local-proxy.yaml
```

用已绑定 A 的同一会话请求 C，要求成功，C 更新为 cluster-b，B 内出现 Pod 绑定。分别查询 C 与 B 的 Redis。

```bash
kubectl --context "$CTX_A" -n llm-local delete networkpolicy routing-test-block-local-proxy
```

等待至少 1800 秒，让 A 在入口的 cooldown 结束。相同会话应继续 B，全新会话应优先 A。确认期间没有其他故障继续延长冷却。

### 公网兜底

```bash
kubectl --context "$CTX_A" -n llm-local apply -f fail-local-proxy.yaml
kubectl --context "$CTX_B" -n llm-local apply -f fail-local-proxy.yaml
```

请求 C，要求公网真实推理成功，C 更新为 public-fallback。恢复 A、B，并在 cooldown 结束后，检查原会话保持公网，新会话优先 A。模拟响应不作为公网验收通过证据。

```bash
kubectl --context "$CTX_A" -n llm-local delete networkpolicy routing-test-block-local-proxy
kubectl --context "$CTX_B" -n llm-local delete networkpolicy routing-test-block-local-proxy
```

### 流式和多个 worker

```bash
curl --fail-with-body --max-time 650 -N -sS \
  "$EDGE_URL/v1/chat/completions" \
  -H "Authorization: Bearer $EDGE_KEY" \
  -H 'Content-Type: application/json' \
  -H "x-litellm-session-id: $SESSION_ID" \
  -d '{"model":"chat","stream":true,"messages":[{"role":"user","content":"请用三句话解释前缀缓存。"}]}'
```

要求事件持续返回，正常流结束后绑定正确。分别验证首 Token 前失败、正常结束和中途断流。

检查每处两个代理副本和每副本两个实际服务 worker 的 PID。对两个副本分别做端口转发，串行发送同一 session，核对共享绑定。只经过 Ingress 重复请求不足以证明覆盖所有副本与 worker。

| 验收项 | 应保存的证据 |
|---|---|
| 版本 | 三层实际镜像 digest、启动参数与 worker PID |
| 正常多轮 | 同一 session 的响应和各层绑定 |
| Pod 失败 | 原 Pod、新 Pod、成功响应、cooldown 结束后绑定 |
| 集群失败 | A 到 B、恢复后同一会话保持 B、新会话选 A |
| 公网 | 真实公网请求成功与绑定 |
| 流式 | 完整结束、首 Token 前失败与中途断流结果 |
| 缓存 | vLLM 版本、缓存开关、Token 前缀、缓存指标与 TTFT |

普通两轮聊天不能证明缓存收益，需要记录并发、前缀长度、缓存压力和测量方法。使用测试内容，保存结果时排除 Authorization、API key 和敏感原始日志。

## 配置更新和清理

ConfigMap 或 Secret 更新后显式重启相应层。下面清理命令只删除本次测试的策略，不清空共享 Redis。

```bash
kubectl --context "$CTX_A" -n llm-local rollout restart deployment/litellm-proxy
kubectl --context "$CTX_B" -n llm-local rollout restart deployment/litellm-proxy
kubectl --context "$CTX_C" -n llm-edge rollout restart deployment/litellm-proxy

kubectl --context "$CTX_A" -n model-serving delete networkpolicy routing-test-block-pod --ignore-not-found
kubectl --context "$CTX_A" -n llm-local delete networkpolicy routing-test-block-local-proxy --ignore-not-found
kubectl --context "$CTX_B" -n llm-local delete networkpolicy routing-test-block-local-proxy --ignore-not-found
```

退出验收后确认三个入口健康，新建会话验证正常路径。Redis PVC 保留已有状态，删除代理工作负载不会清除会话绑定。

## 官方资料

[快速部署](https://docs.litellm.ai/docs/proxy/docker_quick_start)说明基础代理部署和固定生产镜像。

[缓存感知路由](https://docs.litellm.ai/docs/proxy/cache_aware_routing)说明 Pod 发现与两级跨集群结构，也明确重试不会永久迁移 Pod。

[路由配置](https://docs.litellm.ai/docs/routing)说明会话亲和、TTL、成功重新绑定和 deployment order。本文组合这些能力实现主备，仍需所选镜像完成行为验收。

[跨 worker 共享状态](https://docs.litellm.ai/docs/router/learn/shared_state)说明共享 Redis 和状态可见性的边界。

[StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)说明稳定 Pod 标识与 Headless Service 的关系。

[模型别名](https://docs.litellm.ai/docs/model_latest_aliases)说明提供方维护的别名，公网模型和区域需按实际账号确认。
