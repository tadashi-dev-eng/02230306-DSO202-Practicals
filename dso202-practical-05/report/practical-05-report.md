# DSO202 Practical 5 — Environment-Specific Configuration with Kustomize on Kind

| | |
| --- | --- |
| Module | DSO202 — Scaling, Orchestration, Monitoring & Observability |
| Practical | 5 — Kustomize bases and overlays |
| Student | tadashi-dev-eng (02230306) |
| Date performed | 2026-11-03 |

---

## 1. Objective

The aim of this practical was to deploy one application to several environments (dev, staging, prod, and my own QA overlay) **without copying** its Deployment or Service. The parts shared by every environment live once in a Kustomize **base**. Each environment is an **overlay** that states only what differs: namespace, labels, replica count, image tag, generated content, and patches.

Along the way I:

- rendered a base and overlays and read the output before applying anything;
- followed the safe workflow `render → diff → apply → verify`;
- proved the chain from a content change to a new ConfigMap hash, a changed Deployment, and a rollout;
- compared a **strategic-merge patch** (prod resources) with a **JSON 6902 patch** (QA annotation);
- used `namePrefix` in a sandbox overlay to see how Kustomize rewrites references (challenge extension).

---

## 2. Environment

| Item | Value | Verify with |
| --- | --- | --- |
| Host | Ubuntu 25.04, Docker Desktop 29.2.1 | `lsb_release -ds; docker --version` |
| kind | v0.32.0 (`~/go/bin/kind`) | `~/go/bin/kind --version` |
| Cluster | `dso202` (context `kind-dso202`): `control-plane`, `worker-node-1`, `worker-node-2` | `kubectl get nodes -o wide` |
| Kubernetes | v1.36.1 | `kubectl get nodes` |
| kubectl / Kustomize | v1.36.3 / v5.8.1 (built into kubectl) | `kubectl version --client -o yaml` |
| Images | `nginx:1.30-alpine`, `nginx:1.31-alpine` (dev), `busybox:1.37` (test clients) | `kubectl get deploy -A -l app.kubernetes.io/name=webapp -o wide` |

**Notes and deviations**

- The guide assumes an `examples/webapp` repository. None was supplied, so I wrote the base and the dev/staging/prod overlays myself to match what the tasks describe (generator, image transform, replicas, prod resources patch).
- The Practical 2 cluster (`dso202-p2`) had been deleted, and with no current context kubectl silently fell back to `localhost:8080`, which on this laptop is a Jenkins container. I therefore created a new cluster from `cluster/kind-cluster.yaml`.
- `/usr/local/bin/kind` is v0.20.0 and comes first on `PATH`. I used `~/go/bin/kind` (v0.32.0), which matches the pinned node image.
- The port-forward uses **8081** because 8080 is taken by Jenkins.
- Rendered files go to `evidence/rendered/` instead of `/tmp`, so they are kept with the submission.

---

## 3. Procedure and Observations

Commands are run from the `dso202-practical-05/` folder. Raw output is in `evidence/`.

### 3.1 Task 0 — Pre-flight

Evidence: `evidence/task0-cluster-create.txt`, `evidence/task0-preflight.txt`

```bash
~/go/bin/kind create cluster --config cluster/kind-cluster.yaml
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl version --client -o yaml
kubectl apply --help | grep -- '--kustomize'
```

```text
Set kubectl context to "kind-dso202"
kind-dso202
Kubernetes control plane is running at https://127.0.0.1:45725

NAME            STATUS   ROLES           AGE   VERSION
control-plane   Ready    control-plane   58s   v1.36.1
worker-node-1   Ready    <none>          42s   v1.36.1
worker-node-2   Ready    <none>          43s   v1.36.1

  gitVersion: v1.36.3
kustomizeVersion: v5.8.1

    -k, --kustomize='':
```

The context reaches the kind cluster, all three nodes are `Ready`, and kubectl ships Kustomize v5.8.1, so both `kubectl kustomize` and `-k` are available.

### 3.2 Task 1 — Read the repository before running it

Evidence: `evidence/task1-repo-tree.txt`

```bash
tree examples/webapp
```

