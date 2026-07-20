---
title: "Why Simple Kind"
date: 2026-07-20
draft: false
tags: ["k8s"]
collections: ["Kubernetes Operators"]
weight: 2
summary: "为什么 Kubernetes 需要 Simple kind。"
ShowToc: true
TocOpen: true
ShowReadingTime: true
---

## 一句话回答

我们和 Kubernetes 交互时，有很多需求并不对应任何 object，也不存储到 etcd：

- 查 `/status` —— 想拿 apiserver 实时算出来的状态
- 调 `/scale` —— 想用一个 PUT 改副本数，但 scale 本身不是独立资源
- `kubectl exec` / admission webhook —— 传参数进去，拿结果回来，都不存 etcd

问题是 Kubernetes 有一条硬规定：**任何 API 响应都必须自带类型标签（`apiVersion` + `kind`）**，客户端才能统一解码。上面这些响应既然不是 Object（没存 etcd），又得有 `kind`，那就只能发明第三种类型——**Simple kind**。

## 一、三种 Kind：回顾

Kubernetes 把 API 类型分为三类：

| 类型 | 存 etcd？ | 有 UID？ | 能 `kubectl get`？ | 例子 |
|------|----------|---------|-------------------|------|
| **Object** | ✅ 持久化 | ✅ | ✅ | Pod, Deployment, Service |
| **List** | ❌ 计算生成 | ❌ | ✅（查询时） | PodList, DeploymentList |
| **Simple** | ❌ | ❌ | ❌（不能独立查询） | Status, Scale, Binding |

**为什么叫 "Simple"？** 名字指的是**元数据简单**，不是"简单/容易"的意思。Kubernetes [原始 API conventions 文档（2015）](https://github.com/kubernetes/kubernetes/blob/release-1.2/docs/devel/api-conventions.md#types-kinds) 是这么定义的：

> Simple kinds are used for specific actions on objects and for non-persistent entities. Given their limited scope, they have the same set of limited common metadata as lists.

两个关键点：

1. **用于"对 object 的特定操作"和非持久化实体** —— 这就是本文反复强调的"对应一个 action"的官方出处
2. **和 List 一样只有有限的 common metadata** —— 这就是上表里 Simple "没有 UID" 的根本原因。Object 有完整 `ObjectMeta`（带 UID、resourceVersion、generation、labels、annotations...），Simple 和 List 只有精简版 `ListMeta`，根本不承载身份字段

所以"Simple"这个名字本质上是相对于 Object 的"元数据更简单"——没有 UID、没有 resourceVersion、没有 labels/annotations，因为它们不需要身份，也不被持久化。

## 二、直观对比：Object vs Simple

**Object — 有身份的实体，存在 etcd 里：**

```bash
$ kubectl get deployments nginx -o json
```
```json
{
  "kind": "Deployment",
  "apiVersion": "apps/v1",
  "metadata": {
    "uid": "abc-123-def",
    "resourceVersion": "456789"
  },
  "spec": { "replicas": 3 }
}
```

Deployment 是存在 etcd 里的，你删了它才消失。

**Simple — 不对应一个 object，而对应一个操作（action）：**

```bash
# 场景：HPA 想知道 nginx 这个 Deployment 现在几个副本，好决定要不要扩缩容
# 它不会去拉整个 Deployment 对象，只打 /scale 这个 subresource：
#   GET /apis/apps/v1/namespaces/default/deployments/nginx/scale
# 注意：没有 `kubectl get scale` 这种命令，Scale 不是可独立列出的资源。
```
```json
{
  "kind": "Scale",
  "apiVersion": "autoscaling/v1",
  "spec": {
    "replicas": 3
  },
  "status": {
    "replicas": 3,
    "selector": "app=nginx"
  }
}
```

Scale 不存 etcd，它的 `spec.replicas` 和 `status` 就是底层 Deployment 里对应字段的镜像——apiserver 收到请求时才从 Deployment 取出来填进去返回，并不持久化。

## 三、为什么一定要包一个 Kind？

看完上面的对比，一个自然的问题：

> "/scale 端点直接返回 `{"replicas": 3}` 不行吗？为什么要包成 `kind: Scale`？"

**因为 Kubernetes 有一个全局约定：每一个通过 API 收发的数据，都必须有 `apiVersion` + `kind`。**

这不是"好不好看"的问题，而是整个系统的基石——所有客户端、所有工具链都依赖这个约定来工作。

### 原因 1：类型识别 → 自动反序列化

想象 kubectl 收到一串 JSON payload。传统 REST 可以靠"我请求的是哪个 URL"来推断这是什么对象，但 Kubernetes 不行——**因为 CRD 的存在**：用户可以随时装一个 `S3Bucket`、`MySQLCluster` 之类的自定义资源，kubectl 编译时根本不知道这些类型存在，没法预置一张"URL → Go struct"的静态映射表。

所以 Kubernetes 走的是**动态分发**：读 payload 里的 `kind` 字段，再去 scheme 里查"这个 GVK 对应哪个 Go struct"，最后用那个 struct 解码。这条路径对内置类型和 CRD 是同一套代码——kubectl 不会为 `/scale` 这个 URL 专门写一个解码分支，而是统一靠 `kind` 字段说话。

这条路要跑通，就要求**所有 API 收发的数据必须自带 `kind`，并且结构和标准资源保持一致**——哪怕它只是 `/scale` 这种"操作型"响应也不例外。如果 `/scale` 直接返回 `{"replicas": 3}` 这种不带 `kind` 的 JSON，generic 解码器读不到 `kind`，整条链路立刻断掉。

```
kubectl → GET /scale → { "kind": "Scale", ... } → 读 kind=Scale → 查 scheme → 用 Scale 的 Go struct 解码
kubectl → GET /pod   → { "kind": "Pod", ... }   → 读 kind=Pod   → 查 scheme → 用 Pod 的 Go struct 解码
```

Kubernetes 的所有语言客户端（Go、Python、Java…）都靠这套 GVK 路由做序列化/反序列化，没有 `kind` 客户端就只能猜"这串 JSON 是什么"。

而且这不只是写文档的约定，是被 Go 类型系统强制的——所有 Kubernetes API 对象都必须实现这个接口：

```go
type Object interface {
    GetObjectKind() schema.ObjectKind  // 返回 GVK
}
```

编解码器拿到一个不实现这个接口的对象，连编译都过不了。所以"自带 GVK"不是建议，是机制层面的硬规定。

### 原因 2：跨资源统一接口（HPA 是最佳案例）

先解释 HPA 是什么。**HPA（HorizontalPodAutoscaler）** 是 Kubernetes 内置的一个控制器，负责根据负载自动水平扩缩容——你 CPU 涨了它就加 Pod，闲下来了它就减 Pod。使用方式长这样：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: nginx-hpa }
spec:
  scaleTargetRef:          # ← 指向"我要扩缩容谁"
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

