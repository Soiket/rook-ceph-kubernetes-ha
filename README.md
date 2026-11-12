# Standard Operating Procedure (SOP)
## Kubernetes HA, Rook-Ceph, Ceph CSI and Portainer Deployment

**Document Version:** 1.0  
**Date:** 12 November 2026  
**Platform:** Ubuntu 24.04.5 LTS  
**Kubernetes:** v1.36.6  
**Container Runtime:** containerd 2.2.1  
**Rook:** v1.21.2  
**Ceph:** v19.2.3 (Squid)  
**Ceph CSI Operator:** v1.1.0  
**Deployment Method:** Helm and Kubernetes YAML manifests

---

# 1. Purpose

This SOP describes the installation, configuration, verification and basic troubleshooting of a three-node Kubernetes HA cluster with Rook-Ceph distributed storage.

It covers:

1. Kubernetes cluster prerequisite verification.
2. Helm installation and repository configuration.
3. Rook-Ceph operator deployment.
4. Ceph cluster deployment using Helm.
5. Ceph pool creation.
6. Ceph CSI Operator and RBD driver deployment.
7. Kubernetes StorageClass configuration.
8. Persistent Volume Claim (PVC) provisioning.
9. Ceph RBD volume mounting and testing.
10. Portainer deployment using Helm.
11. Storage and cluster verification.
12. Basic troubleshooting and operational commands.

# 2. Infrastructure Overview

## 2.1 Hardware and node information

| Node | Management IP | Role |
|---|---|---|
| k8s-storage-node1 | 10.10.10.86 | Kubernetes control plane, Ceph MON/MGR/OSD |
| k8s-storage-node2 | 10.10.10.87 | Kubernetes control plane, Ceph MON/MGR/OSD |
| k8s-storage-node3 | 10.10.10.88 | Kubernetes control plane, Ceph MON/MGR/OSD |

Each node has four 3.84 TB SSDs allocated for Ceph OSD storage.

**Total Ceph OSDs:** 12  
**OSDs per node:** 4

## 2.2 Software information

| Component | Version |
|---|---|
| Operating System | Ubuntu 24.04.5 LTS |
| Kubernetes | v1.36.6 |
| Container Runtime | containerd 2.2.1 |
| Rook-Ceph | v1.21.2 |
| Ceph | v19.2.3 |
| Ceph CSI Operator | v1.1.0 |
| CNI | Calico VXLAN |
| Helm | Helm 3 |
| Portainer | Portainer CE |

## 2.3 Network configuration

| Network | VLAN | Subnet / Address |
|---|---|---|
| Kubernetes management | VLAN 60 | 10.10.10.0/24 |
| Kubernetes API VIP | VLAN 60 | 10.10.10.100 |
| Ceph public network | VLAN 62 | 10.10.11.0/24 |
| Ceph cluster network | VLAN 61 | 10.10.12.0/24 |

Kubernetes API VIP: `10.10.10.100`

The API VIP is provided by Keepalived.

Ceph public and cluster networks are configured separately. The Ceph network provider is `host`.

## 2.4 Storage architecture

- Ceph is deployed through Rook.
- Four SSDs per node are used for Ceph OSDs.
- Ceph pools use three-way replication.
- Ceph CSI provides Kubernetes block storage.
- Portainer uses Ceph RBD-backed persistent storage.

# 3. Pre-deployment Verification

Before deploying Rook-Ceph, verify that Kubernetes is healthy.

## 3.1 Check Kubernetes version

Run:

```bash
kubectl version
```

Check nodes:

```bash
kubectl get nodes -o wide
```

Expected:

- Three nodes should be `Ready`.
- Kubernetes version should be `v1.36.6`.
- All nodes should be reachable.

## 3.2 Check Kubernetes system pods

```bash
kubectl get pods -n kube-system -o wide
```

Check Calico:

```bash
kubectl get pods -A | grep calico
```

Check API VIP:

```bash
kubectl get --raw='/readyz?verbose'
```

## 3.3 Check container runtime

Run on all three nodes:

```bash
containerd --version
```

Expected:

```text
containerd github.com/containerd/containerd 2.2.1
```

## 3.4 Check network connectivity

Check management connectivity:

```bash
ping -c 4 10.10.10.86
ping -c 4 10.10.10.87
ping -c 4 10.10.10.88
```

Check Ceph public network connectivity between nodes:

```bash
ping -c 4 10.10.11.86
ping -c 4 10.10.11.87
ping -c 4 10.10.11.88
```

Check Ceph cluster network connectivity:

```bash
ping -c 4 10.10.12.86
ping -c 4 10.10.12.87
ping -c 4 10.10.12.88
```

Use the actual Ceph interface IPs if they differ from the node numbering shown here.

## 3.5 Verify disks

Run on every storage node:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL,SERIAL
```

Check disk identifiers:

```bash
ls -l /dev/disk/by-id/
```

Check disk health:

```bash
sudo smartctl -a /dev/sdX
```

**Important:** Identify the exact four Ceph data disks on each node before deployment. Do not select operating-system disks or disks containing required data.

# 4. Install and Configure Helm

Helm is used to deploy Rook-Ceph and its components.

Check Helm:

```bash
helm version
```

If Helm is not installed, install it using the official Helm installation procedure. Verify the version afterward.

Add the Rook Helm repository:

```bash
helm repo add rook-release https://charts.rook.io/release
```

Update repositories:

```bash
helm repo update
```

Verify:

```bash
helm search repo rook-release
```

Check the Rook chart version:

```bash
helm show chart rook-release/rook-ceph --version 1.21.2
```

# 5. Deploy Rook-Ceph Operator Using Helm

The Rook operator manages Ceph resources inside Kubernetes.

## 5.1 Create the operator namespace

```bash
kubectl create namespace rook-ceph
```

If the namespace already exists, this command may report that it already exists. That is normal.

## 5.2 Prepare operator values

The deployment used:

```text
rook-operator-values.yaml
```

This values file contains the Rook operator scheduling and toleration configuration used for the Kubernetes nodes.

Preserve the original deployed values file.

You can retrieve the installed Helm values using:

```bash
helm get values rook-ceph \
  -n rook-ceph -a \
  > /root/rook-operator-values-backup.yaml
```

## 5.3 Install Rook operator

Run:

```bash
helm upgrade --install rook-ceph \
  rook-release/rook-ceph \
  --namespace rook-ceph \
  --create-namespace \
  --version 1.21.2 \
  -f rook-operator-values.yaml
```

## 5.4 Verify operator deployment

```bash
kubectl get pods -n rook-ceph -o wide
```

Check the deployment:

```bash
kubectl get deployment -n rook-ceph
```

Check Helm:

```bash
helm list -n rook-ceph
```

Expected:

```text
rook-ceph
```

The Rook operator should be running successfully before proceeding.

# 6. Deploy Ceph Cluster Using Helm

The Ceph cluster is deployed through the Rook Ceph cluster Helm chart.

## 6.1 Obtain the chart package

Update the repository:

```bash
helm repo update
```

Download and extract the chart:

```bash
helm pull rook-release/rook-ceph-cluster \
  --version 1.21.2 \
  --untar
```

This creates the local chart directory:

```text
rook-ceph-cluster/
```

## 6.2 Prepare Ceph cluster values

The deployed configuration file was:

```text
ceph-cluster-values.yaml
```

The configuration contains these important settings:

| Parameter | Configuration |
|---|---|
| Ceph image | quay.io/ceph/ceph:v19.2.3 |
| MON count | 3 |
| MGR count | 2 in the original chart configuration |
| Network provider | host |
| Public network | 10.10.11.0/24 |
| Cluster network | 10.10.12.0/24 |
| Storage nodes | 3 |
| OSD disks | 4 per node |
| Total OSDs | 12 |

The live cluster subsequently showed three MGR pods, including one active and two standbys. The original chart values and live resource state should be treated separately.

**Important:** The exact twelve `/dev/disk/by-id/` paths are hardware-specific and were defined in the original `ceph-cluster-values.yaml`. Preserve that original file or retrieve the current Helm release values before recreating the deployment. Do not substitute guessed disk paths.

Back up the deployed Helm values:

```bash
helm get values rook-ceph-cluster \
  -n rook-ceph -a \
  > /root/ceph-cluster-values-backup.yaml
```

Check the CephCluster resource:

```bash
kubectl get cephcluster -n rook-ceph
```

## 6.3 Install Ceph cluster

Run:

```bash
helm upgrade --install rook-ceph-cluster \
  ./rook-ceph-cluster \
  --namespace rook-ceph \
  -f ceph-cluster-values.yaml
