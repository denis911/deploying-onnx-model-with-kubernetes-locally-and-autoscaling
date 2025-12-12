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

You should see predictions for the clothing item.