```text
examples/webapp
├── base
│   ├── deployment.yaml
│   ├── index.html
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    ├── dev
    │   ├── index.html
    │   ├── kustomization.yaml
    │   └── namespace.yaml
    ├── prod
    │   ├── index.html
    │   ├── kustomization.yaml
    │   ├── namespace.yaml
    │   └── patch-resources.yaml
    ├── qa                (added in Task 9)
    ├── sandbox           (added in the challenge)
    └── staging
        ├── index.html
        ├── kustomization.yaml
        └── namespace.yaml
```

1. **Files that exist once for all environments:** `base/deployment.yaml` and `base/service.yaml`, plus the base `kustomization.yaml` that declares the `web-content` ConfigMap generator.
2. **Values that differ between environments:** namespace, the `environment` label, replica count, nginx image tag, page content (`index.html`), container resources (prod only), and annotations (prod and QA).
3. **Where the differences live:** only in each overlay's `kustomization.yaml` fields (`namespace`, `labels`, `replicas`, `images`, `configMapGenerator` with `behavior: replace`, `commonAnnotations`, `patches`), in that overlay's own `index.html`, and in patch files. The whole dev overlay is about 20 lines of YAML.

### 3.3 Task 2 — Render the base (not applied)

Evidence: `evidence/task2-render-base.txt`, `evidence/rendered/base.yaml`

```bash
kubectl kustomize examples/webapp/base | grep '^kind:'
kubectl kustomize examples/webapp/base | grep 'name: web-content'
kubectl kustomize examples/webapp/base | grep -n 'namespace:' || echo '(no namespace in the base)'
```

```text
kind: ConfigMap
kind: Service
kind: Deployment

  name: web-content-8kt8822t88
          name: web-content-8kt8822t88

(no namespace in the base)
```

- **Deployment:** `webapp`, 1 replica, `nginx:1.30-alpine`.
- **Service:** `webapp`, ClusterIP on port 80.
- **Generated ConfigMap:** `web-content-8kt8822t88`, built from `base/index.html`.
- **Hash suffix:** `8kt8822t88`.
- **Rewritten reference:** the Deployment's `volumes[].configMap.name` was rewritten from `web-content` in the source to `web-content-8kt8822t88`.

**Checkpoint — why the name is not exactly `web-content`:** `configMapGenerator` appends a hash computed from the ConfigMap's content. Any change to `index.html` therefore produces a differently named ConfigMap, and Kustomize rewrites every reference to it. Because the Deployment's Pod template then changes, a content change cannot go unnoticed by running Pods (proved in Task 6). The base has no namespace, so it is environment-neutral and is never applied directly.

### 3.4 Task 3 — Compare dev and prod without touching the cluster

Evidence: `evidence/task3-dev-vs-prod-diff.txt`, `evidence/rendered/webapp-{dev,prod}.yaml`

```bash
kubectl kustomize examples/webapp/overlays/dev  > evidence/rendered/webapp-dev.yaml
kubectl kustomize examples/webapp/overlays/prod > evidence/rendered/webapp-prod.yaml
diff -u evidence/rendered/webapp-dev.yaml evidence/rendered/webapp-prod.yaml || true
```

Diff excerpt:

```diff
-  name: webapp-dev
+  name: webapp-prod
...
-        <h1>DEVELOPMENT environment</h1>
+        <h1>PRODUCTION environment</h1>
...
-  name: web-content-cmb6kg9m8t
-  namespace: webapp-dev
+  name: web-content-chb9cdd8k9
+  namespace: webapp-prod
...
+  annotations:
+    training.example.com/change-policy: cab-approved
-    environment: dev
+    environment: prod
...
-  replicas: 1
+  replicas: 3
...
-      - image: nginx:1.31-alpine
+      - image: nginx:1.30-alpine
...
           limits:
-            cpu: 200m
-            memory: 128Mi
+            cpu: 500m
+            memory: 256Mi
           requests:
-            cpu: 50m
+            cpu: 100m
             memory: 64Mi
```

