# Kubernetes HA with Rook-Ceph

Three-node Kubernetes HA cluster with Rook-Ceph distributed storage,
Ceph CSI RBD integration, and Portainer deployment.

## Environment

| Component | Version |
|---|---|
| Ubuntu | 24.04.5 LTS |
| Kubernetes | v1.36.6 |
| containerd | 2.2.1 |
| Rook-Ceph | v1.21.2 |
| Ceph | v19.2.3 |
| Ceph CSI Operator | v1.1.0 |
| CNI | Calico VXLAN |
| Deployment | Helm |

## Nodes

| Node | IP Address |
|---|---|
| k8s-storage-node1 | 10.10.10.86 |
| k8s-storage-node2 | 10.10.10.87 |
| k8s-storage-node3 | 10.10.10.88 |

## Repository Structure

- `01-rook-operator/` - Rook operator Helm configuration
- `02-ceph-cluster/` - Ceph cluster Helm configuration
- `03-ceph-pools/` - Ceph block pool configuration
- `04-ceph-csi/` - Ceph CSI Helm values and overrides
- `05-storage-classes/` - Ceph RBD StorageClasses
- `06-backup/` - Configuration and deployment backups

## Storage Architecture

- 3 Ceph MONs
- 3 Ceph MGRs
- 12 Ceph OSDs
- 3-way replicated Kubernetes pool
- Ceph RBD CSI controller and node plugins
- Portainer with Ceph RBD persistent storage

## Deployment

Refer to this README and the associated YAML files for deployment
configuration. Use the matching Helm chart versions.

## Important

This repository contains deployment configuration only.
It is not a backup of Ceph data, RBD images, Kubernetes etcd,
or application persistent data.

Review all configuration files before committing.
Never commit Ceph keys, passwords, Kubernetes credentials,
or other sensitive information.