```

## 6.4 Verify Ceph cluster deployment

Check CephCluster:

```bash
kubectl get cephcluster -n rook-ceph
```

Check pods:

```bash
kubectl get pods -n rook-ceph -o wide
```

Check MON pods:

```bash
kubectl get pods -n rook-ceph | grep mon
```

Check MGR pods:

```bash
kubectl get pods -n rook-ceph | grep mgr
```

Check OSD pods:

```bash
kubectl get pods -n rook-ceph | grep osd
```

Expected:

- Three MONs.
- Active MGR and standby MGRs.
- Twelve OSDs.
- OSD pods running across all three storage nodes.

# 7. Ceph Cluster Health Verification

Use the Rook tools deployment:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph -s
```

Check detailed health:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph health detail
```

Check OSD status:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd tree
```

Check OSD usage:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd df
```

Check MON quorum:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph quorum_status
```

Check pool list:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd pool ls detail
```

Expected final health:

```text
HEALTH_OK
```

Do not proceed with application storage provisioning while Ceph has unresolved critical health problems.

# 8. Create Kubernetes Ceph Block Pool

Create a dedicated pool for Kubernetes persistent volumes.

## 8.1 Create pool configuration

Create:

```bash
nano /root/kubernetes-pool.yaml
```

Paste:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: kubernetes
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
```

## 8.2 Apply the configuration

```bash
kubectl apply -f /root/kubernetes-pool.yaml
```

## 8.3 Verify pool

```bash
kubectl get cephblockpool -n rook-ceph
```

Check the pool:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool ls
```

Check replication:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool get kubernetes size
```

Check minimum replication size:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool get kubernetes min_size
```

Expected:

```text
size: 3
min_size: 2
```

# 9. Configure Ceph CSI Operator and RBD Driver

Ceph CSI connects Kubernetes persistent storage requests to Ceph RBD.

The deployed CSI components include:

- Ceph CSI Operator.
- RBD CSI Controller Plugin.
- RBD CSI Node Plugin.
- CephConnection.
- ClientProfile.
- CSI Driver resource.

## 9.1 Verify CSI Operator CRDs

```bash
kubectl get crd | grep csi.ceph.io
```

Expected CRDs include:

```text
cephconnections.csi.ceph.io
clientprofilemappings.csi.ceph.io
clientprofilereplications.csi.ceph.io
clientprofiles.csi.ceph.io
drivers.csi.ceph.io
operatorconfigs.csi.ceph.io
```

## 9.2 Verify CephConnection

The configured CephConnection:

```yaml
apiVersion: csi.ceph.io/v1
kind: CephConnection
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  monitors:
    - 10.10.11.88:3300
    - 10.10.11.86:3300
    - 10.10.11.87:3300
```

Check:

```bash
kubectl get cephconnection -n rook-ceph
```

Inspect:

```bash
kubectl get cephconnection rook-ceph \
  -n rook-ceph -o yaml
```

## 9.3 Verify ClientProfile

The deployed ClientProfile uses the CephConnection named `rook-ceph`.

Its RBD secret references are:

```text
Controller publish:
  rook-csi-rbd-provisioner

Node publish:
  rook-csi-rbd-node

Namespace:
  rook-ceph
```

Check:

```bash
kubectl get clientprofile -n rook-ceph
```

Inspect:

```bash
kubectl get clientprofile rook-ceph \
  -n rook-ceph -o yaml
```

Expected status:

```text
phase: Ready
message: ClientProfile reconciled successfully
```

Verify secret objects exist:

```bash
kubectl get secrets -n rook-ceph | grep rook-csi
```

Never display or share decoded Ceph secret keys.

## 9.4 Obtain the CSI driver Helm chart

The deployed CSI driver chart was:

```text
ceph-csi-drivers-1.1.0.tgz
```

Chart version:

```text
1.1.0
```

Inspect the local chart:

```bash
helm show chart /tmp/ceph-csi-drivers-1.1.0.tgz
```

The original deployment used the Rook CSI chart default values file:

```bash
curl -fsSL \
  https://raw.githubusercontent.com/rook/rook/v1.21.2/deploy/charts/ceph-csi-drivers/values.yaml \
  -o /root/rook-ceph-csi-values.yaml
```

## 9.5 Create CSI overrides

Create:

```bash
nano /root/ceph-csi-overrides.yaml
```

Paste:

```yaml
operatorConfig:
  driverSpecDefaults:
    controllerPlugin:
      replicas: 3

