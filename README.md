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

kubectl get nodes
`

You should see one node in "Ready" status.
