# Readonly RBAC Requirements

## Recommended Permissions

Allow:

- `get/list/watch pods`
- `get/list/watch services`
- `get/list/watch endpoints/endpointslices`
- `get/list/watch deployments/replicasets/statefulsets/daemonsets/jobs`
- `get/list/watch events`
- `get/list/watch nodes`
- `get/list/watch namespaces`
- `get pod logs`
- Optional: `metrics.k8s.io pods/nodes` for `kubectl top`

Avoid:

- `create/update/patch/delete`
- `pods/exec`
- `pods/attach`
- `pods/portforward`
- `secrets get/list`
- `configmaps get/list`, unless diagnosis explicitly requires it and the risk is accepted

## Operational Notes

- Use a dedicated ServiceAccount such as `mcp-k8s-diagnoser`.
- Prefer namespace-scoped permissions for production.
- Rotate kubeconfig periodically.
- Keep audit logs for MCP invocations.
- Do not rely on Skill instructions as the only safety control.