drivers:
  rbd:
    enabled: true
    name: rook-ceph.rbd.csi.ceph.com
    controllerPlugin:
      replicas: 3

  cephfs:
    enabled: false

  nfs:
    enabled: false

  nvmeof:
    enabled: false
```

This enables the RBD driver and disables the other drivers for this deployment.

## 9.6 Validate Helm templates

```bash
helm template ceph-csi-drivers \
  /tmp/ceph-csi-drivers-1.1.0.tgz \
  --namespace rook-ceph \
  -f /root/rook-ceph-csi-values.yaml \
  -f /root/ceph-csi-overrides.yaml \
  > /tmp/ceph-csi-rendered.yaml
```

Inspect:

```bash
grep -n "replicas:" /tmp/ceph-csi-rendered.yaml
```

## 9.7 Install CSI driver using Helm

```bash
helm upgrade --install ceph-csi-drivers \
  /tmp/ceph-csi-drivers-1.1.0.tgz \
  --namespace rook-ceph \
  -f /root/rook-ceph-csi-values.yaml \
  -f /root/ceph-csi-overrides.yaml \
  --wait \
  --timeout 10m
```

## 9.8 Verify CSI pods

```bash
kubectl get pods -n rook-ceph -o wide | grep csi
```

Expected deployment:

| Component | Expected |
|---|---|
| RBD Controller Plugin | 3 replicas |
| RBD Node Plugin | 3 pods |
| CSI Controller Manager | 3 replicas in the deployed environment |

Check CSI registration:

```bash
kubectl get csidrivers
```

Expected:

```text
rook-ceph.rbd.csi.ceph.com
```

Check Driver CR:

```bash
kubectl get drivers.csi.ceph.io -n rook-ceph
```

Check controller manager:

```bash
kubectl get deployment ceph-csi-controller-manager \
  -n rook-ceph
```

The three CSI Controller Manager replicas were observed in the deployed environment. A manual replica scaling operation is not necessarily persistent across reconciliation or Helm upgrades. Confirm the supported HA configuration and manage it through the owning configuration before relying on it for production.

# 10. Create Ceph RBD StorageClass

The StorageClass defines how Kubernetes dynamically provisions RBD volumes.

## 10.1 Create StorageClass with Delete policy

Create:

```bash
nano /root/ceph-rbd-sc.yaml
```

Paste:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: kubernetes
  imageFeatures: layering

  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph

  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph

  csi.storage.k8s.io/controller-publish-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-publish-secret-namespace: rook-ceph

  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

  csi.storage.k8s.io/node-publish-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-publish-secret-namespace: rook-ceph

  csi.storage.k8s.io/fstype: ext4

reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

**Important:** `clusterID: rook-ceph` refers to the ClientProfile name used in this CSI Operator setup, not the Ceph FSID.

## 10.2 Apply StorageClass

```bash
kubectl apply -f /root/ceph-rbd-sc.yaml
```

Verify:

```bash
kubectl get storageclass
```

Inspect:

```bash
kubectl describe storageclass ceph-rbd
```

# 11. Create Retain StorageClass

For important application data, use a separate StorageClass with a `Retain` reclaim policy.

Create:

```bash
nano /root/ceph-rbd-retain.yaml
```

Paste:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd-retain
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: kubernetes
  imageFeatures: layering

  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph

  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph

  csi.storage.k8s.io/controller-publish-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-publish-secret-namespace: rook-ceph

  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

  csi.storage.k8s.io/node-publish-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-publish-secret-namespace: rook-ceph

  csi.storage.k8s.io/fstype: ext4

reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

Apply:

```bash
kubectl apply -f /root/ceph-rbd-retain.yaml
```

Verify:

```bash
kubectl get storageclass
```

The `Retain` policy protects the underlying volume from automatic deletion when the PVC is deleted. It does not replace a backup.

# 12. Test Dynamic PVC Provisioning

## 12.1 Create a test PVC

Create:

```bash
nano /root/ceph-rbd-pvc.yaml
```

Paste:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ceph-rbd-test
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-rbd
  resources:
    requests:
      storage: 10Gi
```

Apply:

```bash
kubectl apply -f /root/ceph-rbd-pvc.yaml
```

Check:

```bash
kubectl get pvc
```

Check PV:

