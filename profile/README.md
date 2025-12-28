# KalypsoServing 

Model Serving Platform

* This project is currently incomplete

## Step 1: Prepare Cluster

### Using Kind

```bash
# Create Kind cluster
kind create cluster

```

### Using Existing Cluster

```bash
# Check current context
kubectl config current-context

# Switch context if needed
kubectl config use-context kind-kind
```

## Step 2: Install KalypsoServing

```shell
helm repo add kalypso https://kalypsoserving.github.io/helm-charts

helm install --create-namespace -n kalypso-system kalypso-serving kalypso/kalypso-serving 

```


## Step 3: Create Your First Resources

### 3.1 Create KalypsoProject

KalypsoProject defines a logical workspace for ML projects.

```yaml
# project.yaml
apiVersion: serving.serving.kalypso.io/v1alpha1
kind: KalypsoProject
metadata:
  name: sample-project
  namespace: kalypso-system
spec:
  displayName: "Sample Project"
  owner: "team-ai-research"
  environments:
    dev:
      namespace: sample-project-dev
      description: "Development environment"

    stage:
      namespace: sample-project-stage
      description: "Pre-deployment verification stage"
      resourceQuota:
        limits:
          nvidia.com/gpu: "1"
    prod:
      namespace: sample-project-prod
      description: "Production environment"
      resourceQuota:
        limits:
          nvidia.com/gpu: "10"
```

```bash
kubectl apply -f project.yaml
```

### 3.2 Create KalypsoApplication

KalypsoApplication groups related Triton servers together.

```yaml
# application.yaml
apiVersion: serving.serving.kalypso.io/v1alpha1
kind: KalypsoApplication
metadata:
  name: recommendation-application
  namespace: kalypso-system
spec:
  projectRef: "sample-project"
  description: "Product recommendation model serving application"
  modelStorage:
    url: "s3://kalypso-models/sample-project"
    secretName: "aws-s3-credentials"
    endpoint: "http://minio.minio.svc:9000"
```

```bash
kubectl apply -f application.yaml
```

### 3.3 Create KalypsoTritonServer

KalypsoTritonServer deploys actual Triton Inference Server instances.

```yaml
# tritonserver.yaml
apiVersion: serving.serving.kalypso.io/v1alpha1
kind: KalypsoTritonServer
metadata:
  name: add-sub-server
spec:
  applicationRef: "recommendation-application"
  
  tritonConfig:
    image: "nvcr.io/nvidia/tritonserver"
    tag: "24.12-py3"
    backendType: "python"
  replicas: 1
  resources:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "1Gi"
  
  observability:
    logging: # file scrape (loki)
      enabled: true
      level: "INFO"
    metrics: # prometheus (mimir)
      enabled: true
    tracing: # otel  (tempo)
      enabled: true
    profiling: # pyrospcoe (with alloy)
      enabled: true
      cpu: true
      memory: true
     
```

```bash
kubectl apply -f tritonserver.yaml
```

## Step 4: Verify Status

### Check Resource Status

```bash
# Check KalypsoProject status
kubectl get kalypsoproject -n kalypso-system
# NAME             PHASE   OWNER              AGE
# sample-project   Ready   team-ai-research   1m

# Check created namespaces
kubectl get ns -l kalypso-serving.io/project=sample-project
# NAME                   STATUS   AGE
# sample-project-dev     Active   1m
# sample-project-prod    Active   1m
# sample-project-stage   Active   1m

# Check KalypsoApplication status
kubectl get kalypsoapplication -n kalypso-system
# NAME                         PROJECT          PHASE   MODELS   AGE
# recommendation-application   sample-project   Ready   1        1m

# Check KalypsoTritonServer status
kubectl get kalypsotritonserver -n kalypso-system
# NAME             APPLICATION                  PHASE     REPLICAS   AVAILABLE   AGE
# add-sub-server   recommendation-application   Running   1          1           1m

# Check Deployment and Service
kubectl get deployment,svc -n kalypso-system -l kalypso-serving.io/tritonserver=add-sub-server
```

### Check Detailed Information

```bash
# KalypsoProject details
kubectl describe kalypsoproject sample-project -n kalypso-system

# KalypsoTritonServer details
kubectl describe kalypsotritonserver add-sub-server -n kalypso-system

# Check Pod logs
kubectl logs -n kalypso-system -l kalypso-serving.io/tritonserver=add-sub-server
```

## Step 5: Test Model Serving

### HTTP API Testing

```bash
# Port forward Service
kubectl port-forward -n kalypso-system svc/add-sub-server-http 8000:8000

# In another terminal, Health Check
curl http://localhost:8000/v2/health/ready

# List models
curl http://localhost:8000/v2/models

# Model inference (example)
curl -X POST http://localhost:8000/v2/models/add_sub/infer \
  -H "Content-Type: application/json" \
  -d '{
    "inputs": [
      {
        "name": "INPUT0",
        "shape": [1],
        "datatype": "INT32",
        "data": [1]
      },
      {
        "name": "INPUT1",
        "shape": [1],
        "datatype": "INT32",
        "data": [2]
      }
    ],
    "outputs": [{"name": "OUTPUT0"}]
  }'
```

### gRPC API Testing

```bash
# Port forward gRPC port
kubectl port-forward -n kalypso-system svc/add-sub-server-grpc 8001:8001

# Use gRPC client (grpcurl, etc.)
grpcurl -plaintext localhost:8001 list
```

## Step 6: Cleanup

### Delete Resources

```bash
# Delete KalypsoTritonServer
kubectl delete kalypsotritonserver add-sub-server -n kalypso-system

# Delete KalypsoApplication
kubectl delete kalypsoapplication recommendation-application -n kalypso-system

# Delete KalypsoProject
kubectl delete kalypsoproject sample-project -n kalypso-system

# Delete all resources at once
kubectl delete kalypsotritonserver --all -n kalypso-system
kubectl delete kalypsoapplication --all -n kalypso-system
kubectl delete kalypsoproject --all -n kalypso-system
```

### Remove Controller and CRDs

```bash
# Remove Controller
make undeploy

# Remove CRDs
make uninstall

# Delete namespace
kubectl delete namespace kalypso-system
```

## Next Steps

- Check the full documentation in [README.md](README.md)
- Review detailed fields for each resource in [CRD Reference](README.md#crd-reference)
- Configure Observability (Logging, Tracing, Profiling, Metrics)
- Allocate and manage GPU resources
- Set up model storage (S3, GCS, etc.)

## Troubleshooting

### Controller Not Running

```bash
# Check Controller Pod status
kubectl get pods -n kalypso-system

# Check Controller logs
kubectl logs -n kalypso-system -l control-plane=controller-manager
```

### Resources Not Reaching Ready State

```bash
# Check resource events
kubectl describe kalypsotritonserver <name> -n kalypso-system

# Check related Pod status
kubectl get pods -n kalypso-system -l kalypso-serving.io/tritonserver=<name>
```

### Namespaces Not Created

```bash
# Check KalypsoProject events
kubectl describe kalypsoproject <name> -n kalypso-system

# Check for errors in Controller logs
kubectl logs -n kalypso-system -l control-plane=controller-manager | grep -i error
```

## Additional Resources

- [NVIDIA Triton Inference Server Documentation](https://docs.nvidia.com/deeplearning/triton-inference-server/)
- [Kubebuilder Documentation](https://book.kubebuilder.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
