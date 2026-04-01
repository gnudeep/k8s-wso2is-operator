# Deploying WSO2 Identity Server with CloudNativePG

This guide walks through deploying WSO2 Identity Server 7.3.0 with a PostgreSQL database managed by the [CloudNativePG](https://cloudnative-pg.io) operator on Kubernetes.

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                  │
│                                                       │
│  ┌─────────────────┐        ┌──────────────────────┐ │
│  │  WSO2 IS Operator│        │  CloudNativePG       │ │
│  │  (wso2-iam-      │        │  Operator            │ │
│  │   system ns)     │        │  (cnpg-system ns)    │ │
│  └────────┬─────────┘        └──────────┬───────────┘ │
│           │ manages                      │ manages     │
│           ▼                              ▼             │
│  ┌─────────────────┐        ┌──────────────────────┐ │
│  │  WSO2 Identity   │──────▶│  PostgreSQL Cluster   │ │
│  │  Server Pod      │ JDBC   │  (wso2is-pg-rw:5432) │ │
│  │  (port 9443)     │        │                      │ │
│  └─────────────────┘        │  ├─ wso2_identity_db  │ │
│                              │  └─ wso2_shared_db   │ │
│                              └──────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

## Prerequisites

- Kubernetes v1.22+ cluster
- `kubectl` configured with cluster-admin access
- Docker (for building the custom IS image with PostgreSQL driver)
- WSO2 IS operator already deployed ([see main README](../../../README.md))

## Step 1: Install CloudNativePG Operator

```bash
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.28/releases/cnpg-1.28.1.yaml
```

Verify the operator is running:

```bash
kubectl get deployment -n cnpg-system cnpg-controller-manager
```

## Step 2: Build the WSO2 IS Image with PostgreSQL Driver

The default WSO2 IS image does not include the PostgreSQL JDBC driver. Build a custom image that adds it:

```bash
docker build -t wso2/wso2is:7.3.0-pg -f Dockerfile.wso2is-pg .
```

The `Dockerfile.wso2is-pg` adds the PostgreSQL JDBC driver:

```dockerfile
FROM wso2/wso2is:7.2.0

USER root
RUN wget -q -O /home/wso2carbon/wso2is-7.2.0/repository/components/lib/postgresql-42.7.3.jar \
    https://jdbc.postgresql.org/download/postgresql-42.7.3.jar && \
    chown wso2carbon:wso2 /home/wso2carbon/wso2is-7.2.0/repository/components/lib/postgresql-42.7.3.jar
USER wso2carbon
```

If using k3d, import the image:

```bash
k3d image import wso2/wso2is:7.3.0-pg -c <cluster-name>
```

> **Note:** When the official WSO2 IS 7.3.0 image is released, update the base image and paths accordingly.

## Step 3: Deploy PostgreSQL Cluster

Create the PostgreSQL credentials and cluster:

```bash
kubectl apply -f postgres-cluster.yaml
```

This creates:
- A `Secret` with PostgreSQL credentials (`wso2carbon`/`wso2carbon`)
- A CloudNativePG `Cluster` with two databases: `wso2_identity_db` and `wso2_shared_db`

Wait for the cluster to be healthy:

```bash
kubectl get cluster wso2is-pg -w
```

Expected output:

```
NAME        INSTANCES   READY   STATUS                     PRIMARY
wso2is-pg   1           1       Cluster in healthy state   wso2is-pg-1
```

Verify the services are available:

```bash
kubectl get svc -l cnpg.io/cluster=wso2is-pg
```

```
NAME           TYPE        CLUSTER-IP      PORT(S)
wso2is-pg-r    ClusterIP   10.43.x.x      5432/TCP
wso2is-pg-ro   ClusterIP   10.43.x.x      5432/TCP
wso2is-pg-rw   ClusterIP   10.43.x.x      5432/TCP
```

The `wso2is-pg-rw` service is the read-write endpoint that WSO2 IS will connect to.

## Step 4: Initialize Database Schemas

WSO2 IS requires pre-created database schemas for PostgreSQL. Apply the schema ConfigMaps and run the initialization Job:

```bash
kubectl apply -f postgres-schemas.yaml
kubectl apply -f postgres-schema-job.yaml
```

Wait for the Job to complete:

```bash
kubectl wait --for=condition=complete job/wso2is-pg-schema-init --timeout=120s
```

Verify schemas were created:

```bash
kubectl exec wso2is-pg-1 -c postgres -- psql -U postgres -d wso2_shared_db \
  -c "SELECT count(*) as tables FROM information_schema.tables WHERE table_schema='public';"

kubectl exec wso2is-pg-1 -c postgres -- psql -U postgres -d wso2_identity_db \
  -c "SELECT count(*) as tables FROM information_schema.tables WHERE table_schema='public';"
```

Expected: ~60 tables in shared_db, ~164 tables in identity_db.

## Step 5: Create PVC and Ingress

```bash
# PVC for user store storage (use ReadWriteOnce for single-node, ReadWriteMany for multi-node)
kubectl apply -f ../../artifacts/07-pvc.yaml

# Ingress for external access
kubectl apply -f ../../artifacts/08-ingress.yaml
```

## Step 6: Deploy WSO2 Identity Server

Before deploying, update the container image in the operator's `constants.go` to `wso2/wso2is:7.3.0-pg` and rebuild the operator, or use the `tomlConfig` approach which works with any IS image that has the PostgreSQL driver.

```bash
kubectl apply -f wso2is-with-postgres.yaml
```

This creates a `Wso2Is` CR with a custom `tomlConfig` that configures:
- PostgreSQL as the identity and shared database
- Database connection via the `wso2is-pg-rw` ClusterIP service
- PKCS12 keystores (IS 7.x default)

Monitor the startup:

```bash
# Watch pod status
kubectl get pods -l deployment=identity-server -w

# Check startup logs
kubectl logs -l deployment=identity-server -f
```

WSO2 IS takes approximately 30-40 seconds to start. Look for:

```
WSO2 Carbon started in XX sec
```

## Step 7: Access the Deployment

Add a host entry to `/etc/hosts`:

```
127.0.0.1   identityserver
```

Start port-forwarding:

```bash
kubectl port-forward svc/wso2is-service 9443:9443
```

Open in your browser:

| URL | Description |
|-----|-------------|
| `https://identityserver:9443/console` | IS 7.x Console (new UI) |
| `https://identityserver:9443/carbon` | Carbon Management Console |

Login: `admin` / `admin`

## Verification

### Verify IS is using PostgreSQL

Check that IS has written data to the PostgreSQL databases:

```bash
# Check user data in shared_db
kubectl exec wso2is-pg-1 -c postgres -- psql -U postgres -d wso2_shared_db \
  -c "SELECT count(*) FROM um_user;"

# Check OAuth apps in identity_db
kubectl exec wso2is-pg-1 -c postgres -- psql -U postgres -d wso2_identity_db \
  -c "SELECT count(*) FROM idn_oauth_consumer_apps;"
```

### Verify all components are healthy

```bash
kubectl get wso2is -o wide
kubectl get cluster wso2is-pg
kubectl get pods
```

## Production Considerations

### PostgreSQL High Availability

For production, increase the CloudNativePG replica count:

```yaml
spec:
  instances: 3  # 1 primary + 2 replicas
```

### Persistent Storage

Configure a production-grade StorageClass:

```yaml
spec:
  storage:
    size: 20Gi
    storageClass: gp3  # AWS EBS
```

### Database Credentials

Use a proper secret management solution. Avoid hardcoding passwords:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: wso2is-pg-credentials
type: kubernetes.io/basic-auth
stringData:
  username: wso2carbon
  password: <generate-a-strong-password>
```

### WSO2 IS Replicas

Scale IS for high availability:

```yaml
spec:
  replicas: 2
  configurations:
    clustering:
      membership_scheme: kubernetes
      properties:
        KUBERNETES_SERVICES: wso2is-service
```

## File Reference

| File | Description |
|------|-------------|
| `Dockerfile.wso2is-pg` | Custom IS image with PostgreSQL JDBC driver |
| `postgres-cluster.yaml` | CloudNativePG Cluster CR and credentials Secret |
| `postgres-schemas.yaml` | ConfigMaps containing WSO2 IS database schemas |
| `postgres-schema-job.yaml` | Kubernetes Job to initialize database schemas |
| `wso2is-with-postgres.yaml` | Wso2Is CR configured for PostgreSQL |