| # | Difference | dev | prod | Produced by |
| --- | --- | --- | --- | --- |
| 1 | Namespace | `webapp-dev` | `webapp-prod` | `namespace:` + `namespace.yaml` |
| 2 | Environment label | `environment: dev` | `environment: prod` | `labels:` |
| 3 | Replicas | 1 | 3 | `replicas:` |
| 4 | Image tag | `nginx:1.31-alpine` | `nginx:1.30-alpine` | `images:` |
| 5 | Resources | 50m/64Mi → 200m/128Mi | 100m/64Mi → 500m/256Mi | `patch-resources.yaml` (prod only) |
| 6 | Page content | DEVELOPMENT | PRODUCTION | overlay `index.html` + `behavior: replace` |
| 7 | ConfigMap name (hash) | `web-content-cmb6kg9m8t` | `web-content-chb9cdd8k9` | follows from #6 |
| 8 | Annotations | none | `change-policy: cab-approved` on every object, including the Pod template | `commonAnnotations:` |

Each of the 8 differences can be traced to one or two lines of an overlay, not to a duplicated manifest.

### 3.5 Task 4 — Deploy dev safely

Evidence: `evidence/task4-deploy-dev.txt`

```bash
kubectl kustomize examples/webapp/overlays/dev | grep -E '^kind:|^  name:|namespace:|replicas:|image:'   # render
kubectl diff  -k examples/webapp/overlays/dev || true                                               # diff
kubectl apply -k examples/webapp/overlays/dev                                                       # apply
kubectl rollout status deployment/webapp -n webapp-dev                                              # verify
kubectl get all -n webapp-dev
kubectl get configmap -n webapp-dev
```

```text
$ kubectl diff -k examples/webapp/overlays/dev
Error from server (NotFound): namespaces "webapp-dev" not found

$ kubectl apply -k examples/webapp/overlays/dev
namespace/webapp-dev created
configmap/web-content-cmb6kg9m8t created
service/webapp created
deployment.apps/webapp created

deployment "webapp" successfully rolled out

NAME                          READY   STATUS    RESTARTS   AGE
pod/webapp-6c8f9d6d9b-zgvb7   1/1     Running   0          12s

NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/webapp   ClusterIP   10.96.224.154   <none>        80/TCP    12s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/webapp   1/1     1            1           12s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/webapp-6c8f9d6d9b   1         1         1       12s

NAME                     DATA   AGE
kube-root-ca.crt         1      12s
web-content-cmb6kg9m8t   1      12s
```

The Deployment keeps the base name `webapp`; the namespace `webapp-dev` is what separates the environment. `kubectl diff` failed on the first run because a server-side dry-run of namespaced objects needs the namespace to exist already, and this overlay is what creates it. After the apply, `kubectl diff -k ...; echo $?` printed nothing and exit code `0`, which means the cluster matches the render exactly.

### 3.6 Task 5 — Reach the application

Evidence: `evidence/task5-reach-dev.txt`

```bash
kubectl port-forward -n webapp-dev service/webapp 8081:80     # terminal 2
curl http://127.0.0.1:8081
```

```text
Forwarding from 127.0.0.1:8081 -> 80

<!doctype html>
<html>
  <head><title>webapp - DEVELOPMENT</title></head>
  <body>
    <h1>DEVELOPMENT environment</h1>
    <p>environment: dev</p>
  </body>
</html>
```

The response identifies the **development** environment. The port-forward was stopped afterwards.

> 📸 *Optional: screenshot of the curl output: `![dev page](../assets/dev-curl.png)`*

### 3.7 Task 6 — The ConfigMap hash/rollout chain

Evidence: `evidence/task6-configmap-hash-rollout.txt`

**Before:**

```bash
kubectl get configmap -n webapp-dev
kubectl get pods -n webapp-dev -o wide
kubectl get deploy webapp -n webapp-dev -o jsonpath='{.spec.template.spec.volumes[0].configMap.name}{"\n"}'
```

```text
web-content-cmb6kg9m8t   1      49s
webapp-6c8f9d6d9b-zgvb7   1/1     Running   0          50s   10.244.2.2   worker-node-1
web-content-cmb6kg9m8t
```

**Edit** `examples/webapp/overlays/dev/index.html`, changing the heading to `<h1>DEV v2 — configuration changed</h1>`.

**Render before applying:**

