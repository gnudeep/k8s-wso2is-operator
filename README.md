## WSO2 Identity Server 7.3.0 - Kubernetes Operator

A Kubernetes CRD operator for deploying and managing WSO2 Identity Server 7.3.0 on Kubernetes clusters.

If you prefer Helm-based deployment, refer to: [https://github.com/wso2/kubernetes-is](https://github.com/wso2/kubernetes-is)

### Key Benefits

- Auto healing and self-recovery
- Single and multi-replica deployments
- Ability to provision multiple IS instances on the same cluster
- Custom keystore mounting (PKCS12)
- Custom `deployment.toml` configuration support
- Rolling update strategy with zero downtime
- Kubernetes-native clustering support

## Prerequisites

- Kubernetes v1.22+ cluster
- `kubectl` configured with cluster-admin privileges
- A PersistentVolume with `ReadWriteMany` (production) or `ReadWriteOnce` (single-node test) access mode
- An Ingress controller (e.g., NGINX) for external access

## Quick Start (Operator Deployment)

### Step 1: Deploy the operator

Apply the all-in-one manifest to install the CRDs, RBAC, and controller:

```bash
kubectl apply -f https://raw.githubusercontent.com/wso2/k8s-wso2is-operator/is-7.3.0/artifacts/operator.yaml
```

Or apply the individual manifests:

```bash
kubectl apply -f artifacts/00-namespace.yaml
kubectl apply -f artifacts/01-cluster-role.yaml
kubectl apply -f artifacts/02-service-account.yaml
kubectl apply -f artifacts/03-cluster-role-binding.yaml
kubectl apply -f artifacts/05-crd-iam.wso2.com_wso2is.yaml
kubectl apply -f artifacts/06-controller.yaml
```

### Step 2: Verify the operator is running

```bash
kubectl get pods -n wso2-iam-system
```

You should see:

```
NAME                          READY   STATUS    RESTARTS   AGE
controller-xxxxx-xxxxx        1/1     Running   0          30s
```

### Step 3: Create supporting resources

Create a PersistentVolumeClaim for user store storage:

```bash
kubectl apply -f artifacts/07-pvc.yaml
```

Create an Ingress for external access:

```bash
kubectl apply -f artifacts/08-ingress.yaml
```

### Step 4: Deploy WSO2 Identity Server

**Single-node test cluster:**

```bash
kubectl apply -f config/samples/single_node_test_cluster.yaml
```

This creates a minimal single-replica IS deployment:

```yaml
apiVersion: iam.wso2.com/v1beta1
kind: Wso2Is
metadata:
  name: identity-server-test
spec:
  replicas: 1
  version: "7.3.0"
  configurations:
    host: identityserver
    serviceType: ClusterIP
```

**Multi-node production cluster with MySQL:**

```bash
kubectl apply -f config/samples/standard_multi_node.yaml
```

### Step 5: Verify the deployment

```bash
# Check the Wso2Is custom resource status
kubectl get wso2is -o wide

# Check the IS pod
kubectl get pods -l deployment=identity-server-test

# Check the service
kubectl get svc wso2is-service
```

WSO2 IS takes approximately 30-60 seconds to start. The readiness probe has a 250-second initial delay.

### Step 6: Access the deployment

**Add a host entry** so your browser can resolve the configured hostname. WSO2 IS validates the hostname in incoming requests against its `server.hostname` configuration, so accessing via `localhost` may cause redirect issues in the console.

Add to `/etc/hosts` (Linux/macOS) or `C:\Windows\System32\drivers\etc\hosts` (Windows):

```
127.0.0.1   identityserver
```

> Replace `identityserver` with the value of `configurations.host` in your Wso2Is CR if you changed it.

**Start port-forwarding:**

```bash
kubectl port-forward svc/wso2is-service 9443:9443
```

**Open in your browser:**

| URL | Description |
|-----|-------------|
| `https://identityserver:9443/console` | IS 7.x Console (new UI) |
| `https://identityserver:9443/carbon` | Carbon Management Console (legacy) |

Default login credentials: `admin` / `admin`

> Your browser will show a certificate warning (self-signed cert) — accept it to proceed.

## Configuration Reference

### Wso2Is Spec

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `replicas` | int | - | Number of IS pod replicas |
| `version` | string | `7.3.0` | WSO2 IS version |
| `configurations.host` | string | - | Hostname for the IS instance |
| `configurations.serviceType` | string | `NodePort` | Kubernetes Service type (`NodePort`, `ClusterIP`, `LoadBalancer`) |
| `configurations.superAdmin.username` | string | `admin` | Admin username |
| `configurations.superAdmin.password` | string | `admin` | Admin password |
| `configurations.userStore.type` | string | `database_unique_id` | User store type |
| `configurations.database.identityDb` | object | H2 embedded | Identity database connection |
| `configurations.database.sharedDb` | object | H2 embedded | Shared database connection |
| `configurations.keystore.primary.name` | string | `wso2carbon.p12` | Primary keystore file |
| `configurations.clustering` | object | Kubernetes scheme | Clustering configuration |
| `tomlConfig` | string | - | Custom `deployment.toml` content (overrides all other config) |
| `keystoreMounts` | list | - | Custom keystores to mount |

### Custom TOML Configuration

You can provide a full custom `deployment.toml` using the `tomlConfig` field:

```yaml
apiVersion: iam.wso2.com/v1beta1
kind: Wso2Is
metadata:
  name: identity-server
spec:
  replicas: 1
  configurations:
    host: identityserver
  tomlConfig: |
    [server]
    hostname = "identityserver"

    [super_admin]
    username = "admin"
    password = "admin"

    [user_store]
    type = "database_unique_id"

    [database.identity_db]
    type = "mysql"
    url = "jdbc:mysql://mysql:3306/IS_IDENTITY_DB"
    username = "dbuser"
    password = "dbpass"
    driver = "com.mysql.cj.jdbc.Driver"
```

### Custom Keystores

Mount custom keystores using the `keystoreMounts` field:

```yaml
spec:
  keystoreMounts:
    - name: custom-keystore.p12
      data: <base64-encoded-keystore-data>
```

## External Database Setup

For production deployments, configure external MySQL databases. Refer to the WSO2 documentation:

- https://is.docs.wso2.com/en/7.3.0/setup/changing-to-mysql/
- https://is.docs.wso2.com/en/7.3.0/setup/changing-datasource-bpsds/
- https://is.docs.wso2.com/en/7.3.0/setup/changing-datasource-consent-management/

### Required Databases

- `WSO2_IDENTITY_DB`
- `WSO2_SHARED_DB`
- `WSO2_CONSENT_DB` (Optional)
- `WSO2_BPS_DB` (Optional)

## Development

### Prerequisites

- [Go](https://golang.org/) 1.13+
- [Operator SDK](https://sdk.operatorframework.io/) (`brew install operator-sdk`)
- Access to a Kubernetes cluster
- Docker for building container images

### Local Development

```bash
# Clone the repository
git clone https://github.com/wso2/k8s-wso2is-operator.git
cd k8s-wso2is-operator

# Install CRDs
kubectl apply -f config/crd/bases/iam.wso2.com_wso2is.yaml

# Run the operator locally
make run

# In another terminal, apply a sample config
kubectl apply -f config/samples/single_node_test_cluster.yaml
```

### Building the Operator Image

```bash
# Build
make docker-build IMG=wso2/wso2-iam-operator:7.3.0

# Push
make docker-push IMG=wso2/wso2-iam-operator:7.3.0
```

## System Architecture

![System Architecture](https://user-images.githubusercontent.com/3047253/105663226-b9149900-5ef7-11eb-825b-0413649a99ed.jpg)

## Sample Configurations

See the [config/samples](config/samples/) directory:

| File | Description |
|------|-------------|
| `single_node_test_cluster.yaml` | Minimal single-replica test deployment |
| `standard_multi_node.yaml` | Multi-replica production deployment with MySQL |
| `standard_multi_factor.yaml` | Multi-replica with TOTP authentication enabled |
