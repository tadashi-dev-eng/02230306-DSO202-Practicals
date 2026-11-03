# DSO202 Practical 4 — GitOps and Argo CD

| | |
| --- | --- |
| Module | DSO202 — Scaling, Orchestration, Monitoring & Observability |
| Practical | 4 — GitOps with Argo CD |
| Student | tadashi-dev-eng (02230306) |
| Date performed | 2026-11-03 |
| Desired-state repo | https://github.com/tadashi-dev-eng/02230306-DSO202-Practicals (folder `dso202-practical-04/gitops-demo/manifests`) |

---

## 1. Objective

The aim of this practical was to replace push-based deployment (a person or CI job running `kubectl apply` against the cluster) with **pull-based GitOps**. Argo CD runs inside the cluster, continuously compares the live state with the manifests stored in a Git repository, and makes the cluster match Git.

I installed Argo CD on the course kind cluster, registered an application **declaratively** with an `Application` custom resource, and ran three experiments:

1. **Git as the single source of truth** — scaling the app by committing a change to Git, not by running kubectl.
2. **Configuration drift and self-healing** — changing the cluster by hand and watching Argo CD revert it (`selfHeal: true`).
3. **Automatic pruning** — deleting a manifest from Git and watching Argo CD delete the matching object (`prune: true`), then restoring it with `git revert`.

---

## 2. Environment

| Item | Value | Verify with |
| --- | --- | --- |
| Host OS | Ubuntu 25.04, Linux 6.14.0-37-generic, Docker Desktop 29.2.1 | `uname -sr; docker --version` |
| Cluster | kind cluster `dso202-p2` (context `kind-dso202-p2`): `control-plane`, `worker-node-1`, `worker-node-2` | `kubectl config current-context; kubectl get nodes` |
| Kubernetes | v1.36.1 | `kubectl get nodes` |
| kubectl | v1.36.3 | `kubectl version --client` |
| Argo CD | v3.5.7 (`stable` install manifest, image `quay.io/argoproj/argocd:v3.5.7`) | `curl -sk https://localhost:8443/api/version` |
| App image | `nginxdemos/hello:plain-text` | `kubectl get deploy webapp-deploy -n demo-app -o jsonpath='{..image}'` |

**Deviations from the guide**

| Guide | What I did | Why |
| --- | --- | --- |
| Separate repo `k8s-gitops-demo` | Desired state in `dso202-practical-04/gitops-demo/manifests` inside my public coursework repo; `spec.source.path` points there | Keeps the whole practical in one submission repository |
| Cluster named `dso202` | Reused the existing course cluster `dso202-p2` (same 1 CP + 2 worker layout) | The guide says "same K8s cluster" |
| `port-forward ... 8080:443` | `port-forward ... 8443:443`, UI at `https://localhost:8443` | Port 8080 was already in use on my laptop (`ss -ltn` shows `*:8080 LISTEN`) |
| `kubectl apply -n argocd -f install.yaml` | Added `--server-side --force-conflicts` | The ApplicationSet CRD was missing after the plain apply (see §3.2 and Reflection) |
| Click **Refresh** in the UI | Also ran `kubectl annotate application my-webapp -n argocd argocd.argoproj.io/refresh=normal --overwrite` | Same action as the button, scriptable and recordable |

---

## 3. Procedure and Observations

All raw output is in `evidence/`. Commands are run from the `dso202-practical-04/` folder.

### 3.1 Module 4 Step 1 — The 3-node cluster

```bash
kubectl config current-context
kubectl get nodes
```

```text
kind-dso202-p2

NAME            STATUS   ROLES           AGE   VERSION
control-plane   Ready    control-plane   58d   v1.36.1
worker-node-1   Ready    <none>          58d   v1.36.1
worker-node-2   Ready    <none>          58d   v1.36.1
```

One control-plane and two worker nodes, all `Ready`.

### 3.2 Module 4 Step 2 — Install Argo CD

