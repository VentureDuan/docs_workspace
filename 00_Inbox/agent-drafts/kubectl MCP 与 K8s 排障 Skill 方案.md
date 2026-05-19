# kubectl MCP 与 K8s 排障 Skill 方案

背景：

希望通过 `MCP + Skill` 的方式，让 Codex 能够使用 `kubectl` 连接 Kubernetes 集群，查看线上或测试环境中的 Pod 运行情况，并辅助排查常见问题。用户当前不希望编写业务代码，因此方案重点放在：安装 `kubectl`、使用运维提供的只读 kubeconfig、通过 Skill 沉淀排障流程和操作边界。

当前环境维度较多，需要支持按集群属性选择目标集群：

- 部署地点：`union`、`edge`
- 环境属性：`dev`、`sit`、`uat`、`prod`
- 业务属性：`cg`、`yl`、`mg`

示例请求：

```text
帮我查看 uat_union_cg【uat:union:cg】环境下 cgboss 的 pod 的运行情况
```

问题：

需要解决的核心矛盾是：让 AI Agent 能高效排查 Kubernetes Pod 问题，同时不能把线上集群暴露成一个无边界的命令执行入口。

具体问题包括：

- 多环境、多业务、多部署地点下，如何准确选择 kubeconfig。
- `cgboss` 这类目标名称可能是 namespace、deployment、service 或 label，不能假设唯一对应关系。
- Skill 可以指导模型行为，但不能作为真正的安全边界。
- 排障需要返回结构化结论，而不是简单粘贴 `kubectl` 输出。
- 线上日志可能包含敏感信息，输出时需要控制摘录范围。

方案：

采用“三层边界”设计：

```text
用户问题
  -> Skill 解析环境、目标和排障意图
  -> MCP / 本地工具受控执行 kubectl
  -> kubeconfig + Kubernetes RBAC 提供硬权限控制
  -> Codex 汇总 Pod / Event / Log / Resource / Owner 信息
  -> 输出结构化诊断结论
```

第一阶段目标定位为：**只读 Kubernetes Pod 排障助手**。

## 1. 安装 kubectl

在运行 Codex 或 MCP 的机器上安装 `kubectl`，并确保版本与集群版本兼容。

基础验证命令：

```bash
kubectl version --client
kubectl config current-context
```

实际执行时不依赖默认 kubeconfig，而是显式传入目标环境对应的 kubeconfig：

```bash
kubectl --kubeconfig <kubeconfig_path> get pods -n <namespace> -o wide
```

## 2. 运维提供只读 kubeconfig

由运维为各环境提供只读权限 kubeconfig。推荐使用专用 ServiceAccount，例如：

```text
mcp-k8s-diagnoser
```

建议允许权限：

- `get/list/watch pods`
- `get/list/watch services`
- `get/list/watch endpoints/endpointslices`
- `get/list/watch deployments/replicasets/statefulsets/daemonsets/jobs`
- `get/list/watch events`
- `get/list/watch nodes`
- `get/list/watch namespaces`
- `get pod logs`
- 可选：`metrics.k8s.io pods/nodes`，用于 `kubectl top`

不建议授权：

- `create/update/patch/delete`
- `pods/exec`
- `pods/attach`
- `pods/portforward`
- `secrets get/list`
- `configmaps get/list`，除非后续确认排查配置问题必须使用

线上环境建议按 namespace 最小化授权，不要默认给全集群只读。

## 3. kubeconfig 环境映射

建议维护独立环境配置，而不是把所有环境写死在 Skill 里。Skill 负责说明解析规则，环境清单由配置文件维护。

示例：

```yaml
clusters:
  uat:union:cg:
    alias:
      - uat_union_cg
    kubeconfig: D:/kubeconfigs/uat_union_cg.kubeconfig
    default_namespaces:
      - cgboss
      - cg-system
      - default
    app_label_keys:
      - app
      - app.kubernetes.io/name
      - k8s-app
```

环境标识格式：

```text
<env>:<region>:<biz>
```

其中：

- `env`：`dev`、`sit`、`uat`、`prod`
- `region`：`union`、`edge`
- `biz`：`cg`、`yl`、`mg`

支持别名：

```text
uat:union:cg
uat_union_cg
```

两者语义等价。

## 4. MCP / 工具层边界

如果使用 MCP 暴露 `kubectl` CLI，MCP 层应该做命令白名单和参数限制。不要暴露无限制 shell。

允许命令：

```text
kubectl get
kubectl describe
kubectl logs
kubectl top
kubectl config current-context
```

禁止命令：

```text
kubectl exec
kubectl delete
kubectl apply
kubectl patch
kubectl edit
kubectl scale
kubectl rollout restart
kubectl port-forward
kubectl get secret
kubectl describe secret
```

建议 MCP 层增加：

- 命令超时，例如 10 到 30 秒。
- 输出大小限制，避免日志过大。
- kubeconfig 路径白名单。
- context / namespace 白名单。
- 审计日志，记录命令、环境、时间、调用原因。