```bash
kubectl kustomize examples/webapp/overlays/dev | grep 'name: web-content'
```

```text
  name: web-content-g9688fbdg7
          name: web-content-g9688fbdg7
```

**Apply and inspect:**

```bash
kubectl apply -k examples/webapp/overlays/dev
kubectl rollout status deployment/webapp -n webapp-dev
kubectl get configmap -n webapp-dev
kubectl get pods -n webapp-dev -o wide
kubectl get rs -n webapp-dev
kubectl rollout history deployment/webapp -n webapp-dev
kubectl exec -n webapp-dev deploy/webapp -- cat /usr/share/nginx/html/index.html | grep '<h1>'
```

```text
configmap/web-content-g9688fbdg7 created
deployment.apps/webapp configured
deployment "webapp" successfully rolled out

NAME                     DATA   AGE
web-content-cmb6kg9m8t   1      61s
web-content-g9688fbdg7   1      11s

NAME                      READY   STATUS        RESTARTS   AGE   IP           NODE
webapp-69c7c5dd9b-vtl7w   1/1     Running       0          11s   10.244.1.2   worker-node-2
webapp-6c8f9d6d9b-zgvb7   1/1     Terminating   0          61s   10.244.2.2   worker-node-1

NAME                DESIRED   CURRENT   READY   AGE
webapp-69c7c5dd9b   1         1         1       11s
webapp-6c8f9d6d9b   0         0         0       61s

REVISION  CHANGE-CAUSE
1         <none>
2         <none>

    <h1>DEV v2 — configuration changed</h1>
```

| | Before | After |
| --- | --- | --- |
| Generated ConfigMap | `web-content-cmb6kg9m8t` | `web-content-g9688fbdg7` |
| Pod | `webapp-6c8f9d6d9b-zgvb7` | `webapp-69c7c5dd9b-vtl7w` |
| ReplicaSet (pod-template-hash) | `6c8f9d6d9b` | `69c7c5dd9b` |
| Rollout revision | 1 | 2 |

**The chain, explained:**

1. **File content changed.** I edited one line of `overlays/dev/index.html`.
2. **Generated ConfigMap content changed.** The generator copies the file into the `index.html` key of the ConfigMap's `data`.
3. **Generated ConfigMap name hash changed.** The suffix is a hash of that content, so `cmb6kg9m8t` became `g9688fbdg7`, and `kubectl apply` *created* a new ConfigMap instead of editing the old one.
4. **Deployment reference changed.** Kustomize rewrote `spec.template.spec.volumes[0].configMap.name` to the new name (visible in `kubectl diff` as `- web-content-cmb6kg9m8t / + web-content-g9688fbdg7`).
5. **Deployment Pod template changed.** That field is inside `spec.template`, so the template hash changed (`6c8f9d6d9b` → `69c7c5dd9b`).
6. **Rollout occurred.** The Deployment controller created a new ReplicaSet, started a new Pod that mounts the new ConfigMap, waited for it to be Ready, and then terminated the old Pod (revision 1 → 2).

If the ConfigMap had kept a fixed name, editing it would not have touched the Pod template, and the running nginx would have kept serving the old page until someone restarted it.

**Side effect:** the old `web-content-cmb6kg9m8t` is still in the namespace, because `kubectl apply` never deletes objects it is no longer given. That is useful for rolling back to revision 1, but old ConfigMaps pile up unless they are pruned (`kubectl apply --prune`, or Argo CD with `prune: true` as in Practical 4).

### 3.8 Task 7 — Deploy staging and prod

Evidence: `evidence/task7-staging-prod.txt`

```bash
kubectl diff  -k examples/webapp/overlays/staging || true
kubectl apply -k examples/webapp/overlays/staging
kubectl diff  -k examples/webapp/overlays/prod || true
kubectl apply -k examples/webapp/overlays/prod
kubectl get deploy -A -l app.kubernetes.io/name=webapp
kubectl get pods -A -l app.kubernetes.io/name=webapp -o wide
```