```bash
kubectl get pv
```

Expected:

```text
STATUS: Bound
CAPACITY: 10Gi
STORAGECLASS: ceph-rbd
```

# 13. Test Ceph RBD Volume Mounting

## 13.1 Create test pod

Create:

```bash
nano /root/ceph-rbd-test-pod.yaml
```

Paste:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ceph-rbd-test-pod
  namespace: default
spec:
  containers:
    - name: test
      image: busybox:1.36
      command: ["sh", "-c", "sleep 36000"]
      volumeMounts:
        - name: ceph-volume
          mountPath: /data
  volumes:
    - name: ceph-volume
      persistentVolumeClaim:
        claimName: ceph-rbd-test
```

Apply:

```bash
kubectl apply -f /root/ceph-rbd-test-pod.yaml
```

Wait for pod readiness:

```bash
kubectl wait --for=condition=Ready pod/ceph-rbd-test-pod \
  --timeout=180s
```

## 13.2 Test file write

```bash
kubectl exec ceph-rbd-test-pod -- sh -c \
  'echo "Ceph RBD storage is working" > /data/test.txt'
```

Read file:

```bash
kubectl exec ceph-rbd-test-pod -- cat /data/test.txt
```

Check filesystem:

```bash
kubectl exec ceph-rbd-test-pod -- df -h /data
```

Expected filesystem:

```text
/dev/rbd0
```

## 13.3 Verify volume attachment

```bash
kubectl get volumeattachment
```

Check PV details:

```bash
kubectl describe pv \
  $(kubectl get pvc ceph-rbd-test -o jsonpath='{.spec.volumeName}')
```

The PV should identify the CSI driver:

```text
rook-ceph.rbd.csi.ceph.com
```

# 14. Deploy Portainer Using Helm

Portainer provides a graphical management interface for Kubernetes.

## 14.1 Add Portainer Helm repository

```bash
helm repo add portainer https://portainer.github.io/k8s/
```

Update:

```bash
helm repo update
```

Inspect chart:

```bash
helm show values portainer/portainer
```

Check persistence configuration:

```bash
helm show values portainer/portainer | grep -A15 '^persistence:'
```

## 14.2 Install Portainer with Ceph RBD storage

The deployed Portainer installation uses the `ceph-rbd-retain` StorageClass.

Run:

```bash
helm upgrade --install --create-namespace \
  -n portainer \
  portainer portainer/portainer \
  --set persistence.enabled=true \
  --set persistence.storageClass=ceph-rbd-retain \
  --set persistence.size=10Gi \
  --set service.type=NodePort
```

## 14.3 Verify Portainer deployment

Check pods:

```bash
kubectl get pods -n portainer -o wide
```

Check PVC:

```bash
kubectl get pvc -n portainer
```

Check PV:

```bash
kubectl get pv
```

Check service:

```bash
kubectl get svc -n portainer
```

Check volume attachment:

```bash
kubectl get volumeattachment
```

Expected:

- Portainer pod running.
- Portainer PVC `Bound`.
- PV `Bound`.
- StorageClass `ceph-rbd-retain`.
- Volume attachment `true`.

## 14.4 Access Portainer

The deployed NodePort service reported:

| Service port | NodePort |
|---|---|
| HTTP 9000 | 30777 |
| HTTPS 9443 | 30779 |
| Additional port | 30776 |

Access through HTTPS:

```text
https://10.10.10.86:30779
```

Other node addresses can also be used:

```text
https://10.10.10.87:30779
https://10.10.10.88:30779
```

Use HTTPS for administration. Confirm firewall rules permit access to the selected NodePort.

# 15. Verify Portainer Uses Ceph RBD

This is an important post-installation validation.

## 15.1 Check Portainer PVC

```bash
kubectl get pvc -n portainer
```

## 15.2 Describe the Portainer PV

Get PV name:

```bash
kubectl get pvc portainer -n portainer \
  -o jsonpath='{.spec.volumeName}{"\n"}'
```

Describe it:

```bash
kubectl describe pv \
  $(kubectl get pvc portainer -n portainer -o jsonpath='{.spec.volumeName}')
```

Confirm:

```text
StorageClass: ceph-rbd-retain
Reclaim Policy: Retain
Driver: rook-ceph.rbd.csi.ceph.com
pool=kubernetes
```

## 15.3 Obtain RBD image name

```bash
kubectl get pv \
  $(kubectl get pvc portainer -n portainer -o jsonpath='{.spec.volumeName}') \
  -o jsonpath='{.spec.csi.volumeHandle}{"\n"}'
