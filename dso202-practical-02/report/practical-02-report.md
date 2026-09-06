
## Step 5.2 Verification steps

### Step 1

Checking the required CLI tools like docker, kind, and kubectl are installed, accessible in the system path and functioning properly.

![alt text](../assets/step1.png)

Output shows the version strings for Docker, Kind and the kubectl client without throwing any errors.

### step 3
Created the ``/tmp/dso202-p2-storage`` directory on the host machine. This folder will serve as host-path persistent storage mounted directly into the Kind cluster nodes.

![alt text](../assets/step3.png)

The directory is created without errors. Running  ``ls -ld /tmp/dso202-p2-storage`` shows the directory details and confirms standard user read/write/execute permissions as drwxr-xr-x.

### Step 4
Available storage on the partition where Docker images and containers reside to ensure there is enough disk space for new cluster nodes and images.

![alt text](../assets/step4.png)

The output shows the available disk space on the host system.

## Stage 1 - Cluster, Namespace, and the Storage Landscape

### step 1

Deployed a multi-node Kubernetes cluster inside Docker containers using kind and the specified configuration file (kind-cluster.yaml).

![alt text](../assets/kind-cluster.png)

The kind automatically installs a default local storage provisioner add-on alongside essential components like the Container Network Interface (CNI).

### step 2 
Listed the cluster nodes using kubectl and compares them against running Docker containers using docker ps.

![alt text](../assets/kind-get.png)

### step 3
Uses docker exec to run ls -ld inside the worker-node-1 container, verifying that the host directory created earlier was successfully mounted to /mnt/dso202-static.

![alt text](../assets/mnt.png)

## 6.2 Create the namespace, the quota, and the retaining StorageClass

### Step 4
Applied core manifests to create the working namespace  as dso202-practical-02 and set it as the active context default. Created both a ResourceQuota and a custom StorageClass as dso202-retain.

![alt text](../assets/namespace.png)

The active namespace shortens subsequent kubectl commands by eliminating the need to pass -n dso202-practical-02 every time.

### Step 5
Used kubectl describe resourcequota to inspect the newly created resource constraints.

![alt text](../assets/resource-quota.png)

The command Verifies that the storage enforcement limits are active specifically cap constraints on total PVC count, overall requested storage (20Gi) and per-StorageClass storage caps.

## 6.3 Locate the provisioner

### step 6
Listed all StorageClass resources in the cluster to identify installed provisioners and compare configuration parameters.

![alt text](../assets/storageclass.png)

The output compares the default standard class against the new dso202-retain class. It highlights key storage policies: PROVISIONER, RECLAIMPOLICY, VOLUMEBINDINGMODE and ALLOWVOLUMEEXPANSION is (false).

### Step 7
Searched for running pods in the local path storage system namespace to find the active storage controller.

![alt text](../assets/pods.png)

It identifies the exact running pod that actively watches for PersistentVolumeClaims and creates the physical backing storage directories on the nodes.

### Step 8 
It queries the local-path-config ConfigMap in the local path storage namespace using jsonpath to extract the host path where dynamic volumes will be stored.

![alt text](../assets/json.png)

It determines the authoritative on-node directory (/var/local-path-provisioner) where all dynamically provisioned persistent volumes will actually reside when created by the provisioner.

## Stage 2 — Static Provisioning, and the Meaning of Retain

### 7.1 Create the PersistentVolume


### Step 1

Applied PV Manifest and Inspect Status. It defines and registers the static PersistentVolume (pv-web-static) in the Kubernetes API, then checks its current phase.

![alt text](../assets/PV.png)

The PV status displays as Available meaning it exists in the cluster state and is waiting for a matching PersistentVolumeClaim to claim it.