```text
NAMESPACE        NAME     READY   UP-TO-DATE   AVAILABLE   AGE
webapp-dev       webapp   1/1     1            1           89s
webapp-prod      webapp   3/3     3            3           17s
webapp-staging   webapp   2/2     2            2           17s

NAMESPACE        NAME                      READY   STATUS    NODE
webapp-dev       webapp-69c7c5dd9b-vtl7w   1/1     Running   worker-node-2
webapp-prod      webapp-57bbf77fc4-28bk5   1/1     Running   worker-node-1
webapp-prod      webapp-57bbf77fc4-9qt26   1/1     Running   worker-node-2
webapp-prod      webapp-57bbf77fc4-k2df2   1/1     Running   worker-node-1
webapp-staging   webapp-cbfc4bdcf-bj57t    1/1     Running   worker-node-1
webapp-staging   webapp-cbfc4bdcf-ml4pv    1/1     Running   worker-node-2
```

Verify the per-environment settings in one table, and that each Service serves its own page:

```bash
kubectl get deploy -A -l app.kubernetes.io/name=webapp \
  -o custom-columns=NAMESPACE:.metadata.namespace,ENV:.metadata.labels.environment,REPLICAS:.spec.replicas,IMAGE:.spec.template.spec.containers[0].image,CONFIGMAP:.spec.template.spec.volumes[0].configMap.name
kubectl run curl-prod -n webapp-prod --rm -i --restart=Never --image=busybox:1.37 -- wget -qO- http://webapp
```

```text
NAMESPACE        ENV       REPLICAS   IMAGE               CONFIGMAP
webapp-dev       dev       1          nginx:1.31-alpine   web-content-g9688fbdg7
webapp-prod      prod      3          nginx:1.30-alpine   web-content-chb9cdd8k9
webapp-staging   staging   2          nginx:1.30-alpine   web-content-5979g766k5

    <h1>PRODUCTION environment</h1>       (staging returned "STAGING environment")
```

**Replica counts by environment:** dev **1**, staging **2**, prod **3**. All three use the same Deployment and Service names in separate namespaces, and each has its own content hash.

### 3.9 Task 8 — What the prod patch changed

Evidence: `evidence/task8-prod-patch.txt`

```bash
cat examples/webapp/overlays/prod/patch-resources.yaml
kubectl kustomize examples/webapp/base | grep -A6 'resources:'
kubectl kustomize examples/webapp/overlays/prod | grep -A6 'resources:'
kubectl get deploy webapp -n webapp-prod -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'
kubectl get deploy webapp -n webapp-staging -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'
```

```yaml
# patch-resources.yaml (body)
spec:
  template:
    spec:
      containers:
        - name: nginx
          resources:
            requests: { cpu: 100m }
            limits:   { cpu: 500m, memory: 256Mi }
```

| Field | Base | Patch | Rendered prod |
| --- | --- | --- | --- |
| `requests.cpu` | 50m | 100m | **100m** |
| `requests.memory` | 64Mi | *(not in patch)* | **64Mi** ← kept from base |
| `limits.cpu` | 200m | 500m | **500m** |
| `limits.memory` | 128Mi | 256Mi | **256Mi** |

```text
prod:    {"limits":{"cpu":"500m","memory":"256Mi"},"requests":{"cpu":"100m","memory":"64Mi"}}
staging: {"limits":{"cpu":"200m","memory":"128Mi"},"requests":{"cpu":"50m","memory":"64Mi"}}
```

1. **Deleted or merged?** Merged. `requests.memory: 64Mi` is not in the patch, yet it survives in the rendered prod output and in the live object. The image, port, readiness probe and volume mount from the base container also survived. A strategic-merge patch matches the container by its merge key (`name: nginx`) and overrides only the fields it states.
2. **Who owns the production resource policy?** The prod overlay (`overlays/prod/patch-resources.yaml`). Staging still runs the base values, so the policy applies to prod alone.
3. **Why a patch rather than copying `deployment.yaml` into `prod/`?** The patch is 8 meaningful lines that show exactly what production changes. A copy would duplicate around 50 lines, and any later base fix (a probe, a security setting, a new volume) would silently miss prod unless someone remembered to edit both copies. Reviewers would also have to diff two full files to find the real difference.

### 3.10 Task 9 — My QA overlay

