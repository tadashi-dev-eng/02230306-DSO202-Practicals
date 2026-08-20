# DSO202 Practical 1

This repository contains a reproducible Kubernetes practical for creating, inspecting, exposing, testing, updating, failing, recovering, cleaning up, and rebuilding an nginx web workload.

The practical uses a local kind cluster with one control-plane node and two worker nodes. It covers namespaces, resource quotas, limit ranges, standalone Pods, Deployments, ReplicaSets, rolling updates, rollback, readiness, ClusterIP Services, NodePort Services, EndpointSlices, and an in-cluster client Pod.

## Repository Purpose

- `cluster/` contains the kind cluster definition.
- `manifests/` contains the numbered declarative Kubernetes objects.
- `evidence/` contains captured command output and practical evidence.
- `assets/` contains screenshots used by the report.
- `report/` contains the detailed practical report.

The numbered manifests are applied in lexical order. The namespace is created first, followed by namespace policies, workloads, and Services.

## Software Versions

The versions declared by this repository are:

- Kubernetes node image: `kindest/node:v1.36.1`
- Kubernetes API configuration: `kind.x-k8s.io/v1alpha4`
- Cluster name: `dso202`
- nginx workload image: `nginx:1.30-alpine`
- Deliberate update image: `nginx:1.31-alpine`
- Diagnostic client image: `busybox:1.37`
- Pod network: `10.244.0.0/16`
- Service network: `10.96.0.0/16`
- NodePort and host port: `30080`

The exact kind and kubectl client versions are not committed in this repository. The installed clients must support the pinned Kubernetes `v1.36.1` node image and the commands below. Verify them before starting:

```sh
docker --version
kind version
kubectl version --client
```

## Prerequisites

An empty Linux machine needs:

- Docker Engine with permission to run Docker commands.
- `kubectl` available on `PATH`.
- `kind` available on `PATH`.
- Git for pushing the code

The local machine must have enough CPU, memory, and disk space for three Docker-based Kubernetes nodes. Port `30080` must be free on `127.0.0.1`.

## Rebuild From an Empty Machine

The following is the complete command sequence after Docker, kubectl, kind, and Git have been installed.

### 1. Obtain the repository

Replace the placeholder URL with the repository's Git URL:

```sh
git clone <repository-url> dso202-practical-01
cd dso202-practical-01
```

If the repository is already present, use:

```sh
cd /Desktop/dso202-practical-01
```

### 2. Create and select the kind cluster

```sh
kind create cluster --config cluster/kind-cluster.yaml
kubectl config use-context kind-dso202
kubectl cluster-info
kubectl get nodes -o wide
```

The cluster contains `control-plane`, `worker-node-1`, and `worker-node-2`. The cluster definition publishes `127.0.0.1:30080` so the NodePort Service can later be tested from the host.

### 3. Apply the practical manifests

Apply the numbered directory in its declared dependency order:

```sh
kubectl apply -f manifests/00-namespace.yaml
kubectl apply -f manifests/01-quota-and-limits.yaml
kubectl apply -f manifests/02-pod-web.yaml
kubectl apply -f manifests/03-deployment-web.yaml
kubectl apply -f manifests/04-service-clusterip.yaml
kubectl apply -f manifests/05-service-nodeport.yaml
kubectl apply -f manifests/06-pod-client.yaml
```

Wait for the Deployment and inspect the resulting objects:

```sh
kubectl rollout status deployment/web-deployment -n dso202-practical-01
kubectl get pods,services,deployments,replicasets -n dso202-practical-01 -o wide
kubectl get endpointslice -n dso202-practical-01
```

The same rebuild can be performed with one directory command because the filenames are numbered:

```sh
kubectl apply -f manifests/
```

### 4. Verify Service access

Test the ClusterIP Service from the diagnostic client Pod:

```sh
kubectl exec -n dso202-practical-01 client-pod -- nslookup web-clusterip
kubectl exec -n dso202-practical-01 client-pod -- wget -qO- http://web-clusterip
```

Test the NodePort Service from the host:

```sh
curl -s http://127.0.0.1:30080
```

Inspect resource policy and final workload state:

```sh
kubectl get resourcequota,limitrange -n dso202-practical-01
kubectl get deployment web-deployment -n dso202-practical-01
kubectl get pods -n dso202-practical-01 --show-labels -o wide
```

## Cleanup Commands

Delete the objects using the same manifests that created them, in reverse dependency order:

```sh
kubectl delete -f manifests/06-pod-client.yaml
kubectl delete -f manifests/05-service-nodeport.yaml
kubectl delete -f manifests/04-service-clusterip.yaml
kubectl delete -f manifests/03-deployment-web.yaml
kubectl delete -f manifests/02-pod-web.yaml --ignore-not-found
kubectl delete -f manifests/01-quota-and-limits.yaml
kubectl delete -f manifests/00-namespace.yaml
```

Delete the kind cluster and its Docker containers:

```sh
kind delete cluster --name dso202
```

Confirm that the cluster is gone:

```sh
kind get clusters
docker ps -a --filter name=dso202
```

These commands remove the practical workload and the local cluster. They do not remove Docker, kubectl, kind, Git, the repository, or downloaded container images.
