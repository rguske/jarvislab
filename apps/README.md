# Jarvislab Retroplay Online Applications

```code
tree
.
├── assets
│   ├── retroplay.jpg
│   └── retroplay.png
├── pacman
│   ├── pacman-deployment.yaml
│   ├── pacman-sa-role-rb.yaml
│   └── README.md
├── postgres
│   ├── base
│   │   ├── kustomization.yaml
│   │   ├── persistentvolumeclaims.yaml
│   │   ├── services.yaml
│   │   └── statefulset.yaml
│   ├── overlays
│   └── README.md
└── README.md
```

Deployed via GitOps.

Per default, OpenShift GitOps RBAC is configured like:

```yaml
[...]
spec:
  rbac:
    defaultPolicy: ""
    policy: |
      g, system:cluster-admins, role:admin
      g, cluster-admins, role:admin
    scopes: '[groups]'
[...]
```

Creation of a cluster-admin group will help to create GitOps Application:

```code
oc adm groups new cluster-admins
oc adm groups add-users cluster-admins $(oc whoami)
oc adm policy add-cluster-role-to-group cluster-admin cluster-admins
```