## 5. Skill 触发条件

当用户请求包含以下意图时，触发 Kubernetes Pod 排障 Skill：

- 查看某环境下服务或 Pod 运行情况。
- 排查 Pod 启动失败、重启、不可用。
- 查看某个业务服务在 Kubernetes 中的部署状态。
- 分析 `CrashLoopBackOff`、`ImagePullBackOff`、`Pending`、`OOMKilled`、`Ready` 失败等问题。

典型输入：

```text
帮我查看 uat_union_cg 环境下 cgboss 的 pod 的运行情况
帮我排查 prod:edge:cg 环境 cgboss 为什么一直重启
看一下 sit:union:yl 环境某个服务是不是 Pending
```

## 6. Skill 环境解析规则

收到用户请求后，Skill 先解析：

```text
cluster_key = uat_union_cg
env = uat
region = union
biz = cg
target = cgboss
intent = 查看 pod 运行情况 / 排查 pod 异常
```

然后根据环境配置找到 kubeconfig：

```text
uat_union_cg -> uat:union:cg -> D:/kubeconfigs/uat_union_cg.kubeconfig
```

如果环境标识缺失或无法唯一匹配，必须先向用户确认，不能猜测线上环境。

## 7. 目标发现流程

按照 D 方案设计：`cgboss` 可能同时或分别对应 namespace、deployment、service、label。

Skill 不直接假设目标类型，而是按以下顺序逐层发现。

### 7.1 先当 namespace 查

```bash
kubectl --kubeconfig <kubeconfig> get pods -n <target> -o wide
```

如果 namespace 存在且有 Pod，直接进入 Pod 诊断流程。

### 7.2 再当 workload 查

在默认 namespace 范围或全 namespace 范围内查找 deployment、statefulset、daemonset：

```bash
kubectl --kubeconfig <kubeconfig> get deploy,statefulset,daemonset -A
```

从输出中定位名称包含 `<target>` 的 workload。找到后记录：

- namespace
- workload kind
- workload name

再通过 label 或 owner reference 查关联 Pod。

### 7.3 再当 service 查

```bash
kubectl --kubeconfig <kubeconfig> get svc -A
```

如果找到 service，再查看 selector：

```bash
kubectl --kubeconfig <kubeconfig> get svc <svc> -n <namespace> -o yaml
```

根据 service selector 查询 Pod。

### 7.4 最后按常见 label 查

依次尝试配置中的 label key：

```bash
kubectl --kubeconfig <kubeconfig> get pods -A -l app=<target> -o wide
kubectl --kubeconfig <kubeconfig> get pods -A -l app.kubernetes.io/name=<target> -o wide
kubectl --kubeconfig <kubeconfig> get pods -A -l k8s-app=<target> -o wide
```

如果仍然无法定位，输出已尝试路径，并要求用户补充 namespace、deployment、service 或 label。

## 8. Pod 诊断流程

定位到 Pod 后，统一采集：

```bash
kubectl --kubeconfig <kubeconfig> get pods -n <namespace> -o wide
kubectl --kubeconfig <kubeconfig> describe pod <pod> -n <namespace>
kubectl --kubeconfig <kubeconfig> logs <pod> -n <namespace> --tail=200 --since=30m
```

如果是多容器 Pod，先查容器列表：

```bash
kubectl --kubeconfig <kubeconfig> get pod <pod> -n <namespace> -o jsonpath="{.spec.containers[*].name}"
```

再针对异常容器查询日志：

```bash
kubectl --kubeconfig <kubeconfig> logs <pod> -n <namespace> -c <container> --tail=200 --since=30m
```

如需查看上一轮崩溃日志：

```bash
kubectl --kubeconfig <kubeconfig> logs <pod> -n <namespace> -c <container> --previous --tail=200
```

## 9. 常见异常判断规则

### CrashLoopBackOff

重点查看：

- restart count
- last state
- exit code
- recent logs
- liveness / readiness / startup probe
- event 时间线

判断方向：

- 应用启动即退出。
- 配置错误。
- 依赖服务不可用。
- 探针配置过严。
- 资源不足导致进程异常退出。

### ImagePullBackOff / ErrImagePull

重点查看：

- image 地址
- pull event
- imagePullSecret
- 仓库权限
- 镜像 tag 是否存在

### OOMKilled

重点查看：

- last state reason
- exit code 137
- memory limit
- node memory pressure
- 日志中是否有内存峰值或大对象处理痕迹

### Pending

重点查看：

- scheduling event
- nodeSelector
- affinity / anti-affinity
- taint / toleration
- PVC 绑定状态
- CPU / Memory 资源不足

### Running 但 Ready 异常

重点查看：

- readiness probe
- service selector
- endpoints / endpointslices
- 容器日志
- 依赖服务连通性

### 频繁重启但当前 Running

重点查看：