Evidence: `evidence/task9a-qa-first-attempt.txt`, `evidence/task9b-qa-overlay.txt`, `evidence/rendered/webapp-qa.yaml`

```text
overlays/qa/
├── index.html
├── kustomization.yaml
├── namespace.yaml
└── patch-annotation.yaml
```

`overlays/qa/kustomization.yaml` (comments removed):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
  - namespace.yaml
namespace: webapp-qa
labels:
  - pairs:
      environment: qa
    includeSelectors: false
    includeTemplates: true
replicas:
  - name: webapp
    count: 2
configMapGenerator:
  - name: web-content
    behavior: replace
    files:
      - index.html
patches:
  - path: patch-annotation.yaml
    target:
      group: apps
      version: v1
      kind: Deployment
      name: webapp
```

`overlays/qa/patch-annotation.yaml` (JSON 6902):

```yaml
- op: add
  path: /metadata/annotations/training.example.com~1owner
  value: qa-team
```

`overlays/qa/namespace.yaml` creates `webapp-qa`, and `overlays/qa/index.html` contains `<h1>QA environment</h1>`.

**Render first; do not proceed until the annotation appears:**

```bash
kubectl kustomize examples/webapp/overlays/qa | grep -B2 -A1 'training.example.com/owner'
```

```text
metadata:
  annotations:
    training.example.com/owner: qa-team
  labels:
```

**Diff, apply and verify:**

```bash
kubectl diff  -k examples/webapp/overlays/qa || true
kubectl apply -k examples/webapp/overlays/qa
kubectl rollout status deployment/webapp -n webapp-qa
kubectl get deployment webapp -n webapp-qa -o yaml | sed -n '1,25p'
kubectl get deployment webapp -n webapp-qa -o jsonpath='{.metadata.annotations.training\.example\.com/owner}{"\n"}'
kubectl run curl-qa -n webapp-qa --rm -i --restart=Never --image=busybox:1.37 -- wget -qO- http://webapp
```

```text
namespace/webapp-qa created
configmap/web-content-5kg4ghcdcg created
service/webapp created
deployment.apps/webapp created
deployment "webapp" successfully rolled out

metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    ...
    training.example.com/owner: qa-team
  labels:
    ...
    environment: qa
  name: webapp
  namespace: webapp-qa
spec:
  replicas: 2

qa-team

    <h1>QA environment</h1>
    <p>environment: qa</p>
