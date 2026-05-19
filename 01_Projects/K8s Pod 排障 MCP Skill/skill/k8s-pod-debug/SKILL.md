---
name: k8s-pod-debug
description: Use this skill when diagnosing Kubernetes Pod or workload health through a readonly kubectl MCP/CLI flow, especially for requests like checking pod status, CrashLoopBackOff, ImagePullBackOff, Pending, OOMKilled, readiness failures, service-to-pod discovery, or environment-targeted cluster checks using env:region:biz or env_region_biz keys such as uat:union:cg and uat_union_cg.
---

# K8s Pod Debug

User-facing alias: `k8s_pod_debug`. The installable Skill name uses hyphen-case because Codex Skill validation only accepts lowercase letters, digits, and hyphens.

## Core Rule

Use this skill only for readonly diagnosis. Do not run write, mutation, interactive, or tunnel commands.

Treat Kubernetes RBAC and the MCP command allowlist as the hard security boundary. Treat this skill as the operating procedure.

## Inputs

Parse the user request into:

```text
cluster_key = <env>:<region>:<biz> or <env>_<region>_<biz>
env = dev | sit | uat | prod
region = union | edge
biz = cg | yl | mg
target = namespace, workload, service, label value, or pod name
intent = status check, restart diagnosis, startup failure, readiness failure, pending diagnosis, image pull issue, resource issue
```

Examples:

```text
uat_union_cg cgboss
uat:union:cg cgboss
prod:edge:cg cgboss CrashLoopBackOff
```

If the environment is missing or ambiguous, ask the user to clarify before querying a cluster. Never guess `prod`.

## Environment Selection

Resolve both formats to the same cluster key:

```text
uat_union_cg -> uat:union:cg
uat:union:cg -> uat:union:cg
```

Resolve the kubeconfig before querying the cluster:

1. First try the dedicated MCP kubeconfig file at `<user-home>/.mymcps/k8s-pod-debug/config`.
2. If that file does not exist, fall back to `<user-home>/.kube/config`.
3. Always pass the selected kubeconfig explicitly with `--kubeconfig`.
4. Prefer a context whose name or alias matches the requested cluster key, such as `sit_union_cg`.
5. Do not search the Vault or project notes first when the dedicated kubeconfig file exists.

Use readonly kubectl config commands only to confirm context availability, for example:

```bash
kubectl --kubeconfig <selected-config> config current-context
```

If the dedicated kubeconfig file exists but does not contain the requested context, report that mismatch and then check whether the fallback kubeconfig contains the requested context.

If a separate cluster mapping is available from the local project or MCP configuration, it may provide default namespaces and label keys. Expected shape:

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

If there is no mapping for the requested cluster, report the missing key and ask for configuration.

## Allowed Commands

Use only these kubectl command families:

```text
kubectl get
kubectl describe
kubectl logs
kubectl top
kubectl config current-context
```

Always pass the resolved kubeconfig explicitly:

```bash
kubectl --kubeconfig <kubeconfig> get pods -n <namespace> -o wide
```

## Forbidden Commands

Do not run:

```text
kubectl exec
kubectl attach
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

If the user asks for a forbidden action, explain that the skill only performs readonly diagnosis and provide the evidence an operator would need for a manual action.

## Target Discovery

The target may be a namespace, workload, service, label value, or pod name. Discover in this order.

### 1. Treat Target As Namespace

```bash
kubectl --kubeconfig <kubeconfig> get pods -n <target> -o wide
```

If pods are found, continue to Pod Diagnosis.

### 2. Treat Target As Workload

Search deployments, statefulsets, and daemonsets:

```bash
kubectl --kubeconfig <kubeconfig> get deploy,statefulset,daemonset -A
```

Find rows whose name matches the target. Record namespace, kind, and workload name. Then inspect selectors or related pods:

```bash
kubectl --kubeconfig <kubeconfig> describe <kind> <name> -n <namespace>
kubectl --kubeconfig <kubeconfig> get pods -n <namespace> -o wide
```

### 3. Treat Target As Service

```bash
kubectl --kubeconfig <kubeconfig> get svc -A
kubectl --kubeconfig <kubeconfig> describe svc <svc> -n <namespace>
```

Use the service selector to query pods:

```bash
kubectl --kubeconfig <kubeconfig> get pods -n <namespace> -l <selector> -o wide
```

### 4. Treat Target As Label Value

Try configured label keys:

```bash
kubectl --kubeconfig <kubeconfig> get pods -A -l app=<target> -o wide
kubectl --kubeconfig <kubeconfig> get pods -A -l app.kubernetes.io/name=<target> -o wide
kubectl --kubeconfig <kubeconfig> get pods -A -l k8s-app=<target> -o wide
```

If discovery fails, report every path attempted and ask for namespace, workload, service, or label.

## Pod Diagnosis

For each relevant pod, collect:

```bash
kubectl --kubeconfig <kubeconfig> get pod <pod> -n <namespace> -o wide
kubectl --kubeconfig <kubeconfig> describe pod <pod> -n <namespace>
kubectl --kubeconfig <kubeconfig> logs <pod> -n <namespace> --tail=200 --since=30m
```

For multi-container pods:

```bash
kubectl --kubeconfig <kubeconfig> get pod <pod> -n <namespace> -o jsonpath="{.spec.containers[*].name}"
kubectl --kubeconfig <kubeconfig> logs <pod> -n <namespace> -c <container> --tail=200 --since=30m
```

For restarted containers:

```bash
kubectl --kubeconfig <kubeconfig> logs <pod> -n <namespace> -c <container> --previous --tail=200
```

Do not paste long logs. Quote only key error lines and summarize the rest.

## Diagnosis Rules

CrashLoopBackOff:

- Check restart count, last state, exit code, recent logs, probes, and events.
- Common causes: startup crash, bad config, unavailable dependency, strict probe, resource issue.

ImagePullBackOff / ErrImagePull:

- Check image address, pull events, imagePullSecret, registry permission, and tag existence.

OOMKilled:

- Check last state reason, exit code 137, memory limit, node pressure, and memory-related logs.

Pending:

- Check scheduling events, nodeSelector, affinity, taints, tolerations, PVC binding, and resource shortage.

Running but not Ready:

- Check readiness probe, service selector, endpoints, container logs, and dependency errors.

Frequent restarts while currently Running:

- Check restart count, last termination reason, previous logs, event timeline, and recent release/config changes.

## Output Format

Respond in Chinese by default. Lead with the conclusion.

```markdown
结论：

证据：

影响范围：

可能原因：

建议下一步：

风险：

待确认项：
```

When evidence is insufficient, say so directly and list the missing query or missing permission.

## Safety Notes

- Do not read secrets.
- Do not execute commands inside containers.
- Do not mutate workloads.
- Do not assume a production environment if the user did not specify one.
- Logs may contain sensitive data; keep excerpts minimal and redact obvious tokens, phone numbers, passwords, and user identifiers.