Evidence: `evidence/module4-setup.txt`

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
kubectl get pods -n argocd
```

An earlier install attempt had left Argo CD in a broken state: one controller was crash-looping.

```text
argocd-applicationset-controller-58bb5c7479-8q4gq   0/1     Error     221 (7m47s ago)   7d7h
```

**Diagnosis:**

```bash
kubectl logs -n argocd deploy/argocd-applicationset-controller --tail=50 | grep -o 'no matches for kind [^,]*'
kubectl get crd | grep argoproj
```

```text
no matches for kind \"ApplicationSet\" in version \"argoproj.io/v1alpha1\"
applications.argoproj.io        2026-10-27T08:17:40Z
appprojects.argoproj.io         2026-10-27T08:17:40Z
```

The `applicationsets.argoproj.io` CRD was missing. Its schema is so large that a client-side `kubectl apply`, which stores the whole object in the `last-applied-configuration` annotation, exceeds the 256 KiB annotation limit and is rejected. Every other object was created, which is why the failure was easy to miss.

**Fix — server-side apply:**

```bash
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get crd | grep argoproj
kubectl rollout restart deployment/argocd-applicationset-controller -n argocd
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
kubectl get pods -n argocd
```

```text
applications.argoproj.io        2026-10-27T08:17:40Z
applicationsets.argoproj.io     2026-11-03T15:49:23Z
appprojects.argoproj.io         2026-10-27T08:17:40Z

NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          ...
argocd-applicationset-controller-5cc5ff4f47-h7lw6   1/1     Running   0          ...
argocd-dex-server-...                               1/1     Running   0          ...
argocd-notifications-controller-788bdb44d5-p57qk    1/1     Running   0          ...
argocd-redis-...                                    1/1     Running   0          ...
argocd-repo-server-656b96f67d-xf2cw                 1/1     Running   0          ...
argocd-server-764dc8f847-56tjx                      1/1     Running   0          ...
```

All three Argo CD CRDs now exist and all 7 components are `1/1 Running`. Because `stable` had moved on, the re-apply also upgraded Argo CD from v3.5.6 to v3.5.7. The new dex and redis images took several minutes to pull, so the first `kubectl wait` timed out and had to be re-run.

### 3.3 Module 4 Step 3 — Admin credentials and the UI

Evidence: `evidence/module4-ui-access.txt`

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
kubectl port-forward svc/argocd-server -n argocd 8443:443     # keep running in a second terminal
```

```text
<16-character password — redacted, this repository is public>
Forwarding from 127.0.0.1:8443 -> 8080
Forwarding from [::1]:8443 -> 8080
```

Verify the UI and API are reachable and the admin login works:

```bash
curl -sk -o /dev/null -w 'HTTP %{http_code}\n' https://localhost:8443/
curl -sk https://localhost:8443/healthz; echo
curl -sk https://localhost:8443/api/version; echo
curl -sk https://localhost:8443/api/v1/session -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"<password>"}'
```

```text
HTTP 200
ok
{"Version":"v3.5.7"}
{"token":"eyJhbGciOi... (session token issued - login OK)"}
```

The UI is served at `https://localhost:8443` with a self-signed certificate, so the browser warning is expected. The `admin` login succeeds.

> 📸 *Add a screenshot of the Argo CD login page / application tile here: `![Argo CD UI](../assets/argocd-ui.png)`*

### 3.4 Module 5 Step 1 — The desired state in GitHub

Evidence: `evidence/module5-step2-register-app.txt`

`gitops-demo/manifests/deployment.yaml` defines `webapp-deploy` (2 replicas, `nginxdemos/hello:plain-text`), and `gitops-demo/manifests/service.yaml` defines the ClusterIP Service `webapp-svc` on port 80. Both were committed and pushed.

```bash
git add gitops-demo argocd
git commit -m "practical-04: initial GitOps desired state (webapp, 2 replicas) and Argo CD Application"
git push origin main
git log --oneline -1 -- gitops-demo/manifests
git ls-remote https://github.com/tadashi-dev-eng/02230306-DSO202-Practicals.git refs/heads/main
```

```text
7abd00f practical-04: initial GitOps desired state (webapp, 2 replicas) and Argo CD Application
7abd00f47daec41c615e1e2c01730f4cf1aa6c91	refs/heads/main
```

The local commit and the GitHub `main` branch both point to `7abd00f`. This commit is the desired state.

### 3.5 Module 5 Step 2 — Register the Application declaratively

`argocd/argocd-app.yaml` has `repoURL` set to the coursework repo, `path: dso202-practical-04/gitops-demo/manifests`, destination namespace `demo-app`, `automated: {prune: true, selfHeal: true}` and `CreateNamespace=true`.