```

| Requirement | Evidence |
| --- | --- |
| Namespace `webapp-qa` | `namespace: webapp-qa` in the live object |
| Replicas 2 | `spec.replicas: 2`, `2/2` ready |
| Label `environment: qa` | in labels |
| Unique `index.html` | curl returns "QA environment" |
| Same base | `resources: [../../base]`; no deployment.yaml or service.yaml in `qa/` |
| Owner annotation via JSON 6902 | `training.example.com/owner: qa-team` |

**Testing the JSON Pointer escaping:** to understand the `~1`, I rendered a test copy of the patch with the key left unescaped (`/metadata/annotations/training.example.com/owner`):

```text
error: add operation does not apply: doc is missing path: "/metadata/annotations/training.example.com/owner": missing value
```

In a JSON Pointer, `/` separates path segments, so the unescaped key is read as an `owner` field inside a non-existent `training.example.com` map. The `/` inside the key has to be written as `~1`. I also noticed that the base Deployment has no `metadata.annotations` at all, yet the escaped `add` still worked: Kustomize's patch engine created the parent map, which a strict RFC 6902 implementation would refuse.

### 3.11 Strategic merge versus JSON 6902

| | Strategic-merge patch (prod `patch-resources.yaml`) | JSON 6902 patch (qa `patch-annotation.yaml`) |
| --- | --- | --- |
| Format | A partial Kubernetes object (`apiVersion`, `kind`, `metadata.name` + the fields to change) | A list of operations: `op` (`add`, `replace`, `remove`, `move`, `copy`, `test`), `path`, `value` |
| How it finds its target | From the patch body (kind and name) | Needs an explicit `target:` in `kustomization.yaml` |
| How it addresses lists | By merge key, e.g. the container `name: nginx`, so order does not matter | By JSON Pointer index or path, e.g. `/spec/template/spec/containers/0/...`, which depends on order |
| Semantics | Merges: unspecified fields are kept (`requests.memory` survived) | Exactly the operations listed, nothing else |
| Strengths | Readable; looks like the resource; good for "change these values" | Precise; can **remove** a field or list item, or act on keys a merge cannot express; works on CRDs that lack merge metadata |
| Pitfalls | Cannot easily delete a field; behaves differently on CRDs without schema merge keys | Path escaping (`~1` for `/`, `~0` for `~`); brittle list indexes; a missing path fails the render |

I chose strategic merge for prod resources because it reads like the Deployment and keeps base values I did not mention. JSON 6902 suits the QA annotation: a single targeted `add` to one exact path.

### 3.12 Challenge extension — `namePrefix` in a sandbox overlay

Evidence: `evidence/challenge-prediction.md`, `evidence/challenge-nameprefix.txt`, `evidence/rendered/webapp-sandbox.yaml`

`overlays/sandbox/kustomization.yaml` adds `namePrefix: sandbox-` with namespace `webapp-sandbox`. I wrote this prediction **before** rendering:

| Field | Predicted | Rendered | Match |
| --- | --- | --- | --- |
| Deployment name | `sandbox-webapp` | `sandbox-webapp` | ✅ |
| Service name | `sandbox-webapp` | `sandbox-webapp` | ✅ |
| ConfigMap name | `sandbox-web-content-<same hash>` | `sandbox-web-content-8kt8822t88` (base: `web-content-8kt8822t88`) | ✅ |
| Deployment `volumes[].configMap.name` | rewritten to the prefixed name | `sandbox-web-content-8kt8822t88` | ✅ |
| Namespace object | **not** prefixed (unsure) | `webapp-sandbox` | ✅ |
| `metadata.namespace` | `webapp-sandbox` | `webapp-sandbox` | ✅ |
| Service selector / `matchLabels` | unchanged | `app.kubernetes.io/name: webapp` | ✅ |
| Container `nginx`, volume `content`, port `http` | unchanged | unchanged | ✅ |

```bash
kubectl kustomize examples/webapp/overlays/sandbox | grep -E '^kind:|^  name:|^  namespace:'
kubectl kustomize examples/webapp/overlays/sandbox | grep -A1 'configMap:'
kubectl kustomize examples/webapp/overlays/sandbox | grep -A2 -E 'selector:|matchLabels:'
```

```text
kind: Namespace
  name: webapp-sandbox
kind: ConfigMap
  name: sandbox-web-content-8kt8822t88
  namespace: webapp-sandbox
kind: Service
  name: sandbox-webapp
  namespace: webapp-sandbox
kind: Deployment
  name: sandbox-webapp
  namespace: webapp-sandbox

      - configMap:
          name: sandbox-web-content-8kt8822t88

  selector:
    app.kubernetes.io/name: webapp
  ...
    matchLabels:
      app.kubernetes.io/name: webapp
