# PostgresDB Deployment Jarvislab Retroplay Online

Create a `secret` which contains the PostgresDB password first:

Example:

```code
oc create secret generic postgresql-secret --from-literal=POSTGRES_PASSWORD=redhat
```

```code
echo 'redhat' | base64`
```

```yaml
oc create -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: postgresql-secret
  labels:
    app: postgres
type: Opaque
data:
  POSTGRES_PASSWORD: ''
EOF
```