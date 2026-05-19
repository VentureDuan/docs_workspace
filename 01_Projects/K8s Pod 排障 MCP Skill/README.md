# K8s Pod 排障 MCP Skill

## 结论

本项目用于沉淀一个“只读 kubectl MCP + Kubernetes Pod 排障 Skill”的方案。第一阶段不编写业务代码，先通过 `kubectl`、运维提供的只读 kubeconfig、MCP 命令边界和 Skill 排障 SOP，实现面向多环境 Kubernetes 集群的 Pod 状态查看和问题定位。

## 目标

- 支持通过环境标识选择集群，例如 `uat_union_cg` 或 `uat:union:cg`。
- 支持目标名称不确定的发现流程：namespace、workload、service、label 都可能对应同一个业务名。
- 支持只读查看 Pod、Event、Log、Resource、Owner 等排障证据。
- 输出结构化诊断结论，而不是直接粘贴 kubectl 原始输出。
- 禁止线上变更操作，避免把 MCP 做成无边界 shell。

## 非目标

- 第一阶段不执行 `exec`、`delete`、`apply`、`patch`、`scale`、`rollout restart` 等操作。
- 第一阶段不读取 Secret。
- 第一阶段不接入日志平台、Trace 平台或监控平台。
- 第一阶段不开发 Go/client-go 版本 MCP 服务。

## 目录结构

```text
K8s Pod 排障 MCP Skill/
  README.md
  项目概览.md
  skill/
    k8s-pod-debug/
      SKILL.md
      agents/openai.yaml
      references/
        environment-config.example.yaml
        readonly-rbac-requirements.md
```

## 使用方式

Skill 真实名称为 `k8s-pod-debug`。用户侧可以把它称为 `k8s_pod_debug`，但 Codex Skill 的可安装名称必须使用连字符。

1. 本机安装 `kubectl`。
2. 运维提供各环境只读 kubeconfig。
3. 在 MCP 或本地配置中维护环境映射。
4. 调用 Skill 时在用户请求中明确环境和目标。

示例：

```text
帮我查看 uat_union_cg【uat:union:cg】环境下 cgboss 的 pod 的运行情况
```

Skill 会解析为：

```text
env = uat
region = union
biz = cg
target = cgboss
```

然后按 namespace -> workload -> service -> label 的顺序发现目标，并采集只读诊断信息。

## 安全边界

硬边界：

- Kubernetes RBAC
- 只读 kubeconfig
- MCP 命令白名单

软边界：

- Skill 中的操作规程
- 禁止命令清单
- 输出脱敏和日志摘录要求

不要把 Skill 当作权限系统。真正不能执行的动作必须由 kubeconfig 权限和 MCP 工具层限制。

## 迁移成本

当前方案迁移成本较低，因为第一阶段只依赖 `kubectl` 和配置文件。后续如果排障路径稳定，可以逐步迁移为专用 MCP 工具，例如：

- `diagnose_pod`
- `find_service_pods`
- `collect_pod_failure_evidence`
- `get_workload_context`

迁移到 Go/client-go MCP 时，Skill 中的排障流程和输出模板仍可复用。

## 风险

- 如果 MCP 暴露无限制 shell，会绕过 Skill 约束。
- 如果 kubeconfig 权限过大，模型误调用或人为误用都会扩大风险。
- 如果环境映射不准确，可能查错集群。
- 如果日志直接输出，可能泄露敏感信息。
- 如果只看 Pod 当前状态，可能漏掉上一轮崩溃原因。

## 待确认

- kubeconfig 实际存放路径和命名规范。
- 各业务的 namespace、service、deployment、label 命名规则。
- 是否允许读取 ConfigMap。
- 是否需要接入 metrics-server 支持 `kubectl top`。
- MCP 使用已有 CLI 工具还是单独配置 kubectl 专用 MCP。
