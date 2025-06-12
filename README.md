# Documentation of my Homelab Jarvis-Lab

![cover](/assets/cover-jarvis-lab-ironman.png)

- [Documentation of my Homelab Jarvis-Lab](#documentation-of-my-homelab-jarvis-lab)
  - [Networking](#networking)
  - [Storage](#storage)
  - [Customization](#customization)

## Networking



## Storage

[Installing the Local Storage Operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/storage/configuring-persistent-storage#local-storage-install_persistent-storage-local)

[Automating discovery and provisioning for local storage devices](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/storage/configuring-persistent-storage#local-storage-discovery_persistent-storage-local)

```yaml
apiVersion: local.storage.openshift.io/v1alpha1
kind: LocalVolumeSet
metadata:
  name: local-nvme-disk
  namespace: openshift-local-storage
spec:
  deviceInclusionSpec:
    deviceMechanicalProperties:
      - NonRotational
    deviceTypes:
      - disk
      - part
    maxSize: 250Gi
    minSize: 1Gi
#    models:
#      - Samsung
#      - SSD 980 250GB
  nodeSelector:
    nodeSelectorTerms:
      - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
              - ocp-mk1.jarvis.lab
  storageClassName: local-nvme-disk
  volumeMode: Block
```

![local-disks](/assets/node-disks.png)

## Customization

Login screen customized.

![login-screen](/assets/login-screen.png)