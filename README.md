# Documentation Jarvis-Lab (Homelab Robert Guske)

Manifest as well as other lab relevant files are located in the respective [REPOSITORY](https://github.com/rguske/jarvislab) on Github.

![cover](/assets/cover-jarvis-lab-ironman.png)

- [Documentation Jarvis-Lab (Homelab Robert Guske)](#documentation-jarvis-lab-homelab-robert-guske)
  - [Networking](#networking)
  - [Storage](#storage)
    - [NFS CSI Driver](#nfs-csi-driver)
    - [Local Storage](#local-storage)
  - [Customization](#customization)

## Networking

![jarvislab](/assets/jarvislab-network.png)

## Storage

### NFS CSI Driver

```code
# Add Helm repo
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts

# List versions
helm search repo -l csi-driver-nfs
```

Install the NFS provisioner:

```code
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --version 4.11.0 \
  --create-namespace \
  --namespace csi-driver-nfs \
  --set controller.runOnControlPlane=true \
  --set controller.replicas=2 \
  --set controller.strategyType=RollingUpdate \
  --set externalSnapshotter.enabled=true \
  --set externalSnapshotter.customResourceDefinitions.enabled=false
```

For a SNO setup:

```code
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --version 4.11.0 \
  --create-namespace \
  --namespace csi-driver-nfs \
  --set controller.runOnControlPlane=true \
  --set controller.strategyType=RollingUpdate \
  --set externalSnapshotter.enabled=true \
  --set externalSnapshotter.customResourceDefinitions.enabled=false
```

Grant additional permissions to the ServiceAccounts:

`oc adm policy add-scc-to-user privileged -z csi-nfs-node-sa -n csi-driver-nfs`

`oc adm policy add-scc-to-user privileged -z csi-nfs-controller-sa -n csi-driver-nfs`

Create a StorageClass:

`oc apply -f https://raw.githubusercontent.com/rguske/jarvislab/refs/heads/main/manifest/storage/nfs-storageclass.yaml`

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: nfs.csi.k8s.io
parameters:
  server: 10.10.42.20   ### NFS server's IP/FQDN
  share: /volume1/nfs_ds/ocp             ### NFS server's exported directory
  subDir: ${pvc.metadata.namespace}-${pvc.metadata.name}-${pv.metadata.name}  ### Folder/subdir name template
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

Create a SnapshotClass:

`oc apply -f https://raw.githubusercontent.com/rguske/jarvislab/refs/heads/main/manifest/storage/nfs-volumesnapshotclass.yaml`

```yaml
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
deletionPolicy: Delete
driver: nfs.csi.k8s.io
metadata:
  name: csi-nfs-snapclass
```

Set the StorageClass to `default`:

`oc annotate storageclass/nfs-csi storageclass.kubernetes.io/is-default-class=true`

### Local Storage

Option 1:

[Installing the Local Storage Operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/storage/configuring-persistent-storage#local-storage-install_persistent-storage-local)

Option 2:

[Logical Volume Manager Storage installation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/storage/configuring-persistent-storage)

Installation via yaml:

`oc apply -f https://raw.githubusercontent.com/rguske/jarvislab/refs/heads/main/manifest/storage/lvm-storage-operator.yaml`

Via Operator [Web Console](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/storage/configuring-persistent-storage#lvms-installing-lvms-with-web-console_logical-volume-manager-storage)

Install the Logical Volume Cluster only including the SSD with the `by-path` identifier:

`ls -li /dev/disk/by-path`

`oc apply -f https://raw.githubusercontent.com/rguske/jarvislab/refs/heads/main/manifest/storage/lvmcluster.yaml`

Create a test `pvc`:

```yaml
oc create -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lvm-block-1
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 10Gi
    limits:
      storage: 20Gi
  storageClassName: lvms-vg1
EOF
```

## Customization

Login screen customized.

![login-screen](/assets/login-screen.png)