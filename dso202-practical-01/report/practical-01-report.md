# DSO202 — Practical 1 Report
### Setting Up a Local Kubernetes Cluster with kind, and Deploying First Workloads

## Stage 1 : 
Created the cluster using the command where it spins up three docker containers that will be working like a Kubernetes node, one control pane node and two worker nodes. 
```cmd 
kind create cluster --config cluster/kind-cluster.yaml
```
Listed all the clusters to confirm that cluster is registered. 

![alt text](../evidence/p1.png)

---
This three confirms that the three container proofs that the Kubernetes cluster is the docker container

![alt text](../evidence/p2.png)

---
kubectl is pointing to the new cluster .
```cmd 
tadashi@tadashi:~/Desktop/dso202-practicals$ kubectl config current-context
kind-dso202
tadashi@tadashi:~/Desktop/dso202-practicals$ 
```
---
## Stage 2 : 
Checking the clusters where the API server is listening which include the IP address and port.

![alt text](../evidence/cluster-info.png)

---
Listing every nodes in the cluster and theie status ( Ready/NotReady). Plus listed the internal IP, OS image, kernal version, container runtime on the nodes in the cluster.

![alt text](../evidence/p3.png)

--- 
The lables includes dso202/node-role=worker, dso202/node-index=1
```cmd 
tadashi@tadashi:~/Desktop/dso202-practicals$ kubectl describe node worker-node-1
Name:               worker-node-1
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    dso202/node-index=1
                    dso202/node-role=worker
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=worker-node-1
                    kubernetes.io/os=linux
```
Shows the total CPU and memory the node has and shows how much the Kubernetes will hand out to the pod that is scheduled.
```cmd 
Capacity:
  cpu:                8
  ephemeral-storage:  65739308Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3715880Ki
  pods:               110

Allocatable:
  cpu:                8
  ephemeral-storage:  65739308Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3715880Ki
  pods:               110
```
It shows the every pod currently placed on the node with its CPU and memory requests.

```cmd 
Non-terminated Pods:          (2 in total)
  Namespace                   Name                CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                ------------  ----------  ---------------  -------------  ---
  kube-system                 kindnet-fwwvw       100m (1%)     100m (1%)   50Mi (1%)        50Mi (1%)      5d12h
  kube-system                 kube-proxy-k5kgx    0 (0%)        0 (0%)      0 (0%)           0 (0%)         5d12h
```