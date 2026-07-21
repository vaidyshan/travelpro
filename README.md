# Travelpro - Travel the world

A simple static website served by Nginx. It can run as a standalone Docker
container or in a local kind Kubernetes cluster.

## Prerequisites

Install and start the following tools:

- Git
- Docker Desktop
- `kubectl`
- `kind`

Verify the installation:

```bash
git --version
docker --version
kubectl version --client
kind version
docker info
```

## Get the source code

Clone the repository over SSH and enter the project directory:

```bash
git clone git@github.com:vaidyshan/travelpro.git
cd travelpro
```

## Run with Docker

Build the image:

```bash
docker build -t travelpro:local .
```

Run the application on port 8080:

```bash
docker run -d --name travelpro -p 8080:80 travelpro:local
```

Open <http://localhost:8080>.

Check or stop the container:

```bash
docker ps --filter name=travelpro
docker logs travelpro
docker stop travelpro
docker rm travelpro
```

## Deploy to a local kind cluster

The kind configuration maps host port `8080` to Kubernetes node port `30080`.
The application is therefore available without running `kubectl port-forward`.

The traffic path is:

```text
localhost:8080 -> kind node:30080 -> Service:80 -> Pod:80
```

### 1. Build the local image

From the repository root:

```bash
docker build -t travelpro:local .
```

### 2. Release port 8080

Only one process can bind to port 8080. If the standalone Docker container is
running, stop and remove it before creating the cluster:

```bash
docker stop travelpro
docker rm travelpro
```

Skip these commands if the container is not running.

### 3. Create the kind cluster

```bash
kind create cluster --name travelpro --config k8s/kind-config.yaml
kubectl cluster-info --context kind-travelpro
kubectl get nodes
```

If a cluster named `travelpro` already exists and you need to recreate its port
mapping, delete it first. This removes every workload in that cluster:

```bash
kind delete cluster --name travelpro
kind create cluster --name travelpro --config k8s/kind-config.yaml
```

### 4. Load the image into kind

kind nodes cannot automatically use images from the host Docker image store.
Load the image explicitly:

```bash
kind load docker-image travelpro:local --name travelpro
```

Verify the image on the node:

```bash
docker exec travelpro-control-plane crictl images | grep travelpro
```

### 5. Deploy the Kubernetes resources

```bash
kubectl config use-context kind-travelpro
kubectl apply -f k8s/deployment.yaml
kubectl rollout status deployment/travelpro
```

Verify the resources:

```bash
kubectl get deployments
kubectl get pods -l app=travelpro
kubectl get service travelpro
```

The pod should show `Running` with `1/1` containers ready. Open
<http://localhost:8080> to view the application.

## Deploy application updates to kind

After changing `index.html` or `styles.css`, rebuild and reload the image, then
restart the Deployment:

```bash
docker build -t travelpro:local .
kind load docker-image travelpro:local --name travelpro
kubectl rollout restart deployment/travelpro
kubectl rollout status deployment/travelpro
```

Refresh <http://localhost:8080> after the rollout completes.

## Troubleshooting

Inspect the application and recent cluster events:

```bash
kubectl get pods -l app=travelpro -o wide
kubectl describe deployment travelpro
kubectl describe pod -l app=travelpro
kubectl logs deployment/travelpro
kubectl get events --sort-by=.metadata.creationTimestamp
```

If the pod reports `ErrImageNeverPull`, rebuild and reload the image:

```bash
docker build -t travelpro:local .
kind load docker-image travelpro:local --name travelpro
kubectl rollout restart deployment/travelpro
```

If port 8080 is already occupied, identify the process using it:

```bash
lsof -nP -iTCP:8080 -sTCP:LISTEN
```

## Cleanup

Remove only the application resources:

```bash
kubectl delete -f k8s/deployment.yaml
```

Remove the entire local cluster:

```bash
kind delete cluster --name travelpro
```