```

**Reference-aware transformation:** Kustomize renamed the ConfigMap *and* rewrote the field in the Deployment that refers to it. Without that, the Pod would mount a ConfigMap that does not exist and stay stuck in `ContainerCreating`. It left labels and selectors alone, because they are not object names, so the Service still selects the Pods. The hash stayed `8kt8822t88` because the content is identical to the base: the suffix depends on content, not on the prefix.

### 3.13 Task 10 — Cleanup

> **TODO:** run this after taking any screenshots, and paste the output here.

```bash
kubectl delete -k examples/webapp/overlays/dev
kubectl delete -k examples/webapp/overlays/staging
kubectl delete -k examples/webapp/overlays/prod
kubectl delete -k examples/webapp/overlays/qa
kubectl get ns | grep 'webapp-' || true
kubectl get configmap -A | grep web-content || echo "no web-content ConfigMaps left"
```

Deleting the overlay also deletes its namespace, which takes the orphaned `web-content-cmb6kg9m8t` with it, even though the current render no longer mentions that ConfigMap.

---

## 4. Analysis

**Q1. What is the value of an overlay compared with a copied manifest?**
A reviewer can see an environment's whole difference in the overlay's `kustomization.yaml`: the dev overlay is around 20 lines, compared with roughly 80 lines of rendered output. A fix to the base (for example the readiness probe) reaches every environment on the next apply, whereas with copies it would have to be repeated four times and could drift.

**Q2. Why does `configMapGenerator` add a hash, and what is the trade-off?**
To turn a configuration change into a Pod-template change, and therefore a rollout (Task 6). It also makes rollbacks safe, because the old revision still points to the old, unchanged ConfigMap. The cost is that old ConfigMaps build up (`cmb6kg9m8t` was left behind) unless they are pruned or the namespace is deleted.

**Q3. Why `includeSelectors: false` for the `environment` label?**
`spec.selector` on a Deployment is immutable. If a label in the selector ever changed, every apply would fail until the Deployment was deleted. Keeping only the stable `app.kubernetes.io/name` in selectors, and putting `environment` on metadata and Pod templates only, avoids that trap while still letting me filter by environment (`--show-labels` shows `environment=dev` on the Pod).

**Q4. Why render and diff before applying?**
The render catches Kustomize errors with nothing sent to the cluster. The unescaped JSON-pointer test failed at render time, and I confirmed the QA annotation before applying. The diff shows exactly what the API server will change; in Task 6 it showed the ConfigMap reference swap before the rollout happened. One limitation I saw: `kubectl diff` cannot dry-run into a namespace that does not exist yet, so for a brand-new environment the render is the main check.

**Q5. Strategic merge versus JSON 6902?**
See §3.11. In short, strategic merge describes the *desired fragment* and keeps everything else (prod `requests.memory` survived). JSON 6902 describes *operations at exact paths*, needs a `target:` and correct escaping, and can do things merge cannot, such as removing a field.

---

## 5. Reflection

> **Rewrite in your own words.** These prompts come from what actually happened in this run.

**A Kustomize mistake and how rendered output exposed it.** The JSON 6902 path. Annotation keys contain `/`, which is also the JSON Pointer separator. Rendering the patch with the key unescaped failed at once with `doc is missing path: "/metadata/annotations/training.example.com/owner"`. Reading that path showed that Kustomize had split the key into two segments, and escaping it as `training.example.com~1owner` fixed it. Because I rendered before applying, the mistake never reached the cluster. *(If you hit a different mistake yourself, describe that one instead.)*

**Something that surprised me.** `kubectl diff -k` failed for every new environment (`namespaces "webapp-dev" not found`), even though the overlay creates that namespace. The server-side dry-run validates namespaced objects against namespaces that already exist. For a first deploy, the rendered YAML is the review; `kubectl diff` becomes useful from the second apply onwards.

**Environment problem.** kubectl suddenly returned a Jenkins `Authentication required` page. The old kind cluster had been deleted, no context was set, and kubectl fell back to `localhost:8080`, where Jenkins runs. `kubectl config current-context` → `current-context is not set` revealed it.

**What I would do differently.** Add a `test` operation to the JSON patch, or a CI step that renders every overlay and checks for the expected fields, so that a patch that silently matches nothing fails the build.

**Still unclear.** The best way to prune the old hashed ConfigMaps in a plain `kubectl apply -k` workflow without risking deletion of objects that belong to other tools, since `--prune` with label selectors seems easy to get wrong.

---

## 6. References

All accessed 2026-11-03.

1. DSO202 Practical 5 guide — https://hackmd.io/@sarojsanyasi/BJ6jbmdFze
2. Kubernetes — Declarative Management of Kubernetes Objects Using Kustomize: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/
3. Kustomize reference — `configMapGenerator`, `labels`, `replicas`, `images`, `namePrefix`, `patches`: https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/
4. RFC 6902 — JavaScript Object Notation (JSON) Patch: https://datatracker.ietf.org/doc/html/rfc6902
5. RFC 6901 — JSON Pointer (`~0` / `~1` escaping): https://datatracker.ietf.org/doc/html/rfc6901
6. Kubernetes — Update API Objects in Place Using kubectl patch (strategic merge): https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/
7. kind — Configuration: https://kind.sigs.k8s.io/docs/user/configuration/
