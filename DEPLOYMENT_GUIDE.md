
## 🎯 Overview

The s390x Llama Stack Distribution uses **PostgreSQL for storage** and requires environment variables to be set for database configuration.

## 📋 Prerequisites

1. **Llama Stack Operator** - Must be installed on your Kubernetes/OpenShift cluster
2. **PostgreSQL Database** - Must be running and accessible
3. **vLLM Server** - Must be running for inference
4. **Environment Variables** - Must be exported before starting the container (for standalone deployment)

## 🔧 Required Environment Variables

### PostgreSQL Configuration (MANDATORY)

Before running the container, you **MUST** export these environment variables:

```bash
# PostgreSQL connection details
export POSTGRES_HOST="your-postgres-host"        # e.g., localhost or postgres.namespace.svc.cluster.local
export POSTGRES_PORT="5432"                      # PostgreSQL port (default: 5432)
export POSTGRES_DB="llamastack"                  # Database name
export POSTGRES_USER="llamastack"                # Database user
export POSTGRES_PASSWORD="your-password"         # Database password

# vLLM configuration
export VLLM_URL="http://your-vllm-server:8000"  # vLLM server URL

# Model configuration (optional, has default)
export INFERENCE_MODEL="TinyLlama/TinyLlama-1.1B-Chat-v1.0"

# Server port (optional, default: 8321)
export PORT="8321"
```

### Why Environment Variables?

The `config.yaml` uses environment variable substitution:

```yaml
storage:
  backends:
    kv_default:
      type: kv_postgres
      host: ${POSTGRES_HOST:localhost}      # Uses POSTGRES_HOST env var
      port: ${POSTGRES_PORT:5432}           # Uses POSTGRES_PORT env var
      db: ${POSTGRES_DB:llamastack}         # Uses POSTGRES_DB env var
      user: ${POSTGRES_USER:llamastack}     # Uses POSTGRES_USER env var
      password: ${POSTGRES_PASSWORD:llamastack}  # Uses POSTGRES_PASSWORD env var
```

**Format**: `${ENV_VAR:default_value}`
- If `ENV_VAR` is set, it uses that value
- If `ENV_VAR` is not set, it uses `default_value`

## 🚀 Deployment Options

### Option 1: Standalone Deployment (Docker/Podman)

#### Step 1: Start PostgreSQL

```bash
# Using Podman/Docker
podman run -d \
  --name postgres \
  -e POSTGRES_USER=llamastack \
  -e POSTGRES_PASSWORD=llamastack \
  -e POSTGRES_DB=llamastack \
  -p 5432:5432 \
  postgres:15
```

#### Step 2: Export Environment Variables

```bash
# PostgreSQL configuration
export POSTGRES_HOST="localhost"
export POSTGRES_PORT="5432"
export POSTGRES_DB="llamastack"
export POSTGRES_USER="llamastack"
export POSTGRES_PASSWORD="llamastack"

# vLLM configuration (adjust to your vLLM server)
export VLLM_URL="http://localhost:8000"

# Model configuration
export INFERENCE_MODEL="TinyLlama/TinyLlama-1.1B-Chat-v1.0"
```

#### Step 3: Run Llama Stack Distribution

```bash
podman run -d \
  --name llama-stack \
  -p 8321:8321 \
  -e POSTGRES_HOST="${POSTGRES_HOST}" \
  -e POSTGRES_PORT="${POSTGRES_PORT}" \
  -e POSTGRES_DB="${POSTGRES_DB}" \
  -e POSTGRES_USER="${POSTGRES_USER}" \
  -e POSTGRES_PASSWORD="${POSTGRES_PASSWORD}" \
  -e VLLM_URL="${VLLM_URL}" \
  -e INFERENCE_MODEL="${INFERENCE_MODEL}" \
  quay.io/rh-ee-dkushwah/llama-stack-distribution:rhoai-3.2-s390x
```

#### Step 4: Verify

```bash
# Check if container is running
podman ps

# Check logs
podman logs llama-stack

# Test inference
curl http://localhost:8321/v1/models
```
### Option 2: Kubernetes/OpenShift Deployment

#### Step 0: Install Llama Stack Operator (If Not Already Installed)

Before deploying the distribution, you must install the Llama Stack Operator:

```bash
# Clone the operator repository
git clone https://github.com/llamastack/llama-stack-k8s-operator.git
cd llama-stack-k8s-operator

# Install the operator using Makefile
make deploy

# Verify the operator is running
kubectl get pods -n llama-stack-k8s-operator-system

# You should see the operator pod in Running state:
# NAME                                                   READY   STATUS    RESTARTS   AGE
# llama-stack-k8s-operator-controller-manager-xxxxx      1/1     Running   0          1m
```

### Option 2: Kubernetes/OpenShift Deployment

#### Step 1: Create PostgreSQL Secret

```bash
kubectl create secret generic postgres-secret \
  --from-literal=POSTGRESQL_USER=llamastack \
  --from-literal=POSTGRESQL_PASSWORD=llamastack \
  --from-literal=POSTGRESQL_DB=llamastack \
  -n your-namespace
```

#### Step 2: Deploy PostgreSQL

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: your-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: registry.redhat.io/rhel9/postgresql-15:latest
        env:
        - name: POSTGRESQL_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRESQL_USER
        - name: POSTGRESQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRESQL_PASSWORD
        - name: POSTGRESQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRESQL_DB
        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: your-namespace
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

#### Step 3: Deploy Llama Stack Distribution