```bash
kubectl get namespace demo-app
kubectl apply -f argocd/argocd-app.yaml
kubectl get applications -n argocd
kubectl get namespace demo-app
kubectl get all -n demo-app
kubectl get pods -n demo-app -o wide
```

```text
Error from server (NotFound): namespaces "demo-app" not found
application.argoproj.io/my-webapp created

NAME        SYNC STATUS   HEALTH STATUS
my-webapp   Synced        Healthy

NAME       STATUS   AGE
demo-app   Active   67s

service/webapp-svc   ClusterIP   10.96.120.181   <none>   80/TCP   67s
deployment.apps/webapp-deploy   2/2 ...
replicaset.apps/webapp-deploy-6bd9c658d4   2   2   2

NAME                             READY   STATUS    RESTARTS   AGE     IP            NODE
webapp-deploy-6bd9c658d4-bhhdg   1/1     Running   0          2m11s   10.244.2.36   worker-node-1
webapp-deploy-6bd9c658d4-rsmgv   1/1     Running   0          2m11s   10.244.1.20   worker-node-2
```

Before the apply, `demo-app` did not exist. Argo CD created the namespace (`CreateNamespace=true`), the Deployment and the Service from Git, and the two Pods were spread across both workers. The only object I applied by hand was the `Application`. For about the first minute health showed `Progressing`, until the image finished pulling on `worker-node-1`.

Verify which resources Argo CD manages and that the app actually serves traffic:

```bash
kubectl get application my-webapp -n argocd \
  -o jsonpath='{range .status.resources[*]}{.kind}{"\t"}{.name}{"\t"}{.status}{"\n"}{end}'
kubectl run curl-test -n demo-app --rm -i --restart=Never --image=busybox:1.37 -- wget -qO- http://webapp-svc
```

```text
Service	webapp-svc	Synced
Deployment	webapp-deploy	Synced

Server address: 10.244.1.20:80
Server name: webapp-deploy-6bd9c658d4-rsmgv
```

> 📸 *Add a screenshot of the `my-webapp` resource tree (Application → Deployment → ReplicaSet → Pods, Service): `![my-webapp tree](../assets/argocd-tree.png)`*

### 3.6 Module 5 Step 3 — Experiment 1: Git as the single source of truth

Evidence: `evidence/module5-exp1-git-scale.txt`

I changed `replicas: 2` to `replicas: 4` in Git. No kubectl command touched the Deployment.

```bash
kubectl get deployment webapp-deploy -n demo-app          # before
sed -i 's/^  replicas: 2$/  replicas: 4/' gitops-demo/manifests/deployment.yaml
git diff -- gitops-demo/manifests/deployment.yaml
git commit -m "practical-04: scale webapp to 4 replicas (Experiment 1)" -- gitops-demo/manifests/deployment.yaml
git push origin main
kubectl annotate application my-webapp -n argocd argocd.argoproj.io/refresh=normal --overwrite   # = UI "Refresh"
```

```text
webapp-deploy   2/2     2            2           2m21s

-  replicas: 2
+  replicas: 4

a091227 practical-04: scale webapp to 4 replicas (Experiment 1)
   7abd00f..a091227  main -> main
application.argoproj.io/my-webapp annotated
```

Verify:

```bash
kubectl get application my-webapp -n argocd \
  -o custom-columns=NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status,REVISION:.status.sync.revision
kubectl get deployment webapp-deploy -n demo-app
kubectl get pods -n demo-app -o wide
kubectl get application my-webapp -n argocd \
  -o jsonpath='{range .status.history[*]}{.id}{"  "}{.revision}{"  "}{.deployedAt}{"\n"}{end}'
```

```text
NAME        SYNC     HEALTH    REVISION
my-webapp   Synced   Healthy   a09122732e6035ce547807475b409409849ff74e

webapp-deploy   4/4     4            4           2m27s

webapp-deploy-6bd9c658d4-5t6h4   1/1   Running   0   1s      10.244.1.22   worker-node-2
webapp-deploy-6bd9c658d4-bhhdg   1/1   Running   0   2m27s   10.244.2.36   worker-node-1
webapp-deploy-6bd9c658d4-rsmgv   1/1   Running   0   2m27s   10.244.1.20   worker-node-2
webapp-deploy-6bd9c658d4-swl7r   1/1   Running   0   1s      10.244.2.37   worker-node-1

0  7abd00f47daec41c615e1e2c01730f4cf1aa6c91  2026-11-03T15:57:24Z
1  a09122732e6035ce547807475b409409849ff74e  2026-11-03T15:59:50Z
```