注意 `scaleTargetRef`——HPA 扩缩容的"目标"是一个**任意资源**。它可以是 Deployment，也可以是 StatefulSet、ReplicaSet，甚至可以是用户自己写的 CRD（比如一个 `S3Bucket` operator 想让自己的"bucket 副本数"被 HPA 弹性伸缩）。

那 HPA 控制器怎么跟这么多不同类型的 target 打交道？它每几秒做一次这种事：

1. **读当前副本数** —— GET target 的 `/scale` subresource
2. 拉 metrics（CPU / 自定义指标）
3. 算出新的期望副本数
4. **写回新副本数** —— PUT target 的 `/scale` subresource

```
HPA → GET /apis/apps/v1/.../deployments/nginx/scale    → kind: Scale, status.replicas: 3
HPA → GET /apis/apps/v1/.../statefulsets/kafka/scale   → kind: Scale, status.replicas: 5
HPA → GET /apis/my.io/v1/.../s3buckets/mine/scale      → kind: Scale, status.replicas: 2
```

关键点：**HPA 全程只跟 `kind: Scale` 这个 Simple kind 打交道，从来不碰底层 Deployment / StatefulSet / S3Bucket 的完整对象。**

为什么这能成立？因为 `/scale` 是一份**契约**：任何资源想被 HPA 管理，就开放 `/scale` 端点，同意返回 `kind: Scale`，并且 apiserver（内置资源）或 operator 作者（CRD）负责把 Scale 的 `spec.replicas` / `status.replicas` 映射到底层对象的对应字段。

```
HPA 视角：       GET /scale → kind: Scale → 读 status.replicas / 写 spec.replicas → PUT /scale
                              ↑
                      这一层是契约，固定不变

资源侧视角：     Scale 字段 ↔ 底层对象字段的映射
                              ↑
                      这一层每个资源自己实现，HPA 不关心
```

对 Deployment，apiserver 把 Scale.spec.replicas 映射到 Deployment.spec.replicas。对 StatefulSet，映射到 StatefulSet.spec.replicas。对一个想被 HPA 弹性伸缩的 S3Bucket CRD，operator 作者自己在 handler 里决定"副本数"是什么意思（也许是 bucket 的镜像数），然后把 Scale 字段映射过去。

