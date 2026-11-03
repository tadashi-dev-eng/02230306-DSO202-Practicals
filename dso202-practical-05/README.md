# DSO202 Practical 5 — Environment-Specific Configuration with Kustomize

One base (Deployment + Service + generated ConfigMap) and five overlays (`dev`, `staging`, `prod`, `qa`, `sandbox`). No overlay copies `deployment.yaml` or `service.yaml`; every environment difference is a few lines of Kustomize.

## Layout

```text
dso202-practical-05/
├── README.md
├── cluster/kind-cluster.yaml             # 3-node kind cluster "dso202"
├── examples/webapp/
│   ├── base/                             # deployment.yaml, service.yaml, index.html, kustomization.yaml
│   └── overlays/
│       ├── dev/                          # webapp-dev,     1 replica,  nginx:1.31-alpine
│       ├── staging/                      # webapp-staging, 2 replicas, nginx:1.30-alpine
│       ├── prod/                         # webapp-prod,    3 replicas, + patch-resources.yaml (strategic merge)
│       ├── qa/                           # webapp-qa,      2 replicas, + patch-annotation.yaml (JSON 6902)
│       └── sandbox/                      # challenge: namePrefix "sandbox-" (render only)
├── evidence/                             # command output per task + rendered/ YAML
├── assets/                               # screenshots
└── report/practical-05-report.md
```

## Environment summary

| Overlay | Namespace | Replicas | Image | Extra |
| --- | --- | --- | --- | --- |
| dev | `webapp-dev` | 1 | `nginx:1.31-alpine` | — |
| staging | `webapp-staging` | 2 | `nginx:1.30-alpine` | — |
| prod | `webapp-prod` | 3 | `nginx:1.30-alpine` | resources patch, `change-policy` annotation |
| qa | `webapp-qa` | 2 | `nginx:1.30-alpine` | `training.example.com/owner: qa-team` (JSON 6902) |
| sandbox | `webapp-sandbox` | 1 | `nginx:1.30-alpine` | `namePrefix: sandbox-` |

## Versions

kind v0.32.0 (`~/go/bin/kind`), Kubernetes v1.36.1, kubectl v1.36.3 with Kustomize v5.8.1.

## Rebuild

```bash
~/go/bin/kind create cluster --config cluster/kind-cluster.yaml
kubectl get nodes

# safe workflow for each overlay: render -> diff -> apply -> verify
for env in dev staging prod qa; do
  kubectl kustomize examples/webapp/overlays/$env > /dev/null      # render (fails fast on errors)
  kubectl diff  -k examples/webapp/overlays/$env || true            # first run: "namespace not found" is expected
  kubectl apply -k examples/webapp/overlays/$env
  kubectl rollout status deployment/webapp -n webapp-$env
done

kubectl get deploy -A -l app.kubernetes.io/name=webapp
kubectl kustomize examples/webapp/overlays/sandbox                  # challenge: render only
```

Reach dev (port 8080 is taken on this laptop, so 8081 is used):

```bash
kubectl port-forward -n webapp-dev service/webapp 8081:80
curl http://127.0.0.1:8081
```

## Cleanup

```bash
for env in dev staging prod qa; do kubectl delete -k examples/webapp/overlays/$env; done
kubectl get ns | grep 'webapp-' || true
# ~/go/bin/kind delete cluster --name dso202      # when the cluster is no longer needed
```
