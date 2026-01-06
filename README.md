# Documentation Jarvis-Lab (Homelab Robert Guske)

Manifest as well as other lab relevant files are located in the respective [REPOSITORY](https://github.com/rguske/jarvislab) on Github.

![cover](/assets/cover-jarvis-lab-ironman.png)

- [Documentation Jarvis-Lab (Homelab Robert Guske)](#documentation-jarvis-lab-homelab-robert-guske)
  - [Networking](#networking)
  - [Storage](#storage)
    - [NFS CSI Driver](#nfs-csi-driver)
    - [Local Storage](#local-storage)
    - [Synology CSI](#synology-csi)
  - [Authentication](#authentication)
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

### Synology CSI

Synology CSI on Github - [HERE](https://github.com/SynologyOpenSource/synology-csi)

Protocol summary by [Blog xphyr - OCP with Synology](https://xphyr.net/post/ocp_synology_csi_2025/)

> iSCSI - best for Block storage, allowing raw block devices to be accessed by multiple servers at once. This feature allows for things like the migration of virtual machines from one node to another with Kubevirt and OpenShift Virtualization. It also supports file access, however only one node can access the storage at a time when the volume has a filesystem on it.
NFS - NFS is best for when you have a filesystem that needs to be accessed by multiple pods, or applications at once. The NFS protocol ensures that file access and locking is in place to keep file corruption from occurring. Typically NFS is used in Linux/Unix systems but it is available in Windows as well. It is recommended that you use version 4.1 of the NFS protocol.
SMB - SMB is the default file sharing protocol for Windows, but it also works with Linux/Unix systems. In theory this should work with things like Windows Containers, however I have not yet tested this. Stay tuned for a future post on this.

| Protocol     | RWX | RWO | Cloning | Expansion | Snapshots |
|---------------|-----|-----|----------|------------|------------|
| iSCSI-block   | X   | X   | X        | X          | X          |
| iSCSI-file    | -   | X   | X        | X          | X          |
| NFS           | X   | X   | X        | X          | X          |
| SMB           | X   | X   | X        | X          | X          |

```code
git clone https://github.com/SynologyOpenSource/synology-csi.git
cd synology-csi
```

Configuration file:

```code
cp config/client-info-template.yml config/client-info.yml
vi config/client-info.yml
```

```yaml
---
clients:
  - host: 10.10.42.20
    port: 5000
    https: false
    username: synology-csi-sa
    password: 'R3dh4t1!'
  - host: 10.10.42.20
    port: 5001
    https: true
    username: synology-csi-sa
    password: '****'
```

```code
oc new-project synology-csi
Now using project "synology-csi" on server

oc create secret generic client-info-secret --from-file=./config/client-info.yml
secret/client-info-secret created
```

```code
mv deploy/kubernetes/v1.20/storage-class.yml config/storage-class-iscsi.yml
```

* Create SCC

```yaml
oc create -f - <<EOF
---
kind: SecurityContextConstraints
apiVersion: security.openshift.io/v1
metadata:
  name: synology-csi-scc
allowHostDirVolumePlugin: true
allowHostNetwork: true
allowPrivilegedContainer: true
allowedCapabilities:
- 'SYS_ADMIN'
defaultAddCapabilities: []
fsGroup:
  type: RunAsAny
groups: []
priority:
readOnlyRootFilesystem: false
requiredDropCapabilities: []
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
users:
- system:serviceaccount:synology-csi:csi-controller-sa
- system:serviceaccount:synology-csi:csi-node-sa
- system:serviceaccount:synology-csi:csi-snapshotter-sa
volumes:
- '*'
EOF
```

* Deploy the Synology Provisioning Pods

```code
oc create -f deploy/kubernetes/v1.20/

oc create -f deploy/kubernetes/v1.20/snapshotter/
```

* Storageclasses

For NFS:

```yaml
oc create -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: synology-nfs-storage
provisioner: csi.san.synology.com
parameters:
  protocol: "nfs"
  dsm: '172...'
  location: '/volume1'
  mountPermissions: '0775'
mountOptions:
  - nfsvers=4.1
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF
```

For iSCSI:

```yaml
oc create -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: synology-iscsi-storage
provisioner: csi.san.synology.com
parameters:
  dsm: '172.16.20.13'
  location: '/volume1'
  csi.storage.k8s.io/fstype: 'ext4'
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF
```

* Default StorageClass

```code
oc annotate storageclass synology-iscsi-storage storageclass.kubernetes.io/is-default-class=true
storageclass.storage.k8s.io/synology-iscsi-storage annotated
```

* Configuring the default StorageProfile for KubeVirt

iSCSI:

```yaml
oc create -f - <<EOF
apiVersion: cdi.kubevirt.io/v1beta1
kind: StorageProfile
metadata:
  name: synology-iscsi-storage
spec:
  claimPropertySets:
  - accessModes:
    - ReadWriteMany
    volumeMode:
      Block
  - accessModes:
    - ReadWriteOnce
    volumeMode:
      Block
  - accessModes:
    - ReadWriteOnce
    volumeMode:
      Filesystem
  cloneStrategy: csi-clone
  dataImportCronSourceFormat: pvc
EOF
```

NFS:

```yaml
oc create -f - <<EOF
apiVersion: cdi.kubevirt.io/v1beta1
kind: StorageProfile
metadata:
  name: synology-nfs-storage
spec:
  claimPropertySets:
  - accessModes:
    - ReadWriteMany
    volumeMode:
      Filesystem
  - accessModes:
    - ReadWriteOnce
    volumeMode:
      Filesystem
  cloneStrategy: csi-clone
  dataImportCronSourceFormat: pvc
EOF
```

## Authentication

The following got validated using:

```shell
ldapsearch -x -H ldap://jarvisnas.jarvis.lab \
-D "uid=root,cn=users,dc=ldap,dc=jarvis,dc=lab" \
-b "dc=ldap,dc=jarvis,dc=lab" \
-W "(objectClass=*)"
```

oAuth config for LDAP:

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
    - github:
        clientID: Ov23ligGyzfBsYoIhpAT
        clientSecret:
          name: github-client-secret-xgllr
        hostname: ''
        organizations:
          - rguske-labs
        teams: []
      mappingMethod: claim
      name: github
      type: GitHub
    - ldap:
        attributes:
          email:
            - mail
          id:
            - dn
          name:
            - cn
          preferredUsername:
            - uid
        bindDN: 'uid=root,cn=users,dc=ldap,dc=jarvis,dc=lab'
        bindPassword:
          name: ldap-bind-password-p6nj9
        insecure: true
        url: 'ldap://jarvisnas.jarvis.lab/cn=users,dc=ldap,dc=jarvis,dc=lab?uid'
      mappingMethod: claim
      name: ldap
      type: LDAP
```

## Customization

Login screen customized.

![login-screen](/assets/login-screen.png)