**翻译工作放在资源那一边，不放在 HPA 这一边。** 如果每个资源返回自己的格式（Deployment 返回 `{"replicas": 3}`，StatefulSet 返回 `{"desiredReplicas": 5}`，S3Bucket 返回 `{"mirrorCount": 2}`），HPA 就要为每一种资源写一套解析逻辑，每来一个新 CRD 都得改 HPA 代码。但因为 `/scale` 端点统一返回 `kind: Scale`，HPA 只需要懂一个类型——你可以把 Scale 理解成一个 Go interface，HPA 依赖这个 interface，每个资源各自实现它，HPA 不知道也不需要知道具体类型。

**同样的模式还在这些地方出现：**

- **`Binding`（scheduler 用）** — 调度器决定 Pod 跑哪个 Node 后，POST 一个 `Binding` 给 apiserver 说"pod X 绑到 node Y"。调度器不修改 Pod 对象、也不需要懂 Node 的完整 schema，只发一个固定形状的 Binding 消息。
- **`Eviction`（kubectl drain 用）** — 驱逐 Pod 时走 `/eviction` subresource，POST 一个 `Eviction`。调用方不用管 Pod 删除的具体细节（grace period、finalizer 处理），由 apiserver 统一处理。
- **`AdmissionReview`（准入 webhook）** — webhook 收到的请求是固定形状的 AdmissionReview，里面的 `request.object` 是 raw bytes。webhook 不需要知道这是 Deployment 还是 S3Bucket，它只对 AdmissionReview 这个固定结构做处理、回 allow/deny。一个 webhook 能同时拦几十种 CRD，就靠这个抽象。

共同模式：**Simple kind 充当两边都不想懂对方 internals 时的"统一接口"**。这就是 Simple kind 的抽象价值——让不同资源的同一类操作共享同一个响应类型。

## 四、Kubernetes 中所有 Simple Kind 一览

### 第一类：错误/状态响应

| Kind | API Group | 场景 |
|------|-----------|------|
| `Status` | core/v1 | API 出错时 apiserver 返回的统一错误格式（你每次 `kubectl apply` 报错都在用） |

### 第二类：Subresource 操作的请求/响应

| Kind | API Group | 场景 |
|------|-----------|------|
| `Scale` | autoscaling/v1 | `/scale` 端点，HPA 拿副本数 |
| `Binding` | core/v1 | POST `/bindings`，scheduler 告诉 apiserver "Pod 绑哪个 Node" |
| `PodExecOptions` | core/v1 | `kubectl exec` — WebSocket 连接参数 |
| `PodAttachOptions` | core/v1 | `kubectl attach` — 连接参数 |
| `PodPortForwardOptions` | core/v1 | `kubectl port-forward` — 连接参数 |
| `PodProxyOptions`、`NodeProxyOptions`、`ServiceProxyOptions` | core/v1 | kubectl proxy 连接参数 |

### 第三类：Review 模式（POST 请求 → 拿结果，不存 etcd）

| Kind | API Group | 场景 |
|------|-----------|------|
| `TokenReview` | authentication.k8s.io/v1 | 验证 token 是否有效 |
| `SubjectAccessReview` | authorization.k8s.io/v1 | 判断某个用户能否执行某操作 |
| `SelfSubjectAccessReview` | authorization.k8s.io/v1 | 判断"我"能否执行 |
| `LocalSubjectAccessReview` | authorization.k8s.io/v1 | 同上，带 namespace 范围 |
| `SelfSubjectRulesReview` | authorization.k8s.io/v1 | 查询"我有哪些权限" |

### 第四类：Webhook 请求/响应

| Kind | API Group | 场景 |
|------|-----------|------|
| `AdmissionReview` | admission.k8s.io/v1 | 准入 Webhook 的请求/响应格式 |
| `ConversionReview` | apiextensions.k8s.io/v1 | CRD 版本转换 Webhook 的请求/响应格式 |

## 五、如何使用 Simple Kind？

### CRD 的限制：能用，不能定义

CRD 机制有一条核心限制：**它只允许你使用 Simple Kind，不允许你定义自己的 Simple Kind。** 你定义 CRD 时，只能定义一个 Object 类型，List 自动派生，Simple 不可自定义：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
spec:
  names:
    kind: MyKind              # ← 你定义一个 Object kind
    listKind: MyKindList      # ← apiserver 自动生成
  versions:
    - name: v1
      schema:
        openAPIV3Schema: ...  # ← 只能描述 MyKind 的字段
