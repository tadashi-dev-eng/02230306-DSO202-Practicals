# Challenge extension - prediction (written BEFORE rendering the sandbox overlay)

| Field | Base value | Predicted sandbox value | Why |
| --- | --- | --- | --- |
| Deployment `metadata.name` | `webapp` | `sandbox-webapp` | namePrefix renames resources |
| Service `metadata.name` | `webapp` | `sandbox-webapp` | namePrefix renames resources |
| ConfigMap `metadata.name` | `web-content-<hash>` | `sandbox-web-content-<hash>` (same hash) | prefix added before the hash suffix; content unchanged |
| Deployment `volumes[].configMap.name` | `web-content-<hash>` | `sandbox-web-content-<hash>` | reference-aware: Kustomize rewrites references to renamed objects |
| Namespace `metadata.name` | `webapp-sandbox` | `webapp-sandbox` (NOT prefixed) | expected to be exempt because `namespace:` must keep pointing at it - **unsure** |
| `metadata.namespace` on objects | - | `webapp-sandbox` | set by `namespace:` |
| Service `spec.selector` / Deployment `matchLabels` | `app.kubernetes.io/name: webapp` | unchanged | labels are not names; namePrefix does not touch them |
| Container name `nginx`, volume name `content`, port name `http` | - | unchanged | not Kubernetes object names |
