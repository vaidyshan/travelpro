# Travelpro - Travel the world

A simple static website served by Nginx, built by GitLab CI, stored in the
private GitLab Container Registry, and deployed to a local multi-node kind
cluster.

## Delivery flow

```text
GitHub source
    -> GitLab CI checkout, test, and image build
    -> private GitLab Container Registry
    -> Kubernetes Deployment on kind
    -> http://localhost:8080
```

## Prerequisites

Install and start:

- Git
- Docker Desktop
- `kubectl`
- `kind`

Verify the tools:

```bash
git --version
docker --version
kubectl version --client
kind version
docker info
```

## Get the source code

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

Useful container commands:

```bash
docker ps --filter name=travelpro
docker logs travelpro
docker stop travelpro
docker rm travelpro
```

## Build and publish with GitLab CI

The GitLab CI project is:

<https://gitlab.com/vaidy.shanmugam1/travelpro>

The pipeline clones the public GitHub repository, tests the image contents,
and pushes two tags to the private GitLab Container Registry:

```text
registry.gitlab.com/vaidy.shanmugam1/travelpro:<GitHub-commit-SHA>
registry.gitlab.com/vaidy.shanmugam1/travelpro:latest
```

Kubernetes uses the immutable GitHub commit SHA tag rather than `latest` so
that every deployed version is reproducible and auditable.

## Create the multi-node kind cluster

The cluster contains one control-plane node and two worker nodes. The kind
configuration maps host port `8080` to Kubernetes NodePort `30080`.

```text
localhost:8080 -> kind node:30080 -> Service:80 -> Pod:80
```

Release port 8080 if the standalone Docker container is running:

```bash
docker stop travelpro
docker rm travelpro
```

Skip those commands if the container is not running.

Create the cluster:

```bash
kind create cluster --name kind --config k8s/kind-config.yaml
kubectl cluster-info --context kind-kind
kubectl get nodes
```

Expected nodes:

```text
kind-control-plane
kind-worker
kind-worker2
```

If the `kind` cluster already exists, do not recreate it unless necessary.
Recreating it deletes all workloads and Secrets in that cluster:

```bash
kind delete cluster --name kind
kind create cluster --name kind --config k8s/kind-config.yaml
```

## Allow Kubernetes to pull the private image

The GitLab Container Registry is private. Create a project deploy token in
GitLab under **Settings > Repository > Deploy tokens** with:

```text
Name: kind-registry-pull
Scope: read_registry only
```

Use an expiration date and save the generated username and token securely.
GitLab displays the token only once. Never commit the token or send it through
chat, email, logs, or screenshots.

Select the local cluster:

```bash
kubectl config use-context kind-kind
```

Read the credentials into temporary shell variables. Run each command first,
then enter the requested value at its prompt. Do not include the credential in
the command itself:

```bash
read -r "GITLAB_DEPLOY_USER?GitLab deploy username: "
read -rs "GITLAB_DEPLOY_TOKEN?GitLab deploy token: "
echo
```

Create or update the registry Secret:

```bash
kubectl create secret docker-registry gitlab-registry \
  --docker-server=registry.gitlab.com \
  --docker-username="$GITLAB_DEPLOY_USER" \
  --docker-password="$GITLAB_DEPLOY_TOKEN" \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

Immediately remove the credentials from the shell:

```bash
unset GITLAB_DEPLOY_USER GITLAB_DEPLOY_TOKEN
```

Verify the Secret without displaying its contents:

```bash
kubectl get secret gitlab-registry
kubectl get secret gitlab-registry -o jsonpath='{.type}{"\n"}'
```

Expected type:

```text
kubernetes.io/dockerconfigjson
```

The Secret is created directly in Kubernetes and is never committed to Git.
It must exist in the same namespace as the Travelpro Deployment.

## Deploy Travelpro to kind

The Deployment manifest references an immutable GitLab Registry image and the
`gitlab-registry` image-pull Secret:

```yaml
spec:
  imagePullSecrets:
    - name: gitlab-registry
  containers:
    - name: travelpro
      image: registry.gitlab.com/vaidy.shanmugam1/travelpro:<GitHub-commit-SHA>
      imagePullPolicy: IfNotPresent
```

Deploy and wait for readiness:

```bash
kubectl apply -f k8s/deployment.yaml
kubectl rollout status deployment/travelpro --timeout=120s
kubectl get pods -l app=travelpro -o wide
kubectl get service travelpro
```

Verify the deployed image:

```bash
kubectl get deployment travelpro \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Verify the application:

```bash
curl -I http://localhost:8080
```

Open <http://localhost:8080>. No `kubectl port-forward` process is required for
the application.

## Deploy an application update

After changing `index.html`, `styles.css`, or `Dockerfile`:

1. Commit and push the application change to GitHub.
2. Run the GitLab pipeline.
3. Confirm the new GitHub SHA tag in **Deploy > Container Registry**.
4. Update the `image` tag in `k8s/deployment.yaml` to that immutable SHA.
5. Apply and verify the Deployment.
6. Commit and push the manifest update to GitHub.

Example commands:

```bash
git add index.html styles.css Dockerfile
git commit -m "Update Travelpro application"
git push

# After GitLab publishes the new immutable tag, update deployment.yaml.
kubectl apply -f k8s/deployment.yaml
kubectl rollout status deployment/travelpro --timeout=120s
git add k8s/deployment.yaml
git commit -m "Deploy updated Travelpro image [skip ci]"
git push
```

Argo CD automation is intentionally deferred. After it is configured, Argo CD
will replace the manual `kubectl apply` step.

## Troubleshooting

Inspect the workload and recent events:

```bash
kubectl get pods -l app=travelpro -o wide
kubectl describe deployment travelpro
kubectl describe pod -l app=travelpro
kubectl logs deployment/travelpro
kubectl get events --sort-by=.metadata.creationTimestamp
```

Common image errors:

- `ErrImagePull`: check the registry image path and tag.
- `ImagePullBackOff`: check that the deploy token is active and the Secret is
  current.
- `secret "gitlab-registry" not found`: create the Secret in the Deployment's
  namespace.
- `manifest unknown`: confirm that the immutable tag exists in GitLab.

Check whether port 8080 is occupied:

```bash
lsof -nP -iTCP:8080 -sTCP:LISTEN
```

## Cleanup

Remove only Travelpro:

```bash
kubectl delete -f k8s/deployment.yaml
kubectl delete secret gitlab-registry
```

Remove the entire cluster and all of its workloads and Secrets:

```bash
kind delete cluster --name kind
```