Argo CD picked up commit `a091227` and created two new Pods (age 1s). The two original Pods kept running. Argo CD's deployment history now maps each change in the cluster to a Git commit hash, which is the audit trail GitOps promises.

### 3.7 Module 5 Step 4 — Experiment 2: Configuration drift and self-healing

Evidence: `evidence/module5-exp2-self-heal.txt`

To act as a "rogue admin", I bypassed Git and scaled the Deployment to 1 with kubectl, then sampled the state once per second.

```bash
kubectl scale deployment webapp-deploy -n demo-app --replicas=1
for i in $(seq 1 15); do
  echo "$(date +%T)  spec=$(kubectl get deploy webapp-deploy -n demo-app -o jsonpath={.spec.replicas})" \
       " ready=$(kubectl get deploy webapp-deploy -n demo-app -o jsonpath={.status.readyReplicas})" \
       " app=$(kubectl get application my-webapp -n argocd -o jsonpath={.status.sync.status}/{.status.health.status})"
  sleep 1
done
```

```text
deployment.apps/webapp-deploy scaled
22:00:07  spec=1  ready=1  app=OutOfSync/Healthy
22:00:08  spec=4  ready=4  app=Synced/Healthy
22:00:09  spec=4  ready=4  app=Synced/Healthy
...
22:00:23  spec=4  ready=4  app=Synced/Healthy
```

Within about 1 second the application went `OutOfSync` and Argo CD put `spec.replicas` back to 4, the value in Git.

Verify that Argo CD itself made the correction:

```bash
kubectl get pods -n demo-app -o wide
kubectl get events -n argocd --field-selector involvedObject.name=my-webapp --sort-by=.lastTimestamp | tail -n 6
kubectl get application my-webapp -n argocd \
  -o jsonpath='{.status.operationState.operation.initiatedBy}{"\n"}{.status.operationState.message}{"\n"}'
```

```text
webapp-deploy-6bd9c658d4-7s7b2   1/1   Running   0   17s   10.244.2.38   worker-node-1
webapp-deploy-6bd9c658d4-mx5xv   1/1   Running   0   17s   10.244.1.23   worker-node-2
webapp-deploy-6bd9c658d4-rsmgv   1/1   Running   0   3m    10.244.1.20   worker-node-2
webapp-deploy-6bd9c658d4-t7n4d   1/1   Running   0   17s   10.244.2.39   worker-node-1

Normal   OperationCompleted   application/my-webapp   Partial sync operation to a09122732e... succeeded
Normal   ResourceUpdated      application/my-webapp   Updated sync status: OutOfSync -> Synced

{"automated":true}
successfully synced (all tasks run)
```

The sync was `initiatedBy: {"automated":true}`, not by a person. Self-heal restored the replica count but not the same Pods: three of the four are new (age 17s) and only `rsmgv` survived. The manual scale-down had already deleted the other three, so their in-memory state was lost anyway. Self-heal corrects declared configuration; it does not undo side effects.

> 📸 *Optional: screenshot of the UI showing `OutOfSync` → `Synced`.*

### 3.8 Module 5 Step 5 — Experiment 3: Automatic pruning

Evidence: `evidence/module5-exp3-prune.txt`

```bash
kubectl get service -n demo-app                                   # before
git rm gitops-demo/manifests/service.yaml
git commit -m "practical-04: remove webapp-svc from Git (Experiment 3 - prune)" -- gitops-demo/manifests/service.yaml
git push origin main
kubectl annotate application my-webapp -n argocd argocd.argoproj.io/refresh=normal --overwrite
```

```text
webapp-svc   ClusterIP   10.96.120.181   <none>   80/TCP   3m14s
09c185a practical-04: remove webapp-svc from Git (Experiment 3 - prune)
   a091227..09c185a  main -> main
```

Verify:

```bash
kubectl get application my-webapp -n argocd -o custom-columns=NAME:.metadata.name,SYNC:.status.sync.status,REVISION:.status.sync.revision
kubectl get service -n demo-app
kubectl get application my-webapp -n argocd \
  -o jsonpath='{range .status.operationState.syncResult.resources[*]}{.kind}{"\t"}{.name}{"\t"}{.status}{"\t"}{.message}{"\n"}{end}'
```

```text
my-webapp   Synced   09c185a867462c1bf780a8f081ebfef0faac728b
No resources found in demo-app namespace.
Service	webapp-svc	Pruned	pruned
Deployment	webapp-deploy	Synced	deployment.apps/webapp-deploy unchanged
```

Argo CD recorded `webapp-svc` as **Pruned** and deleted it from the cluster. The Deployment, which is still in Git, was left `unchanged`.

**Restore with `git revert`:**

```bash
git revert --no-edit HEAD
git push origin main
kubectl annotate application my-webapp -n argocd argocd.argoproj.io/refresh=normal --overwrite
kubectl get service -n demo-app
kubectl run curl-test -n demo-app --rm -i --restart=Never --image=busybox:1.37 -- wget -qO- http://webapp-svc
kubectl get application my-webapp -n argocd \
  -o jsonpath='{range .status.history[*]}{.id}{"  "}{.revision}{"  "}{.deployedAt}{"\n"}{end}'
```

```text
cffd17d Revert "practical-04: remove webapp-svc from Git (Experiment 3 - prune)"
   09c185a..cffd17d  main -> main

webapp-svc   ClusterIP   10.96.27.99   <none>   80/TCP   3s

Server address: 10.244.2.38:80
Server name: webapp-deploy-6bd9c658d4-7s7b2

0  7abd00f47daec41c615e1e2c01730f4cf1aa6c91  2026-11-03T15:57:24Z
1  a09122732e6035ce547807475b409409849ff74e  2026-11-03T15:59:50Z
2  09c185a867462c1bf780a8f081ebfef0faac728b  2026-11-03T16:00:43Z
3  cffd17d3e0fb901119d66216a2131f19b4c68e77  2026-11-03T16:00:53Z
```

The Service came back and serves traffic again, so a rollback in GitOps is just a Git revert. Note that its ClusterIP changed from `10.96.120.181` to `10.96.27.99`, because a pruned object is truly deleted and the restored one is a new object. Clients should use the DNS name `webapp-svc`, not the IP. The four history entries match the four commits in `git log`.

### 3.9 Cleanup (Module 6.3)

The guide ends with `kind delete cluster --name dso202`. My course cluster `dso202-p2` still hosts other practicals' workloads, so I did not delete the cluster. Remove only this practical's objects with:

```bash
kubectl delete -f argocd/argocd-app.yaml
kubectl delete namespace demo-app
kubectl get applications -n argocd
kubectl get ns demo-app
# when all practicals are finished:  kind delete cluster --name dso202-p2
```

> **TODO:** run this when finished and paste the output here.

---

## 4. Analysis

**Q1. Push-based CI/CD versus pull-based GitOps — what changed in this practical?**
In push-based delivery, an external system holds cluster credentials and runs `kubectl apply`. Here nothing outside the cluster touched it after the single `Application` was applied. Argo CD, running inside the cluster, pulled the desired state from GitHub (a public repo, so no credentials were needed) and applied it. Cluster credentials never leave the cluster, and the cluster converges on its own (Experiment 1: commit `a091227` produced 4/4 Pods with no kubectl involved).

**Q2. Map the four GitOps principles to evidence.**
- *Declarative:* the app (`deployment.yaml`, `service.yaml`) and even its registration with Argo CD (`argocd-app.yaml`) are YAML, not commands.
- *Versioned and immutable:* every state is a commit; Argo CD's `status.history` lists `7abd00f → a091227 → 09c185a → cffd17d`, identical to `git log`.
- *Pulled automatically:* `syncPolicy.automated` applied each new revision after a refresh, with no manual Sync.
- *Continuously reconciled:* Experiment 2 — a manual change was reverted in about 1 s (`initiatedBy: {"automated":true}`).