```yaml
apiVersion: llamastack.io/v1alpha1
kind: LlamaStackDistribution
metadata:
  name: llamastack-s390x
  namespace: your-namespace
spec:
  replicas: 1
  server:
    distribution:
      image: "Provide you image name here"
    containerSpec:
      port: 8321
      resources:
        requests:
          memory: "4Gi"
          cpu: "1000m"
        limits:
          memory: "8Gi"
          cpu: "4000m"
      env:
      # PostgreSQL configuration (MANDATORY)
      - name: POSTGRES_HOST
        value: "postgres.your-namespace.svc.cluster.local"
      - name: POSTGRES_PORT
        value: "5432"
      - name: POSTGRES_USER
        valueFrom:
          secretKeyRef:
            name: postgres-secret
            key: POSTGRESQL_USER
      - name: POSTGRES_PASSWORD
        valueFrom:
          secretKeyRef:
            name: postgres-secret
            key: POSTGRESQL_PASSWORD
      - name: POSTGRES_DB
        valueFrom:
          secretKeyRef:
            name: postgres-secret
            key: POSTGRESQL_DB
      # vLLM configuration
      - name: VLLM_URL
        value: "http://vllm-server-service.your-namespace.svc.cluster.local:8000"
      # Model configuration
      - name: INFERENCE_MODEL
        value: "TinyLlama/TinyLlama-1.1B-Chat-v1.0"
    storage:
      size: "20Gi"
      mountPath: "/opt/app-root/.llama"
```

#### Step 4: Verify Deployment

```bash
# Check pod status
kubectl get pods -n your-namespace

# Check logs
kubectl logs -n your-namespace deployment/llamastack-s390x

# Port-forward for testing
kubectl port-forward -n your-namespace svc/llamastack-s390x 8321:8321

# Test inference (in another terminal)
curl http://localhost:8321/v1/models
```

## 🔍 Troubleshooting

### Issue: Container fails with "connection refused" to PostgreSQL

**Cause**: PostgreSQL environment variables not set or incorrect

**Solution**:
```bash
# Check if environment variables are set
echo $POSTGRES_HOST
echo $POSTGRES_USER

# Verify PostgreSQL is accessible
psql -h $POSTGRES_HOST -p $POSTGRES_PORT -U $POSTGRES_USER -d $POSTGRES_DB

# Check container logs
podman logs llama-stack
```

### Issue: "Model not found" error

**Cause**: Model ID doesn't match what's loaded in vLLM

**Solution**:
```bash
# Check what models are available in vLLM
curl http://your-vllm-server:8000/v1/models

# Update INFERENCE_MODEL to match
export INFERENCE_MODEL="the-model-id-from-vllm"
```

### Issue: Container starts but inference fails

**Cause**: vLLM URL is incorrect or vLLM is not running

**Solution**:
```bash
# Verify vLLM is accessible
curl http://your-vllm-server:8000/v1/models

# Update VLLM_URL if needed
export VLLM_URL="http://correct-vllm-url:8000"
```

## 📊 Environment Variable Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `POSTGRES_HOST` | ✅ Yes | `localhost` | PostgreSQL server hostname |
| `POSTGRES_PORT` | ✅ Yes | `5432` | PostgreSQL server port |
| `POSTGRES_DB` | ✅ Yes | `llamastack` | PostgreSQL database name |
| `POSTGRES_USER` | ✅ Yes | `llamastack` | PostgreSQL username |
| `POSTGRES_PASSWORD` | ✅ Yes | `llamastack` | PostgreSQL password |
| `VLLM_URL` | ✅ Yes | `http://localhost:8000` | vLLM server URL |
| `INFERENCE_MODEL` | ⚠️ Optional | `TinyLlama/TinyLlama-1.1B-Chat-v1.0` | Model ID for inference |
| `PORT` | ⚠️ Optional | `8321` | Server listen port |

## 🎯 Quick Start Commands

### For Standalone Deployment:

```bash
# 1. Export all required environment variables
export POSTGRES_HOST="localhost"
export POSTGRES_PORT="5432"
export POSTGRES_DB="llamastack"
export POSTGRES_USER="llamastack"
export POSTGRES_PASSWORD="llamastack"
export VLLM_URL="http://localhost:8000"
export INFERENCE_MODEL="TinyLlama/TinyLlama-1.1B-Chat-v1.0"

# 2. Run the container
podman run -d \
  --name llama-stack \
  -p 8321:8321 \
  -e POSTGRES_HOST -e POSTGRES_PORT -e POSTGRES_DB \
  -e POSTGRES_USER -e POSTGRES_PASSWORD \
  -e VLLM_URL -e INFERENCE_MODEL \
  quay.io/rh-ee-dkushwah/llama-stack-distribution:rhoai-3.2-s390x

# 3. Test
curl http://localhost:8321/v1/models
```

### For Kubernetes Deployment:

See the complete YAML examples in **Option 2** above.

## 📚 Additional Resources

- **Build Guide**: See `build-s390x.sh` for building the image
- **PR Description**: See `PR_DESCRIPTION_S390X.md` for technical details
- **Quick Start**: See `QUICK_START_S390X.md` for quick reference

## ⚠️ Important Notes

1. **PostgreSQL is MANDATORY** - The s390x distribution requires PostgreSQL for storage
2. **Environment variables must be set** - The container will fail if PostgreSQL variables are not configured
3. **vLLM must be running** - The inference API requires a running vLLM server
4. **Use secrets in production** - Never hardcode passwords in deployment files

---