```

Inspect the complete PV:

```bash
kubectl get pv \
  $(kubectl get pvc portainer -n portainer -o jsonpath='{.spec.volumeName}') \
  -o yaml
```

Look for:

```text
imageName
pool: kubernetes
clusterID: rook-ceph
```

## 15.4 Check actual RBD image in Ceph

The deployed RBD image was:

```text
csi-vol-c85e0a1d-a784-49a0-8f97-8573d202311d
```

Inspect:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  rbd info kubernetes/csi-vol-c85e0a1d-a784-49a0-8f97-8573d202311d
```

Check replication:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool get kubernetes size
```

Expected:

```text
size: 3
```

This confirms that the Portainer persistent disk is a real Ceph RBD image in the Kubernetes pool.

# 16. Important Operational Commands

## 16.1 Kubernetes health

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get events -A --sort-by='.lastTimestamp'
kubectl get componentstatuses
```

Note: `kubectl get componentstatuses` is deprecated and may not provide useful health information on modern Kubernetes versions. Prefer node and control-plane pod checks.

## 16.2 Rook-Ceph health

```bash
kubectl get cephcluster -n rook-ceph
kubectl get pods -n rook-ceph -o wide
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph -s
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph health detail
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd tree
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd df
```

## 16.3 CSI health

```bash
kubectl get csidrivers
kubectl get drivers.csi.ceph.io -n rook-ceph
kubectl get pods -n rook-ceph | grep csi
kubectl get volumeattachment
```

## 16.4 Storage health

```bash
kubectl get storageclass
kubectl get pvc -A
kubectl get pv
```

## 16.5 Helm release status

```bash
helm list -A
```

Inspect Rook operator release:

```bash
helm status rook-ceph -n rook-ceph
```

Inspect Ceph cluster release:

```bash
helm status rook-ceph-cluster -n rook-ceph
```

Inspect CSI driver release:

```bash
helm status ceph-csi-drivers -n rook-ceph
```

Inspect Portainer release:

```bash
helm status portainer -n portainer
```

# 17. Troubleshooting Procedures

## 17.1 Ceph OSD is down

Check:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd tree
```

Check health:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph health detail
```

Check OSD pods:

```bash
kubectl get pods -n rook-ceph -o wide | grep osd
```

Inspect the affected pod:

```bash
kubectl describe pod -n rook-ceph <OSD-POD-NAME>
```

Check logs:

```bash
kubectl logs -n rook-ceph <OSD-POD-NAME> --all-containers
```

For a hardware failure, identify the affected physical disk, check its health and follow the approved disk-replacement procedure. Do not delete OSD resources or wipe disks without verifying the affected OSD and its recovery plan.

## 17.2 PVC stuck in Pending

Check:

```bash
kubectl describe pvc <PVC-NAME> -n <NAMESPACE>
```

Check CSI pods:

```bash
kubectl get pods -n rook-ceph | grep csi
```

Check CSI controller logs:

```bash
kubectl logs -n rook-ceph \
  deployment/rook-ceph.rbd.csi.ceph.com-ctrlplugin \
  --all-containers --tail=100
```

The deployment name can vary. Use `kubectl get deployments -n rook-ceph` to confirm it first.

Check StorageClass:

```bash
kubectl describe storageclass ceph-rbd
```

Check Ceph pool:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool ls
```

## 17.3 PVC is Bound but pod cannot mount volume

Check:

```bash
kubectl describe pod <POD-NAME> -n <NAMESPACE>
```

Check attachment:

```bash
kubectl get volumeattachment
```

Check node plugin pods:

```bash
kubectl get pods -n rook-ceph -o wide | grep nodeplugin
```

Check events:

```bash
kubectl get events -n <NAMESPACE> \
  --sort-by='.lastTimestamp'
```

## 17.4 Portainer pod is not starting

Check:

```bash
kubectl get pods -n portainer -o wide
```

Describe pod:

```bash
kubectl describe pod -n portainer \
  -l app.kubernetes.io/name=portainer
