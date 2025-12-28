# KalypsoServing 

Model Serving Platform

* This project is currently incomplete

## Step 1: Install

```shell
helm repo add kalypso https://kalypsoserving.github.io/helm-charts

helm install --create-namespace -n kalypso-system kalypso-serving kalypso/kalypso-serving 
```

## Step 2: Apply Kalypso Custom Resources

<details>
<summary><b>KalypsoProject</b></summary>

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
</details>

<details>
<summary><b>KalypsoApplication</b></summary>

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
</details>

<details>
<summary><b>KalypsoTritonServer</b></summary>

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
</details>

## Verify Status

<details>
<summary><b>Check Resource Status</b></summary>

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
</details>

<details>
<summary><b>Check Detailed Information</b></summary>

```bash
# KalypsoProject details
kubectl describe kalypsoproject sample-project -n kalypso-system

# KalypsoTritonServer details
kubectl describe kalypsotritonserver add-sub-server -n kalypso-system

# Check Pod logs
kubectl logs -n kalypso-system -l kalypso-serving.io/tritonserver=add-sub-server
```
</details>

## Test Model Serving

<details>
<summary><b>HTTP API Testing</b></summary>

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
</details>

<details>
<summary><b>gRPC API Testing</b></summary>

```bash
# Port forward gRPC port
kubectl port-forward -n kalypso-system svc/add-sub-server-grpc 8001:8001

# Use gRPC client (grpcurl, etc.)
grpcurl -plaintext localhost:8001 list
```
</details>

## Cleanup

<details>
<summary><b>Delete Resources</b></summary>

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
</details>

<details>
<summary><b>Remove Controller and CRDs</b></summary>

```bash
# Remove Controller
make undeploy

# Remove CRDs
make uninstall

# Delete namespace
kubectl delete namespace kalypso-system
```
</details>

## Next Steps

- Check the full documentation in [README.md](README.md)
- Review detailed fields for each resource in [CRD Reference](README.md#crd-reference)
- Configure Observability (Logging, Tracing, Profiling, Metrics)
- Allocate and manage GPU resources
- Set up model storage (S3, GCS, etc.)

## Troubleshooting

<details>
<summary><b>Controller Not Running</b></summary>

```bash
# Check Controller Pod status
kubectl get pods -n kalypso-system

# Check Controller logs
kubectl logs -n kalypso-system -l control-plane=controller-manager
```
</details>

<details>
<summary><b>Resources Not Reaching Ready State</b></summary>

```bash
# Check resource events
kubectl describe kalypsotritonserver <name> -n kalypso-system

# Check related Pod status
kubectl get pods -n kalypso-system -l kalypso-serving.io/tritonserver=<name>
```
</details>

<details>
<summary><b>Namespaces Not Created</b></summary>

```bash
# Check KalypsoProject events
kubectl describe kalypsoproject <name> -n kalypso-system

# Check for errors in Controller logs
kubectl logs -n kalypso-system -l control-plane=controller-manager | grep -i error
```
</details>

## Additional Resources

- [NVIDIA Triton Inference Server Documentation](https://docs.nvidia.com/deeplearning/triton-inference-server/)
- [Kubebuilder Documentation](https://book.kubebuilder.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
