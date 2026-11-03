# DSO202 Practical 4 — GitOps with Argo CD

A pull-based GitOps pipeline: Argo CD, running inside the kind cluster, watches a folder of this public repository and keeps the `demo-app` namespace identical to it. Three experiments show Git as the single source of truth, self-healing of configuration drift, and automatic pruning.

## Layout

```text
dso202-practical-04/
├── README.md
├── argocd/argocd-app.yaml          # Argo CD Application (the only file applied with kubectl)
├── gitops-demo/manifests/          # DESIRED STATE watched by Argo CD
│   ├── deployment.yaml             # webapp-deploy (nginxdemos/hello:plain-text)
│   └── service.yaml                # webapp-svc (removed + restored in Experiment 3)
├── evidence/                       # captured command output
├── assets/                         # screenshots of the Argo CD UI
└── report/practical-04-report.md
```

> **Deviation from the guide:** instead of a separate `k8s-gitops-demo` repository, the desired state lives in `dso202-practical-04/gitops-demo/manifests` of the public coursework repo `tadashi-dev-eng/02230306-DSO202-Practicals`. The Application's `spec.source.path` points at that folder.

## Versions

| Component | Version |
| --- | --- |
| Cluster | kind `dso202-p2`, Kubernetes v1.36.1 (control-plane + 2 workers) |
| kubectl | v1.36.3 |
| Argo CD | stable manifest, `quay.io/argoproj/argocd` v3.5.x |
| App image | `nginxdemos/hello:plain-text` |

## Rebuild

```bash
# 1. cluster (reuse the course cluster, or create one)
kubectl get nodes

# 2. Argo CD — use --server-side: the ApplicationSet CRD is too large for client-side apply
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

# 3. UI
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
kubectl port-forward svc/argocd-server -n argocd 8080:443     # https://localhost:8080, user admin

# 4. register the app
kubectl apply -f argocd/argocd-app.yaml
kubectl get application my-webapp -n argocd
kubectl get all -n demo-app
```

## Cleanup

```bash
kubectl delete -f argocd/argocd-app.yaml      # app resources are not removed (no finalizer set)
kubectl delete namespace demo-app
kubectl delete namespace argocd               # optional: remove Argo CD
# kind delete cluster --name dso202-p2        # only when every practical using this cluster is finished
```