```

Check logs:

```bash
kubectl logs -n portainer deployment/portainer --tail=100
```

Check PVC:

```bash
kubectl get pvc -n portainer
```

Check volume attachment:

```bash
kubectl get volumeattachment
```

# 18. Backup and Data Protection Recommendations

The Ceph RBD `Retain` reclaim policy protects against automatic volume deletion when a PVC is removed. It does not provide an independent backup.

For production:

1. Maintain regular backups of Portainer persistent data.
2. Keep backups outside the primary Ceph failure domain.
3. Test restore procedures periodically.
4. Monitor Ceph health and capacity.
5. Monitor OSD, MON, MGR and CSI availability.
6. Keep Helm values and YAML manifests under version control.
7. Preserve the original disk inventory and deployment configuration.
8. Avoid storing only one copy of critical configuration files.

# 19. Deployment Validation Checklist

Use this checklist after deployment.

- [ ] Three Kubernetes nodes are Ready.
- [ ] Kubernetes API VIP is reachable.
- [ ] Calico pods are healthy.
- [ ] Helm is installed and working.
- [ ] Rook operator Helm release is deployed.
- [ ] Rook Ceph cluster Helm release is deployed.
- [ ] CephCluster reports Ready.
- [ ] Three Ceph MONs are running.
- [ ] MGR active and standby services are running.
- [ ] Twelve Ceph OSDs are up and in.
- [ ] Ceph health is HEALTH_OK.
- [ ] Kubernetes CephBlockPool is Ready.
- [ ] Pool replication size is 3.
- [ ] Ceph CSI RBD driver is registered.
- [ ] CSI controller plugins are running.
- [ ] CSI node plugins are running on all three nodes.
- [ ] ClientProfile reports Ready.
- [ ] Ceph RBD StorageClass is created.
- [ ] Test PVC reaches Bound.
- [ ] Test pod mounts the RBD volume.
- [ ] File write and read tests succeed.
- [ ] Portainer Helm release is deployed.
- [ ] Portainer PVC reaches Bound.
- [ ] Portainer PV uses the Ceph CSI driver.
- [ ] Portainer RBD image is present in the Ceph pool.
- [ ] Portainer service is accessible through HTTPS.

# 20. Important Production Considerations

1. A three-way replicated Ceph pool stores three copies across failure domains. The pool uses `failureDomain: host`, so replicas are distributed across different storage hosts when placement is healthy.
2. Ceph replication is not a replacement for independent backups.
3. Portainer currently uses one application replica. Its Ceph-backed persistent disk provides storage resilience, but does not by itself make the Portainer application highly available.
4. The RBD CSI node plugin is deployed on all three Kubernetes nodes.
5. The CSI Controller Plugin has three replicas. The separate CSI Operator Controller Manager also has three replicas in the current deployment; verify its supported leader-election and persistent management configuration.
6. The test StorageClass `ceph-rbd` uses `Delete`. The Portainer StorageClass `ceph-rbd-retain` uses `Retain`.
7. Do not remove the Rook operator, CephCluster, CSI resources or Ceph pools as a troubleshooting shortcut.
8. Do not run disk cleanup or wipe commands against Ceph OSD disks unless following an approved replacement or decommissioning procedure.

# 21. Final Deployment Architecture

```text
                   Kubernetes API VIP
                     10.10.10.100
                           |
             +-------------+-------------+
             |             |             |
        K8s Node 1    K8s Node 2    K8s Node 3
        10.10.10.86   10.10.10.87   10.10.10.88
             |             |             |
             +-------------+-------------+
                           |
                  Kubernetes Cluster
                    Calico VXLAN
                           |
                  Rook-Ceph Operator
                           |
                   Ceph Cluster
             +-------------+-------------+
             |             |             |
           MON 1         MON 2         MON 3
           MGR           MGR           MGR
             |             |             |
          4 OSDs        4 OSDs        4 OSDs
             +-------------+-------------+
                           |
                  Ceph Pool: kubernetes
                     Size: 3
                           |
                    Ceph CSI RBD
                           |
                  Kubernetes StorageClass
                           |
                    PersistentVolume
                           |
                 PersistentVolumeClaim
                           |
                    Portainer Pod
```

**Deployment result:** A three-node Kubernetes cluster with Rook-managed Ceph storage, dynamically provisioned Ceph RBD volumes through CSI, and Portainer using Ceph-backed persistent storage.

---

**Document maintenance:** Update this SOP whenever Kubernetes, Rook, Ceph, CSI, Helm chart versions, network settings, disk inventory or deployment values change.
