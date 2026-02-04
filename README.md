# Clothing Model Classifier - Kubernetes Deployment

## Overview

This project demonstrates deploying a machine learning model to a local Kubernetes cluster using Docker and kind (Kubernetes in Docker). 

The application is a **clothing classifier** that uses an existing ONNX model to predict one of 10 clothing categories (dress, hat, longsleeve, outwear, pants, shirt, shoes, shorts, skirt, t-shirt) from image URLs. The service is built with FastAPI and deployed as containerized microservices in Kubernetes with horizontal auto-scaling capabilities.

### Key Features

- **FastAPI Service**: RESTful API for model inference with `/predict` and `/health` endpoints
- **Docker Containerization**: Packaged application with all dependencies
- **Local Kubernetes Cluster**: Using kind for cost-free, local Kubernetes deployment
- **Horizontal Pod Autoscaling (HPA)**: Automatic scaling based on CPU utilization
- **Health Checks**: Liveness and readiness probes for reliable deployments
- **Load Testing**: Included `load_test.py` to validate HPA behavior

---

## Prerequisites

- Docker (installed and running)
- kubectl (Kubernetes command-line tool)
- kind (Kubernetes in Docker)
- Python 3.11+ with uv package manager
- Basic familiarity with Docker and Kubernetes concepts

---

## Project Structure

```
kubernetes-tutorial/
├── src/
│   ├── app.py                 # FastAPI application
│   └── load_test.py          # Load testing script for HPA validation
├── k8s/
│   ├── deployment.yaml        # Kubernetes Deployment manifest
│   ├── service.yaml           # Kubernetes Service manifest
│   └── hpa.yaml              # Horizontal Pod Autoscaler manifest
├── Dockerfile                 # Docker image definition
├── pyproject.toml             # Python dependencies
├── clothing-model.onnx        # Pre-trained ONNX model
└── README.md                 # This file
```

---

## Docker Configuration

### Dockerfile Purpose

The `Dockerfile` containerizes the FastAPI application with all required dependencies:

- **Base Image**: Python 3.11+ slim image
- **Dependencies**: 
  - `fastapi`: Web framework
  - `uvicorn`: ASGI server
  - `onnxruntime`: ONNX model inference engine
  - `keras-image-helper`: Image preprocessing utilities
  - `numpy`: Numerical computing

### Building and Running Locally

**Build the Docker image:**
```bash
cd src
docker build -t clothing-classifier:v1 .
```

**Run the container locally:**
```bash
docker run -it --rm -p 8080:8080 clothing-classifier:v1
```

**Test the service:**
```bash
curl http://localhost:8080/health
```

---

## Kubernetes Configuration

### Key YAML Manifests

#### 1. `deployment.yaml`

Manages the deployment of the clothing classifier application.

**Key Features:**
- **Replicas**: Runs 2+ copies of the service for high availability
- **Image Pull Policy**: `imagePullPolicy: Never` to use local Docker images
- **Resource Management**: 
  - Requests: CPU and memory reservation
  - Limits: Maximum CPU and memory allocation
- **Health Checks**:
  - **Liveness Probe**: Restarts container if `/health` endpoint fails
  - **Readiness Probe**: Only routes traffic when service is ready
- **Port**: Container listens on port 8080

**Example deployment:**
```bash
kubectl apply -f k8s/deployment.yaml
kubectl get deployments
kubectl get pods
```

#### 2. `service.yaml`

Exposes the application to network traffic.

**Key Features:**
- **Type**: `NodePort` for external access
- **Node Port**: 30080 (accessible from host)
- **Selector**: Routes traffic to pods with label `app=clothing-classifier`
- **Port Mapping**: Internal port 8080 → External port 30080

**Example deployment:**
```bash
kubectl apply -f k8s/service.yaml
kubectl get services
```

#### 3. `hpa.yaml`

Enables automatic horizontal scaling.

**Key Features:**
- **Min Replicas**: 2 pods minimum
- **Max Replicas**: 10 pods maximum
- **Target CPU**: 50% average CPU utilization
- **Scaling Behavior**: Automatically adds/removes pods based on load

**Example deployment:**
```bash
kubectl apply -f k8s/hpa.yaml
kubectl get hpa
kubectl describe hpa clothing-classifier-hpa
```

---

## Quick Start Guide

### 1. Create a Kind Cluster

```bash
kind create cluster --name mlzoomcamp
```

Verify cluster is running:
```bash
kubectl cluster-info
kubectl get nodes
```

### 2. Build and Load Docker Image

```bash
cd src
docker build -t clothing-classifier:v1 .
kind load docker-image clothing-classifier:v1 --name mlzoomcamp
```

### 3. Deploy to Kubernetes

```bash
cd ../k8s
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Verify deployment:
```bash
kubectl get deployments
kubectl get pods
kubectl get services
```

### 4. Setup Port Forwarding

For kind clusters without NodePort support:
```bash
kubectl port-forward service/clothing-classifier 8080:8080
```

---

## Testing the Service

### Health Check

```bash
curl http://localhost:8080/health
```

Expected response:
```json
{"status": "healthy"}
```

### Prediction Request

```bash
curl -X POST http://localhost:8080/predict \
  -H "Content-Type: application/json" \
  -d '{"url": "http://bit.ly/mlbookcamp-pants"}'
```

Expected response:
```json
{
  "predictions": {
    "dress": 0.01,
    "hat": 0.02,
    "longsleeve": 0.05,
    "outwear": 0.03,
    "pants": 0.08,
    "shirt": 0.15,
    "shoes": 0.04,
    "shorts": 0.06,
    "skirt": 0.02,
    "t-shirt": 0.54
  },
  "top_class": "t-shirt",
  "top_probability": 0.54
}
```

---

## HPA Testing and Autoscaling

### Prerequisites for HPA

HPA requires metrics-server to monitor resource usage:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

For kind, patch metrics-server to skip TLS verification:

```bash
kubectl patch deployment metrics-server -n kube-system --type=json -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