**Q3. Why is the Argo CD `Application` a CRD, and which controller acts on it?**
`applications.argoproj.io` extends the Kubernetes API with a new kind, so the app's registration can be stored, versioned and applied like any other manifest. The `argocd-application-controller` (a StatefulSet) watches `Application` objects, asks the `argocd-repo-server` to render the manifests at the target revision, compares them with the live objects, and syncs. The ApplicationSet controller crash-looped precisely because its CRD was missing: a controller cannot watch a kind the API server does not know (`no matches for kind "ApplicationSet"`).

**Q4. What does `selfHeal` do, and what are its limits?**
It makes the application controller re-sync automatically when live state drifts from Git, not only when Git changes. It restored `spec.replicas` from 1 to 4, but it re-creates objects rather than resurrecting them: three of the four Pods were new. It also only covers fields declared in Git. A field Git does not specify (for example an extra label added with kubectl) is not treated as drift.

**Q5. Why is `prune` off by default?**
Deleting live resources because a file vanished from Git is dangerous: a bad merge, a wrong `path:` or a renamed folder could wipe production objects, including PVC-backed data. Off by default, Argo CD only marks such objects as orphaned (`OutOfSync`) and a human decides. With `prune: true` the Service was deleted (`Pruned`), and restoring it produced a new object with a new ClusterIP.

**Q6. How did Argo CD learn about new commits, and how fast?**
By polling Git, every ~3 minutes by default (`timeout.reconciliation`). I forced an immediate refresh with the `argocd.argoproj.io/refresh=normal` annotation, which is what the UI Refresh button does. In production a GitHub **webhook** to `argocd-server` removes the delay. Drift detection (Experiment 2) does not depend on polling, because the controller watches cluster objects and reacted in about 1 s.

**Q7. Where should Secrets go in a GitOps repo?**
Not in Git as plain or base64 values. Base64 is encoding, not encryption, and this repo is public. That is why I redacted the admin password from the evidence. Use **Sealed Secrets** (encrypted in Git, decrypted only by the in-cluster controller) or the **External Secrets Operator** (pulls from Vault or AWS Secrets Manager at runtime).

---

## 5. Reflection

> **Rewrite in your own words.** The prompts below come from what actually happened in this run.

**What was difficult / the error met.** Argo CD from a previous attempt had `argocd-applicationset-controller` in `CrashLoopBackOff` with 221 restarts. `kubectl logs` showed `no matches for kind "ApplicationSet"`, and `kubectl get crd | grep argoproj` listed only two of the three CRDs. The cause was the guide's client-side `kubectl apply`: the ApplicationSet CRD is too big for the `last-applied-configuration` annotation, so that one object failed while everything else succeeded. Re-applying with `--server-side --force-conflicts` fixed it.

**Second issue.** `kubectl port-forward ... 8080:443` failed with `bind: address already in use`. `ss -ltn` showed another process on port 8080, so I forwarded to 8443 instead.

**Third issue.** The re-apply pulled the newer `stable` release (v3.5.6 → v3.5.7), and the new dex and redis images took over 5 minutes to pull, so `kubectl wait --timeout=300s` timed out. Checking `kubectl get events` showed the Pods were still `Pulling` rather than failing. A floating `stable` URL is convenient, but pinning a version tag would make the install reproducible.

**What I would do differently.** Pin the Argo CD version in the install URL, use server-side apply from the start, and add a GitHub webhook so changes appear without a manual refresh.

**Still unclear.** How `selfHeal` interacts with a HorizontalPodAutoscaler that legitimately changes `replicas`. I believe `replicas` must be removed from Git or ignored with `ignoreDifferences`, but I have not tested it.

---

## 6. References

All accessed 2026-11-03.

1. DSO202 Practical 4 — GitOps and Argo CD Lab Guide: https://hackmd.io/@sarojsanyasi/BJIBWeT2Ml
2. Argo CD — Getting Started: https://argo-cd.readthedocs.io/en/stable/getting_started/
3. Argo CD — Automated Sync Policy (prune, selfHeal): https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/
4. Argo CD — Declarative Setup (Application CRD): https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/
5. Argo CD — Installation (server-side apply note for CRDs): https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/
6. OpenGitOps — GitOps Principles v1.0: https://opengitops.dev/
7. Kubernetes — Server-Side Apply: https://kubernetes.io/docs/reference/using-api/server-side-apply/
