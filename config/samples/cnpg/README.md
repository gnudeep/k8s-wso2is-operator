# Deploying WSO2 Identity Server with CloudNativePG

This guide walks through deploying WSO2 Identity Server 7.3.0 with a PostgreSQL database managed by the [CloudNativePG](https://cloudnative-pg.io) operator on Kubernetes.

## Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                         │
│                                                                   │
│  ┌─────────────────┐   ┌──────────────────┐   ┌──────────────┐  │
│  │  WSO2 IS Operator│   │  CloudNativePG   │   │  NFS CSI     │  │
│  │  (wso2-iam-      │   │  Operator        │   │  Driver      │  │
│  │   system ns)     │   │  (cnpg-system)   │   │  (kube-system│  │
│  └────────┬─────────┘   └────────┬─────────┘   └──────┬───────┘  │
│           │ manages              │ manages             │ provides │
│           ▼                      ▼                     ▼          │
│  ┌─────────────────┐   ┌──────────────────┐   ┌──────────────┐  │
│  │  IS Pod 1        │   │  PostgreSQL      │   │  NFS Server  │  │
│  │  (node-0)        │──▶│  Cluster         │   │  Pod         │  │
│  ├─────────────────┤   │  (wso2is-pg-rw)  │   └──────┬───────┘  │
│  │  IS Pod 2        │──▶│                  │          │          │
│  │  (node-1)        │   │  ├ wso2_identity │   ┌──────┴───────┐  │
│  └────────┬─────────┘   │  └ wso2_shared   │   │  NFS PVC     │  │
│           │              └──────────────────┘   │  (RWX)       │  │
│           │                                     └──────┬───────┘  │
│           └── /userstores (shared) ────────────────────┘          │
│                                                                   │
│           ┌─────────────┐                                         │
│           │ wso2is-     │◀── load balances across IS pods         │
│           │ service     │                                         │
│           └─────────────┘                                         │
└───────────────────────────────────────────────────────────────────┘
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

> **Why `wso2is:7.2.0`?** The WSO2 IS 7.3.0 GA image is not yet published on Docker Hub (only a beta tag exists on GitHub). The Dockerfile uses `wso2/wso2is:7.2.0` (latest stable) as the base. When the 7.3.0 image is released, update the `FROM` line and the path from `wso2is-7.2.0` to `wso2is-7.3.0`.

If using k3d, import the image:

```bash
k3d image import wso2/wso2is:7.3.0-pg -c <cluster-name>
```

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

## Step 5: Set Up Shared Storage and Ingress

### Shared storage for secondary userstores

WSO2 IS stores secondary userstore configurations as XML files in the `/repository/deployment/server/userstores` directory. In a multi-replica deployment, all pods need read-write access to this directory so that userstores created on one node are visible to all others.

> **Note:** The primary user store (`database_unique_id`) stores all user data in PostgreSQL and does **not** require shared file storage. The shared PVC is only needed for secondary userstore XML configurations. See the [WSO2 docs on secondary user stores](https://github.com/wso2/docs-is/blob/master/en/identity-server/6.0.0/docs/deploy/configure-secondary-user-stores.md) for details.

**For production (cloud):** Use a `ReadWriteMany` StorageClass provided by your cloud (AWS EFS, Azure Files, GCP Filestore):

```bash
kubectl apply -f ../../artifacts/07-pvc.yaml
```

**For k3s / local development:** The default `local-path` StorageClass only supports `ReadWriteOnce`. Use the in-cluster NFS approach below.

#### Setting up NFS shared storage on k3s

Install the **NFS CSI Driver** (handles NFS mounting via CSI — no NFS client utilities needed on nodes):

```bash
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system \
  --set kubeletDir=/var/lib/kubelet
```

Deploy an **in-cluster NFS server** pod:

```bash
kubectl apply -f - <<'EOF'
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-server-data
  namespace: kube-system
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: local-path
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-server
  template:
    metadata:
      labels:
        app: nfs-server
    spec:
      containers:
        - name: nfs-server
          image: itsthenetwork/nfs-server-alpine:12
          ports:
            - containerPort: 2049
          securityContext:
            privileged: true
          env:
            - name: SHARED_DIRECTORY
              value: /data
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: nfs-server-data
---
apiVersion: v1
kind: Service
metadata:
  name: nfs-server
  namespace: kube-system
spec:
  selector:
    app: nfs-server
  ports:
    - port: 2049
      targetPort: 2049
EOF
```

Create the **NFS StorageClass** and **ReadWriteMany PVC**:

```bash
kubectl apply -f - <<'EOF'
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.kube-system.svc.cluster.local
  share: /
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - nfsvers=4.1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: user-store-pv-claim
spec:
  storageClassName: nfs-csi
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF
```

Verify the PVC is bound with `RWX` access:

```bash
kubectl get pvc user-store-pv-claim
```

```
NAME                  STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS
user-store-pv-claim   Bound    pvc-xxxxx    1Gi        RWX            nfs-csi
```

### Ingress

Create an Ingress for external access:

```bash
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

## Multi-Node Deployment

The `wso2is-with-postgres.yaml` sample deploys 2 IS replicas with Kubernetes clustering enabled. This section explains the key considerations.

### How clustering works

When `replicas > 1`, the IS nodes must discover each other for session replication. The `tomlConfig` includes:

```toml
[clustering]
membership_scheme = "kubernetes"
domain = "wso2.is.domain"
[clustering.properties]
membershipSchemeClassName = "org.wso2.carbon.membership.scheme.kubernetes.KubernetesMembershipScheme"
KUBERNETES_NAMESPACE = "default"
KUBERNETES_SERVICES = "wso2is-service"
KUBERNETES_MASTER_SKIP_SSL_VERIFICATION = true
```

The IS pods use the `wso2is-service` endpoints to discover cluster members. The `wso2is-service` Service load-balances traffic across all ready pods.

### Shared storage requirements

| Component | Storage | Shared across pods? |
|-----------|---------|:---:|
| Primary user store (`database_unique_id`) | PostgreSQL tables | Yes (via DB) |
| Secondary userstores (added via console/API) | XML files in `/userstores` dir | Requires `ReadWriteMany` PVC |
| IS runtime files (e.g. `AGENT.xml`) | `/userstores` dir | Requires `ReadWriteMany` PVC |
| Configuration (`deployment.toml`) | ConfigMap | Yes (mounted in all pods) |
| Keystores | Secret | Yes (mounted in all pods) |

The operator sets `fsGroup: 802` on the pod security context so that NFS and CSI-backed volumes are writable by the `wso2carbon` user (uid 802).

### Shared storage options

| Option | Access Mode | Persists | k3s Local | Cloud |
|--------|:-----------:|:--------:|:---------:|:-----:|
| NFS CSI (in-cluster NFS server) | ReadWriteMany | Yes | Recommended | - |
| AWS EFS / Azure Files / GCP Filestore | ReadWriteMany | Yes | - | Recommended |
| Longhorn | ReadWriteMany | Yes | Heavy but works | - |
| `emptyDir` | Per-pod only | No | Quick testing only | - |

For k3s local, the [NFS CSI setup in Step 5](#setting-up-nfs-shared-storage-on-k3s) provides ReadWriteMany without requiring NFS client utilities on the k3d/k3s nodes.

### Validating shared storage

Write a file from one pod and read it from another:

```bash
POD1=$(kubectl get pods -l deployment=identity-server -o jsonpath='{.items[0].metadata.name}')
POD2=$(kubectl get pods -l deployment=identity-server -o jsonpath='{.items[1].metadata.name}')

# Write from pod 1
kubectl exec $POD1 -- sh -c 'echo "test" > /home/wso2carbon/wso2is-7.3.0/repository/deployment/server/userstores/test.txt'

# Read from pod 2
kubectl exec $POD2 -- cat /home/wso2carbon/wso2is-7.3.0/repository/deployment/server/userstores/test.txt

# Clean up
kubectl exec $POD1 -- rm /home/wso2carbon/wso2is-7.3.0/repository/deployment/server/userstores/test.txt
```

### Scaling replicas

Change the `replicas` field in the Wso2Is CR:

```bash
kubectl patch wso2is identity-server --type merge -p '{"spec":{"replicas":3}}'
```

The operator scales the deployment and the clustering config ensures all new nodes join automatically via the `wso2is-service` endpoint discovery.

### Monitoring deployment health

The operator reports real-time status conditions:

```bash
kubectl get wso2is -o wide
```

```
NAME              READY   DESIRED   SERVICE          HOST             AVAILABLE   STATUS
identity-server   2       2         wso2is-service   identityserver   True        All components are ready
```

Detailed conditions:

```bash
kubectl get wso2is identity-server -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.message}{"\n"}{end}'
```

```
ConfigReady      True    ConfigMap identity-server-conf is available
ServiceReady     True    Service wso2is-service is available with ClusterIP 10.43.x.x
DeploymentReady  True    2/2 replicas are ready
PodsReady        True    2/2 pods are ready
Available        True    All components are ready
```

## Production Considerations

### PostgreSQL High Availability

For production, increase the CloudNativePG replica count:

```yaml
spec:
  instances: 3  # 1 primary + 2 replicas
```

### Persistent Storage

Configure a production-grade StorageClass for PostgreSQL:

```yaml
spec:
  storage:
    size: 20Gi
    storageClass: gp3  # AWS EBS
```

For the IS userstore PVC, use a cloud-native ReadWriteMany StorageClass:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: user-store-pv-claim
spec:
  storageClassName: efs-sc    # AWS EFS
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
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

## File Reference

| File | Description |
|------|-------------|
| `Dockerfile.wso2is-pg` | Custom IS image with PostgreSQL JDBC driver |
| `postgres-cluster.yaml` | CloudNativePG Cluster CR and credentials Secret |
| `postgres-schemas.yaml` | ConfigMaps containing WSO2 IS database schemas |
| `postgres-schema-job.yaml` | Kubernetes Job to initialize database schemas |
| `wso2is-with-postgres.yaml` | Wso2Is CR configured for PostgreSQL |