Verify metrics-server is ready:
```bash
kubectl get deployment metrics-server -n kube-system
```

### Deploy HPA

```bash
kubectl apply -f k8s/hpa.yaml
```

### Run Load Test

In separate terminals, run the load test and watch scaling:

```bash
# Terminal 1: Start load test
uv run python load_test.py

# Terminal 2: Monitor HPA and pods
kubectl get hpa -w
kubectl get pods -w
```

**Expected Behavior:**
- As load increases, CPU usage rises
- HPA detects CPU > 50% threshold
- New pods are created automatically
- As load decreases, pods are removed

### Example HPA Output

```
NAME                          REFERENCE                              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
clothing-classifier-hpa       Deployment/clothing-classifier         85%/50%   2         10        5          2m
```

---

## Common Commands Reference

### Docker Commands

```bash
# Build image
docker build -t clothing-classifier:v1 .

# Run container locally
docker run -it --rm -p 8080:8080 clothing-classifier:v1

# List images
docker images

# Remove image
docker rmi clothing-classifier:v1
```

### Kind & Cluster Management

```bash
# Create cluster
kind create cluster --name mlzoomcamp

# Load local Docker image to kind
kind load docker-image clothing-classifier:v1 --name mlzoomcamp

# Get cluster info
kubectl cluster-info

# List clusters
kind get clusters

# Delete cluster
kind delete cluster --name mlzoomcamp
```

### kubectl Deployment

```bash
# Apply manifests
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa.yaml

# View resources
kubectl get deployments
kubectl get pods
kubectl get services
kubectl get hpa

# Describe resources
kubectl describe deployment clothing-classifier
kubectl describe pod <pod-name>
kubectl describe service clothing-classifier
```

### Service Testing

```bash
# Port forward service
kubectl port-forward service/clothing-classifier 8080:8080

# Test health endpoint
curl http://localhost:8080/health

# Access API docs
# Open browser: http://localhost:8080/docs
```

### Debugging and Logs

```bash
# View logs from all pods
kubectl logs -l app=clothing-classifier --tail=50

# View logs from specific pod
kubectl logs <pod-name>

# Follow logs in real-time
kubectl logs -f -l app=clothing-classifier

# Execute command in pod
kubectl exec -it <pod-name> -- /bin/bash

# Get events
kubectl get events --sort-by='.lastTimestamp'
```

### Manual Scaling

```bash
# Scale to 5 replicas
kubectl scale deployment clothing-classifier --replicas=5

# Check rollout status
kubectl rollout status deployment/clothing-classifier

# Restart deployment
kubectl rollout restart deployment clothing-classifier
```

### Cleanup

```bash
# Delete all resources
kubectl delete all -l app=clothing-classifier
kubectl delete hpa clothing-classifier-hpa

# Delete kind cluster
kind delete cluster --name [clustername]
```

---

## Updating the Application

### When Making Code Changes

1. **Rebuild Docker image with new tag:**
   ```bash
   docker build -t clothing-classifier:v2 .
   ```

2. **Load to kind:**
   ```bash
   kind load docker-image clothing-classifier:v2 --name [clustername]
   ```

3. **Update deployment image:**
   ```bash
   kubectl set image deployment/clothing-classifier \
     clothing-classifier=clothing-classifier:v2
   ```

4. **Watch the rollout:**
   ```bash
   kubectl rollout status deployment/clothing-classifier
   ```

---

## Troubleshooting

### Pods not starting

```bash
# Check pod status and events
kubectl describe pod <pod-name>

# View logs
kubectl logs <pod-name>
```

### Service not accessible

```bash
# Check service is created
kubectl get services

# Use port-forward (for kind)
kubectl port-forward service/clothing-classifier 8080:8080
```

### HPA not scaling

```bash
# Check metrics-server is running
kubectl get deployment metrics-server -n kube-system

# Check HPA status
kubectl describe hpa clothing-classifier-hpa

# Verify metrics are available
kubectl top pods
kubectl top nodes
```

### Image not found

Ensure image is loaded to kind:
```bash
kind load docker-image clothing-classifier:v1 --name [clustername]
```

---

## Summary

This project demonstrates the complete workflow of deploying a machine learning model to Kubernetes:

**Local Kubernetes Setup**
- ✅ Installing kubectl and kind
- ✅ Creating a local Kubernetes cluster
- ✅ Basic kubectl commands

**ONNX Model Deployment**
- ✅ Using pre-converted ONNX model for inference
- ✅ Building FastAPI service with ONNX Runtime

**Docker**
- ✅ Containerizing the application with Docker
- ✅ Building and tagging Docker images
- ✅ Loading images to kind cluster

**Kubernetes Deployment**
- ✅ Creating Deployment manifests
- ✅ Exposing services with NodePort
- ✅ Health checks (liveness and readiness probes)
- ✅ Resource limits and requests

**Scaling and Management**
- ✅ Horizontal Pod Autoscaling based on CPU utilization
- ✅ Manual scaling
- ✅ Rolling updates
- ✅ Debugging and logging

The setup is production-ready and scalable to cloud environments.
---

## Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [kind Documentation](https://kind.sigs.k8s.io/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [ONNX Runtime](https://onnxruntime.ai/)
- [Reference: ML Zoomcamp Kubernetes Workshop](https://github.com/alexeygrigorev/workshops/blob/main/mlzoomcamp-k8s/README.md)

---

## License

This project is provided as-is for educational purposes.
