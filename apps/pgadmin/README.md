# Running PGAdmin as a Kubernetes Deployment or Knative Service

Create a secret which will be used for the PGAdmin instance:

```yaml
oc create -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: pgadmin-credentials
type: Opaque
stringData:
  PGADMIN_SETUP_EMAIL: ""
  PGADMIN_SETUP_PASSWORD: ""
  PGADMIN_DEFAULT_EMAIL: ""
  PGADMIN_DEFAULT_PASSWORD: ""
EOF
```