- restart count
- last termination reason
- 最近 event
- `--previous` 日志
- 重启时间线是否与发布、配置变更、下游故障吻合

## 10. 输出模板

每次诊断输出固定结构：

```markdown
结论：

证据：

影响范围：

可能原因：

建议下一步：

风险：

待确认项：
```

示例：

```markdown
结论：
cgboss 在 uat:union:cg 环境中存在 2 个异常 Pod，其中 1 个 CrashLoopBackOff，1 个 Ready 失败。

证据：
- Pod xxx restart count = 12
- Last State = Terminated, Reason = Error, ExitCode = 1
- 最近日志显示配置连接失败
- Event 中出现 readiness probe failed

影响范围：
当前 cgboss 至少有 1 个副本不可用，需要结合期望副本数判断是否影响整体服务。

可能原因：
应用启动依赖的配置或下游服务不可用。

建议下一步：
优先确认配置中心和下游服务地址是否正确。如需重启或变更配置，必须由人工执行，当前 Skill 不执行写操作。

风险：
日志可能包含敏感信息，输出时只摘取关键错误行。

待确认项：
需要确认 cgboss 的期望副本数、最近是否有发布或配置变更。
```

为什么这么做：

这个方案把“硬权限”和“软流程”分开：

- kubeconfig + Kubernetes RBAC 是真正的安全边界。
- MCP 层命令白名单是第二层保护，防止模型调用危险 kubectl 子命令。
- Skill 是排障 SOP，负责让模型知道如何选环境、如何发现目标、如何组织诊断证据。

这样可以避免直接开发复杂 MCP 服务，同时保留后续演进空间。第一版可以快速落地为 `kubectl + kubeconfig + Skill`；后续如果排障场景稳定，再把常用流程沉淀为专用 MCP 工具，例如：

- `diagnose_pod`
- `get_workload_context`
- `find_service_pods`
- `collect_pod_failure_evidence`

踩坑：

- 不要把 Skill 当作权限系统。Skill 只能约束模型行为，不能防止工具被滥用。
- 不要默认使用当前 kubeconfig context，必须显式指定 kubeconfig。
- 不要默认查 `prod`，线上环境必须由用户明确指定。
- 不要默认读取 secret。
- 不要大段输出日志，线上日志可能包含 token、手机号、用户 ID、内部地址等敏感信息。
- 不要只看 Pod 当前状态，很多问题需要结合 `describe` event 和 `--previous` 日志。
- 不要假设业务名等于 namespace，目标发现必须兼容 namespace、workload、service、label 多种情况。

下次怎么复用：

可以复用为一个 Kubernetes 排障 Skill：

```text
当用户要求查看或排查 Kubernetes 中某个服务、Pod、workload 的运行情况时：
1. 解析环境标识，支持 env:region:biz 和 env_region_biz。
2. 根据环境配置选择 kubeconfig。
3. 确认只读排障意图。
4. 按 namespace -> workload -> service -> label 顺序发现目标。
5. 收集 pod、describe、event、logs、resource 信息。
6. 按异常类型匹配诊断规则。
7. 输出结论、证据、影响范围、可能原因、建议下一步、风险、待确认项。
8. 禁止执行写操作和危险操作。
```

相关链接：

- 暂无。

来源：

- 用户在当前对话中提出：希望制作一个 MCP，通过 `kubectl` 连接 Kubernetes 集群，查看 Pod 信息，排查线上问题。
- 用户确认目标为 B：只读基础上的排障助手。
- 用户确认不希望写代码，希望通过 `skill + MCP` 实现。
- 用户补充环境维度：部署地点 `union/edge`，环境属性 `dev/sit/uat/prod`，业务属性 `cg/yl/mg`。
- 用户确认目标发现按 D 设计：namespace、deployment、service、label 都可能存在。
- 当前 Vault 上下文：用户偏好 Go、后端、微服务、MySQL、Redis、MQ；输出先结论再细节；技术方案需说明风险和迁移成本。

假设：

- 运维可以提供各环境只读 kubeconfig。
- 运行 Codex/MCP 的机器可以安装 `kubectl`。
- 第一阶段只做只读排障，不做变更操作。
- 环境配置文件可以由用户或运维维护。
- `kubectl top` 是否可用取决于集群是否安装 metrics-server 以及 kubeconfig 是否有 metrics 权限。
- 目前只覆盖 Kubernetes 侧诊断，不包含日志平台、Trace 平台、监控告警平台联动。

待确认项：

- kubeconfig 文件的实际存放路径和命名规范。
- 是否需要支持跨集群批量查询。
- 各业务常见 namespace、label key、service 命名规范。
- 是否允许读取 ConfigMap。
- 是否需要接入日志平台或 Trace 平台补充 Kubernetes 外部证据。
- MCP 层是使用已有通用 CLI MCP，还是需要单独配置一个 kubectl 专用 MCP。
- 审计日志保存在哪里，保留多久，是否需要脱敏。
