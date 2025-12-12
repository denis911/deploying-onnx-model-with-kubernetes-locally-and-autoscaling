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

`
mkdir service

cd service

wget https://github.com/DataTalksClub/machine-learning-zoomcamp/releases/download/dl-models/clothing_classifier_mobilenet_v2_latest.onnx -O clothing-model.onnx
`
