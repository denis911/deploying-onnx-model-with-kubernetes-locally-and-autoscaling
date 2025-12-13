# deploying-onnx-model-with-kubernetes-locally-and-autoscaling

Create ONNX model and deploy to a local Kubernetes cluster using Kind (Kubernetes in Docker)

## STEP 0. Create a Kind Cluster

Let's create a local Kubernetes cluster:

`
kind create cluster --name mlzoomcamp
`

This will:

Create a single-node Kubernetes cluster

Configure kubectl to use this cluster

Take a few minutes on first run (downloads images)

Verify the cluster is running:

`
kubectl cluster-info
`

`
kubectl get nodes
`

We should see one node in "Ready" status.

## STEP 1. Model Preparation

We will use a pre-trained PyTorch model that classifies clothing items. The model has already been converted to ONNX format.

Download the ONNX model:

```bash
mkdir service

cd service

wget https://github.com/DataTalksClub/machine-learning-zoomcamp/releases/download/dl-models/clothing_classifier_mobilenet_v2_latest.onnx -O clothing-model.onnx

```

The model predicts one of 10 clothing categories from an image URL.

## STEP 2. Building the FastAPI Service

We will create a FastAPI application that serves the ONNX model for inference.

First, initialize the project:

```bash

# cd service
uv init
rm main.py

```

Add dependencies:

`
uv add fastapi uvicorn onnxruntime keras-image-helper numpy
`

Now create the FastAPI application. See service/app.py.

Key points:

- Uses ONNX Runtime for inference

- Custom PyTorch-style preprocessing function (the PyTorch model requires specific normalization)

- keras-image-helper for image preprocessing

- Pydantic models for input/output validation

- Health endpoint for Kubernetes health checks

## STEP 3. Testing Locally

Run the service:

(cd service first... then check if win firewall is NOT blocking it...)

`
uv run uvicorn app:app --host 0.0.0.0 --port 8080 --reload
`

Open http://localhost:8080/docs to see the Swagger UI API documentation.

Then run the test:

`
uv run python test.py
`

We should see predictions for the clothing item...

Alternative: Test with curl on Windows (on PowerShell):

```bash
curl.exe -X POST http://localhost:8080/predict `
  -H "Content-Type: application/json" `
  -d '{\"url\": \"http://bit.ly/mlbookcamp-pants\"}'
```

Or check the health endpoint:
`
curl http://localhost:8080/health
`

## STEP 4. Docker Containerization

Now let's containerize our application.

Create service/Dockerfile.

Build the image:

`
docker build -t clothing-classifier:v1 .
`

Test the container locally:

`
docker run -it --rm -p 8080:8080 clothing-classifier:v1
`

In another terminal, run the test script:

`
uv run python test.py
`

We should stop our uvicorn in its own terminal by ctrl-c to make sure docker can run on the same port.

## STEP 5. Loading Image to Kind

Kind clusters run in Docker, so they can't access images from your local Docker daemon by default. We need to load the image into Kind.

`
kind load docker-image clothing-classifier:v1 --name mlzoomcamp
`

## STEP 6. Kubernetes Deployment

Now let's deploy our service to Kubernetes.

Understanding Kubernetes Resources

- Pod: The smallest deployable unit in Kubernetes (one or more containers)

- Deployment: Manages a set of identical Pods, handles updates and scaling

- Service: Exposes Pods to network traffic

- HPA (Horizontal Pod Autoscaler): Automatically scales Pods based on metrics

Create Deployment Manifest

Let's create a folder for all the config files:

```bash
mkdir k8s
cd k8s
```

Create k8s/deployment.yaml

Key configuration:

- replicas: 2 - Run 2 copies of our service

- imagePullPolicy: Never - Use local image (don't pull from registry)

- resources - Memory and CPU limits/requests

- livenessProbe - Restart container if unhealthy

- readinessProbe - Only send traffic when ready

Deploy it:

```bash
kubectl apply -f deployment.yaml
```

Check the deployment:

```bash
kubectl get deployments
kubectl get pods
kubectl describe deployment clothing-classifier
```

View logs:

```bash
kubectl logs -l app=clothing-classifier --tail=20
```

Create Service Manifest

Create k8s/service.yaml.

Key configuration:

- type: NodePort - Expose on a static port on each node

- nodePort: 30080 - Accessible on port 30080 from host

- selector - Routes traffic to Pods with matching labels

Deploy it:

```bash
kubectl apply -f k8s/service.yaml
```

Check the service:

```bash
kubectl get services
kubectl describe service clothing-classifier
```

Testing the Deployed Service

With NodePort, the service is accessible on localhost:30080:

Check the health endpoint:

```bash
curl http://localhost:30080/health
```

Our kind cluster is not configured for NodePort, so it won't work. We don't really need this for testing things locally, so let's just use a quick fix: Use kubectl port-forward.

```bash
kubectl port-forward service/clothing-classifier 30080:8080
```

Now it's accessible on port 30080

```bsh
curl http://localhost:30080/health
```

When we deploy to EKS or some other Kubernetes in the cloud, it won't be a problem - there Elastic Load Balancer will solve this problem.

## STEP 7. Horizontal Pod Autoscaling

Kubernetes can automatically scale your application based on CPU or memory usage.

First, we need metrics-server for HPA to work. Install it in kubectl:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

For Kind, we need to patch metrics-server to work without TLS:

```bash
kubectl patch -n kube-system deployment metrics-server --type=json -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

or for windows 1-line CMD shell command:

```bash
kubectl patch -n kube-system deployment metrics-server --type=json -p "[{\"op\":\"add\",\"path\":\"/spec/template/spec/containers/0/args/-\",\"value\":\"--kubelet-insecure-tls\"}]"
```

Wait for metrics-server to be ready:

```bash
kubectl get deployment metrics-server -n kube-system
```

We should see something like that:

```bash
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           72s
```

Now create k8s/hpa.yaml.

Configuration:

- Scale between 2 and 10 replicas

- Target 50% CPU utilization

- Automatically adds/removes Pods based on load

Deploy HPA:

```bash
kubectl apply -f hpa.yaml
```

Check HPA status:

```bash
kubectl get hpa
kubectl describe hpa clothing-classifier-hpa
```

## STEP 8. Testing Autoscaling

Generate load to trigger autoscaling. You can use a simple load test (see load_test.py)

First, check that you can access the endpoint:

```bash
curl http://localhost:30080/health
```

Run the test:

```bash
uv run python load_test.py
```

While running the load test, watch the HPA in another terminal:

```bash
kubectl get hpa -w
```

We should see the number of replicas increase as CPU usage rises.

Check Pods:

```bash
kubectl get pods -w
```
