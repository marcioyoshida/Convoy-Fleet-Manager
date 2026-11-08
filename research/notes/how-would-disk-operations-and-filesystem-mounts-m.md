# How would disk operations and filesystem mounts m

> Source note (PDF, 2 pages). AI-generated take collected during the Convoy design debate; converted to text verbatim. The critical review lives in `docs/debate-review.md`.

In a traditional POSIX environment, disk operations (mount, umount, mkfs) interact directly with 
physical or virtual block devices exposed by the operating system kernel. In a 
Kubernetes-backed shell, these commands must be translated into declarative storage 
abstractions because pods are ephemeral and nodes can come and go. 
Instead of managing raw partition tables (fdisk), the shell translates filesystem commands into 
Persistent Volume Claims (PVCs) and StorageClasses, turning storage provisioning into an 
on-demand, API-driven operation. 
## Core Mappings: POSIX Storage to Kubernetes Primitives 
Traditional POSIX 
Concept 
Kubernetes Translation 
Primitive 
How It Works Under the 
Hood 
Block Device (/dev/sdX) 
PersistentVolume (PV) 
A cluster-scoped storage 
resource provisioned from 
cloud disks (EBS, GCP 
Persistent Disk), local 
NVMe, or network storage 
(NFS/Ceph). 
mount /dev/disk /mnt/data 
PersistentVolumeClaim 
(PVC) + volumeMounts 
Binds a specific storage 
profile and capacity to a 
container path (/mnt/data) 
within the active process 
namespace. 
Filesystem Type (-t ext4, -t 
xfs) 
StorageClass (provisioner, 
parameters) 
Defines the underlying 
driver, file system type, and 
performance characteristics 
(e.g., IOPS, replication tier). 
umount / Detaching Disk 
PVC Deletion or Pod 
Termination 
Safely unbinds the storage. 
Depending on the reclaim 
policy (Retain or Delete), 
data is preserved or wiped. 
## 1. Dynamic Provisioning: Translating mount 
When a user runs a mount command in this shell—for example, requesting a high-performance 
disk—the shell doesn't look for a local SCSI device. Instead, it generates a dynamic PVC 
manifest and applies it to the cluster API server. 
●​ POSIX Command:​

Bash​
mount -t ssd -s 50Gi /dev/storage /app/data​
 
●​ What the Shell Compiles & Applies: The shell translates -t ssd into a corresponding 
StorageClass (e.g., gp3-fast) and -s 50Gi into a 50-gigabyte storage request. Kubernetes 
provisions the cloud volume, attaches it to the node running the pod, and mounts it at 
/app/data. 
## 2. Handling Shared and Network Disks (ReadWriteMany) 
In standard Linux, you can mount an NFS share across multiple servers using standard network 
mount syntax. In Kubernetes, this maps directly to Access Modes in PVCs. 
●​ Single-Node Exclusive (RWO - ReadWriteOnce): Maps to local SSDs or standard block 
storage where only one "process" (pod) can write to the disk at a time. 
●​ Shared Storage (RWX - ReadWriteMany): Maps to distributed file systems like NFS, 
CephFS, or AWS EFS. When a user runs a shared mount command, the shell provisions 
a PVC with ReadWriteMany access, allowing multiple distributed "processes" (pods in a 
deployment) to read and write to the same directory concurrently without race conditions. 
## 3. Ephemeral vs. Persistent Disk Lifecycles 
Traditional shells treat temporary directories (/tmp) as ephemeral space tied to the running host. 
In a containerized Kubernetes environment: 
●​ Ephemeral Storage (emptyDir): If a user writes to standard container scratch space, the 
shell maps it to an ephemeral Kubernetes volume. When the "process" (pod) dies, the 
data is wiped. 
●​ Persistent Storage (PVC): If a user explicitly mounts a persistent volume, the lifecycle of 
the data is decoupled from the process. Even if the pod crashes, gets rescheduled to 
another physical node, or is terminated (kill -9), restarting the shell command and 
re-mounting the same PVC reconnects the application to its exact state.
