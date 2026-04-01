# WSO2 Identity Server Kubernetes Operator — AI Agent Skills

This file provides structured instructions for AI agents to deploy and manage WSO2 Identity Server 7.3.0 on Kubernetes using this operator.

---

## Skill: deploy-operator

**Description:** Deploy the WSO2 IS operator to a Kubernetes cluster.

**Prerequisites:** A running Kubernetes v1.22+ cluster with `kubectl` configured.

**Steps:**

```bash
# Apply all operator resources (namespace, RBAC, CRDs, controller)
kubectl apply -f artifacts/00-namespace.yaml
kubectl apply -f artifacts/01-cluster-role.yaml
kubectl apply -f artifacts/02-service-account.yaml
kubectl apply -f artifacts/03-cluster-role-binding.yaml
kubectl apply -f artifacts/05-crd-iam.wso2.com_wso2is.yaml
kubectl apply -f artifacts/06-controller.yaml

# Wait for controller to be ready
kubectl wait --for=condition=ready pod -l app=controller -n wso2-iam-system --timeout=120s
```

**Validation:**

```bash
kubectl get pods -n wso2-iam-system
# Expect: controller pod 1/1 Running
```

---

## Skill: deploy-is-standalone

**Description:** Deploy a single-node WSO2 IS instance with embedded H2 database (for quick testing).

**Prerequisites:** Operator deployed (`deploy-operator`), PVC and Ingress created.

**Steps:**

```bash
# Create PVC (use ReadWriteOnce for single-node)
kubectl apply -f artifacts/07-pvc.yaml

# Create Ingress
kubectl apply -f artifacts/08-ingress.yaml

# Deploy IS
kubectl apply -f config/samples/single_node_test_cluster.yaml

# Wait for IS to start (250s readiness probe delay)
kubectl wait --for=condition=ready pod -l deployment=identity-server-test --timeout=400s
```

**Validation:**

```bash
kubectl get wso2is -o wide
# Expect: READY=1, AVAILABLE=True
```

---

## Skill: deploy-is-with-postgres

**Description:** Deploy WSO2 IS with PostgreSQL managed by CloudNativePG. This is the recommended production-like deployment pattern.

**Prerequisites:** Operator deployed (`deploy-operator`), Docker available for building custom IS image.

**Steps:**

### 1. Install CloudNativePG operator

```bash
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.28/releases/cnpg-1.28.1.yaml
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=cloudnative-pg -n cnpg-system --timeout=120s
```

### 2. Build IS image with PostgreSQL driver

The default WSO2 IS image does not include the PostgreSQL JDBC driver.

```bash
docker build -t wso2/wso2is:7.3.0-pg -f config/samples/cnpg/Dockerfile.wso2is-pg config/samples/cnpg/
```

If using k3d, import the image into all nodes:

```bash
docker save wso2/wso2is:7.3.0-pg -o /tmp/wso2is-pg.tar
for node in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  docker exec -i $node ctr images import - < /tmp/wso2is-pg.tar
done
```

**Important:** The operator's container image constant (`controllers/constants.go`) must reference `wso2/wso2is:7.3.0-pg`. Rebuild the operator image if changed.

### 3. Deploy PostgreSQL cluster

```bash
kubectl apply -f config/samples/cnpg/postgres-cluster.yaml
kubectl wait --for=condition=ready pod -l cnpg.io/cluster=wso2is-pg --timeout=180s

# Verify cluster is healthy
kubectl get cluster wso2is-pg
# Expect: STATUS = "Cluster in healthy state"
```

### 4. Initialize database schemas

```bash
kubectl apply -f config/samples/cnpg/postgres-schemas.yaml
kubectl apply -f config/samples/cnpg/postgres-schema-job.yaml
kubectl wait --for=condition=complete job/wso2is-pg-schema-init --timeout=120s
```

Verify:

```bash
kubectl exec wso2is-pg-1 -c postgres -- psql -U postgres -d wso2_shared_db \
  -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
# Expect: ~60

kubectl exec wso2is-pg-1 -c postgres -- psql -U postgres -d wso2_identity_db \
  -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
# Expect: ~164
```

### 5. Create shared storage PVC

**For cloud (AWS/Azure/GCP):**

```bash
kubectl apply -f artifacts/07-pvc.yaml
```

**For k3s / local development (NFS CSI):**

```bash
# Install NFS CSI driver
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --namespace kube-system --set kubeletDir=/var/lib/kubelet
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=csi-driver-nfs -n kube-system --timeout=60s

# Deploy in-cluster NFS server
kubectl apply -f - <<'EOF'
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
kubectl wait --for=condition=ready pod -l app=nfs-server -n kube-system --timeout=120s

# Create NFS StorageClass and RWX PVC
kubectl apply -f - <<'EOF'
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
  accessModes: [ReadWriteMany]
  resources:
    requests:
      storage: 1Gi
EOF

# Verify
kubectl get pvc user-store-pv-claim
# Expect: STATUS=Bound, ACCESS MODES=RWX
```

### 6. Create Ingress and deploy IS

```bash
kubectl apply -f artifacts/08-ingress.yaml
kubectl apply -f config/samples/cnpg/wso2is-with-postgres.yaml

# Wait for all IS pods (250s readiness delay)
kubectl wait --for=condition=ready pod -l deployment=identity-server --timeout=400s
```