```

没有"这是我的 Simple kind"这个选项。

### 但 CRD 可以通过 Subresource 用到 Simple Kind

虽然不能定义 Simple Kind，CRD 可以**启用 subresource**，从而让 apiserver 替你产出内置的 Simple Kind 响应。下面是一个完整例子：一个 `S3Bucket` CRD 同时启用 `/status` 和 `/scale`：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: s3buckets.my.io
spec:
  group: my.io
  names:
    kind: S3Bucket
    listKind: S3BucketList
    plural: s3buckets
    singular: s3bucket
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:               # ← operator 读这个字段作为"副本数"
                  type: integer
                bucketName:
                  type: string
            status:                      # ← /status subresource 暴露这一段
              type: object
              properties:
                replicas:
                  type: integer
                ready:
                  type: boolean
                labelSelector:
                  type: string
      # ↓↓↓ 关键：开启 subresource
      subresources:
        status: {}                       # 启用 /status
        scale:                           # 启用 /scale，并告诉 apiserver Scale 字段 ↔ 底层字段的映射
          specReplicasPath: .spec.replicas
          statusReplicasPath: .status.replicas
          labelSelectorPath: .status.labelSelector
```

启用这两个 subresource 之后：

- **`/status`** —— 客户端 `GET .../s3buckets/x/status` 只拿到 status 部分；写 status 时也只能 PUT 到 `/status`，不会误改 spec。出错时 apiserver 返回 `kind: Status`。
- **`/scale`** —— HPA 现在可以把 `S3Bucket` 当扩缩容目标了。HPA `GET .../s3buckets/x/scale` 拿到的是 `kind: Scale`，apiserver 按上面 `specReplicasPath` / `statusReplicasPath` 的配置，把 Scale 字段映射到 `S3Bucket.spec.replicas` 和 `.status.replicas`。**你没有定义 Scale**，它是 K8s 内置的 Simple Kind，你只是打开了开关。

再配一个 admission webhook 的话，你的 webhook 收到的请求体就是 `kind: AdmissionReview`——同样是内置 Simple Kind，你只是消费者。

**关键心智模型：CRD 用户始终是 Simple Kind 的"消费者"，不是"生产者"。** 你能做的是打开 subresource 开关、写 webhook 处理逻辑；你不能发明一个新的 Simple Kind。

### 什么时候需要"自己定义 Simple Kind"

只有当你不写 CRD，而是**自己实现一个 Aggregated API Server**（用 `apiserver-builder` 或直接基于 `k8s.io/apiserver`）时，才能定义自己的 Simple Kind。因为这时候你完全控制每个 subresource 的 handler，可以返回任意形状的响应。

典型场景：

- **自定义 Subresource** —— 给一个资源加 `/metrics`、`/health` 等端点，返回精简的临时数据
- **Review 模式** —— 实现自定义认证/鉴权，类似 `TokenReview`
- **操作绑定** —— 实现类似 scheduler `Binding` 的请求/响应

举个具体例子：你写一个 aggregated apiserver 管理数据库集群，给 `Database` 资源加一个 `/snapshot` subresource，客户端 POST 一次触发快照并拿回快照 ID。这时你定义两个 Simple Kind：

```go
// 请求体：客户端 POST 过来
type DatabaseSnapshotRequest struct {
    metav1.TypeMeta   `json:",inline"`
    // 可以带参数，比如快照标签
    Labels map[string]string `json:"labels,omitempty"`
}

// 响应体：handler 返回
type DatabaseSnapshotResponse struct {
    metav1.TypeMeta   `json:",inline"`
    SnapshotID string `json:"snapshotID"`
    CreatedAt  time.Time `json:"createdAt"`
}
```

handler 收到 POST 后执行快照、填响应返回——这两个对象都不存 etcd，只是一次操作的消息载体。这就是自定义 Simple Kind 的真实场景。

## 六、总结

实操上记住一条线：**写 CRD 时你是 Simple kind 的消费者**（打开 `/status`、`/scale` subresource 就用上了内置的 Scale/Status，写 webhook 就用上了 AdmissionReview）；**写 aggregated apiserver 时你才是 Simple kind 的生产者**（可以定义自己的 subresource 响应类型）。

---

*参考来源*

- [Kubernetes API Conventions — Types (Kinds)（2015 原始版本，v1.2 时期）](https://github.com/kubernetes/kubernetes/blob/release-1.2/docs/devel/api-conventions.md#types-kinds) — "Simple" 的原始定义出处
- [Kubernetes API Conventions — Types (Kinds)（现行版本）](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-architecture/api-conventions.md#types-kinds)
- [Kubernetes API — Subresources](https://kubernetes.io/docs/reference/using-api/api-concepts/#subresources)
- [HPA 如何使用 Scale subresource](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Admission Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