**Validation:**

```bash
kubectl get wso2is -o wide
# Expect: READY=2, DESIRED=2, AVAILABLE=True, STATUS="All components are ready"

kubectl get wso2is identity-server -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.message}{"\n"}{end}'
# Expect: all conditions True
```

---

## Skill: access-is

**Description:** Configure local access to the WSO2 IS console.

**Steps:**

1. Add host entry:

```bash
# Linux/macOS
echo "127.0.0.1   identityserver" | sudo tee -a /etc/hosts

# Windows (PowerShell as Admin)
Add-Content C:\Windows\System32\drivers\etc\hosts "127.0.0.1   identityserver"
```

2. Start port-forward:

```bash
kubectl port-forward svc/wso2is-service 9443:9443
```

3. Open browser:
   - Console: `https://identityserver:9443/console`
   - Carbon: `https://identityserver:9443/carbon`
   - Credentials: `admin` / `admin`
   - Accept the self-signed certificate warning.

---

## Skill: scale-is

**Description:** Scale WSO2 IS replicas up or down.

**Steps:**

```bash
# Scale to N replicas
kubectl patch wso2is identity-server --type merge -p '{"spec":{"replicas":N}}'

# Wait for all replicas
kubectl wait --for=condition=ready pod -l deployment=identity-server --timeout=400s

# Verify
kubectl get wso2is -o wide
```

**Notes:**
- Multi-replica requires Kubernetes clustering config in `tomlConfig` (already configured in `wso2is-with-postgres.yaml`).
- Multi-replica requires a `ReadWriteMany` PVC for the userstores directory.

---

## Skill: validate-deployment

**Description:** Check the health of a WSO2 IS deployment.

**Steps:**

```bash
# Quick status
kubectl get wso2is -o wide

# Detailed conditions
kubectl get wso2is identity-server -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\t"}{.message}{"\n"}{end}'

# Pod health
kubectl get pods -l deployment=identity-server -o wide

# Operator logs
kubectl logs -n wso2-iam-system deployment/controller --tail=20

# IS logs
kubectl logs -l deployment=identity-server --tail=20

# PostgreSQL status (if using CNPG)
kubectl get cluster wso2is-pg
```

**Expected healthy state:**
- `kubectl get wso2is`: AVAILABLE=True
- All conditions: status=True
- All pods: READY=1/1, STATUS=Running
- Operator logs: "Successfully Reconciled"
- PostgreSQL: STATUS="Cluster in healthy state"

---

## Skill: teardown

**Description:** Remove all WSO2 IS and related resources from the cluster.

**Steps:**

```bash
# Delete IS deployment
kubectl delete wso2is --all

# Delete PostgreSQL cluster (if using CNPG)
kubectl delete cluster wso2is-pg
kubectl delete secret wso2is-pg-credentials
kubectl delete configmap wso2is-pg-schema-shared wso2is-pg-schema-identity
kubectl delete job wso2is-pg-schema-init

# Delete operator
kubectl delete -f artifacts/06-controller.yaml
kubectl delete -f artifacts/05-crd-iam.wso2.com_wso2is.yaml
kubectl delete -f artifacts/03-cluster-role-binding.yaml
kubectl delete -f artifacts/02-service-account.yaml
kubectl delete -f artifacts/01-cluster-role.yaml
kubectl delete -f artifacts/00-namespace.yaml

# Delete shared storage (if NFS CSI was set up)
kubectl delete pvc user-store-pv-claim
kubectl delete sc nfs-csi
kubectl delete deployment nfs-server -n kube-system
kubectl delete svc nfs-server -n kube-system
kubectl delete pvc nfs-server-data -n kube-system
helm uninstall csi-driver-nfs -n kube-system

# Delete CloudNativePG operator (if installed)
kubectl delete -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.28/releases/cnpg-1.28.1.yaml

# Delete Ingress and PVC
kubectl delete ingress wso2is-ingress
kubectl delete pvc user-store-pv-claim
```

---

## Quick Reference

| Resource | Command |
|----------|---------|
| Operator status | `kubectl get pods -n wso2-iam-system` |
| IS status | `kubectl get wso2is -o wide` |
| IS conditions | `kubectl describe wso2is identity-server` |
| IS logs | `kubectl logs -l deployment=identity-server -f` |
| PostgreSQL status | `kubectl get cluster wso2is-pg` |
| Scale IS | `kubectl patch wso2is identity-server --type merge -p '{"spec":{"replicas":N}}'` |
| Port forward | `kubectl port-forward svc/wso2is-service 9443:9443` |

## Key Files

| Path | Description |
|------|-------------|
| `artifacts/operator.yaml` | All-in-one operator deployment manifest |
| `config/samples/single_node_test_cluster.yaml` | Minimal single-node IS (H2 database) |
| `config/samples/cnpg/postgres-cluster.yaml` | CloudNativePG cluster + credentials |
| `config/samples/cnpg/postgres-schemas.yaml` | Database schema ConfigMaps |
| `config/samples/cnpg/postgres-schema-job.yaml` | Schema initialization Job |
| `config/samples/cnpg/wso2is-with-postgres.yaml` | Multi-replica IS with PostgreSQL + clustering |
| `config/samples/cnpg/Dockerfile.wso2is-pg` | Custom IS image with PostgreSQL driver |
| `controllers/constants.go` | Container image and resource name constants |
