# Kind Single-Node Kubernetes Setup on Ubuntu

This guide installs:

* Docker Engine (official Docker packages)
* Docker Compose v2
* kubectl
* Kind (Kubernetes in Docker)
* A single-node Kubernetes cluster for local development and testing

This setup provides a lightweight local Kubernetes environment using upstream Kubernetes components and is suitable for application development, Helm testing, CI/CD validation, and Kubernetes learning.

---

# Prerequisites

* Ubuntu 22.04 LTS or newer
* Internet access
* User with sudo privileges

---

# 1. Install Docker Engine and Docker Compose

## Remove Old Docker Packages (Optional)

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt-get remove -y $pkg
done
```

## Configure Docker's Official Repository

Update package lists and install prerequisites:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
```

Create the keyring directory:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Download Docker's GPG key:

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
```

Set permissions:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker's repository:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update package indexes:

```bash
sudo apt-get update
```

## Install Docker

```bash
sudo apt-get install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

## Verify Docker Installation

```bash
docker --version
```

Expected output:

```text
Docker version xx.x.x
```

Test Docker:

```bash
sudo docker run hello-world
```

## Enable Docker for Non-Root Users

```bash
sudo usermod -aG docker $USER
```

Apply the group membership immediately:

```bash
newgrp docker
```

Verify:

```bash
docker run hello-world
```

---

# 2. Verify Docker Compose

Docker Compose v2 is installed as a Docker plugin.

Check the version:

```bash
docker compose version
```

Expected output:

```text
Docker Compose version v2.x.x
```

## Test Docker Compose

Create a test directory:

```bash
mkdir ~/compose-test
cd ~/compose-test
```

Create a file named `compose.yaml`:

```yaml
services:
  nginx:
    image: nginx:latest
    ports:
      - "8080:80"
```

Start the container:

```bash
docker compose up -d
```

Verify:

```bash
docker compose ps
```

Open:

```text
http://localhost:8080
```

You should see the NGINX welcome page.

Stop and remove the container:

```bash
docker compose down
```

---

# 3. Install kubectl

Download the latest stable version:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Install:

```bash
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Verify:

```bash
kubectl version --client
```

Expected output:

```text
Client Version: v1.xx.x
```

---

# 4. Install Kind

Download Kind:

```bash
curl -Lo kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
```

Install:

```bash
chmod +x kind
sudo mv kind /usr/local/bin/
```

Verify:

```bash
kind version
```

Expected output:

```text
kind version x.x.x
```

# 5. Install ArgoCD

## Create installation directory

```bash
mkdir -p argocd-install
cd argocd-install
```

## Configuration File

Create a file called kustomization.yaml
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: argocd
resources:
- https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Deploy ArgoCD

```
kubectl create namespace argocd

kubectl apply --server-side --force-conflicts -k .

kubectl wait --for=condition=available --timeout=600s deployment/argocd-server -n argocd
kubectl wait --for=condition=available --timeout=600s deployment/argocd-repo-server -n argocd
kubectl wait --for=condition=available --timeout=600s deployment/argocd-application-controller -n argocd

kubectl get pods -n argocd

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

kubectl port-forward svc/argocd-server -n argocd 8080:443
```

We should now be able to login using admin / <password from last command>

Location: https://localhost:8080

Reset the password !




# 6. Install Helm

Helm is the package manager for Kubernetes and is commonly used to deploy applications, ingress controllers, monitoring stacks, databases, and other services.

## Install Helm

Download and run the official installation script:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4

chmod 700 get_helm.sh

./get_helm.sh
```



## Verify Installation

```bash
helm version
```

Expected output:

```text
version.BuildInfo{
  Version:"v4.x.x",
  ...
}
```



# 7. Create a Kubernetes Cluster with Control Plane and Worker Node

Create a cluster file, kind-dev.yaml:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
  - role: worker
```

Create the nodes

```bash
kind create cluster --name dev --config kind-dev.yaml
```

Verify cluster creation:

```bash
kubectl cluster-info
```

List nodes:

```bash
kubectl get nodes
```

Expected output:

```text
NAME                STATUS   ROLES           AGE
dev-control-plane   Ready    control-plane   1m
```

---

Verify Cluster Health

Check all system pods:

```bash
kubectl get pods -A
```

You should see pods in namespaces such as:

```text
kube-system
local-path-storage
```

All pods should eventually reach the `Running` state.













# Install the Argo CD CLI

Download the CLI:

```bash
VERSION=$(curl -s https://api.github.com/repos/argoproj/argo-cd/releases/latest | grep tag_name | cut -d '"' -f4)

curl -sSL -o argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
```

Install:

```bash
chmod +x argocd

sudo mv argocd /usr/local/bin/
```

Verify:

```bash
argocd version --client
```

---

# Log In Using the CLI

In a separate terminal:

```bash
argocd login localhost:8080 \
  --username admin \
  --password <password> \
  --insecure
```

Verify:

```bash
argocd account get-user-info
```

---

# Create a GitOps Application

Assume you have a Git repository:

```text
https://github.com/myorg/gitops
```

containing:

```text
apps/
└── nginx/
    ├── deployment.yaml
    ├── service.yaml
    └── namespace.yaml
```

Create an Argo CD application:

```bash
argocd app create nginx \
  --repo https://github.com/myorg/gitops.git \
  --path apps/nginx \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

Synchronize:

```bash
argocd app sync nginx
```

Check status:

```bash
argocd app get nginx
```

---

# Enable Automatic Synchronization

Enable self-healing and automatic deployment:

```bash
argocd app set nginx \
  --sync-policy automated \
  --self-heal
```

Verify:

```bash
argocd app get nginx
```

You should see:

```text
Sync Policy: Automated
```

---

# Recommended Repository Structure

For GitOps with GitHub and GHCR:

```text
github.com/myorg/myapp
├── application source
├── Dockerfile
└── GitHub Actions

github.com/myorg/gitops
└── environments
    ├── dev
    ├── test
    └── prod
```

The application repository:

1. Builds containers.
2. Pushes images to GHCR.
3. Updates Helm values or manifests in the GitOps repository.

Argo CD watches the GitOps repository and deploys changes automatically.

---

# Common Argo CD Commands

List applications:

```bash
argocd app list
```

Get application details:

```bash
argocd app get <app-name>
```

Synchronize an application:

```bash
argocd app sync <app-name>
```

Refresh application status:

```bash
argocd app refresh <app-name>
```

Delete an application:

```bash
argocd app delete <app-name>
```

---

# Uninstall Argo CD

Remove the Helm release:

```bash
helm uninstall argocd -n argocd
```

Delete the namespace:

```bash
kubectl delete namespace argocd
```




---

# 8. Access the Application

Port-forward the service:

```bash
kubectl port-forward service/nginx 8080:80
```

Open:

```text
http://localhost:8080
```

You should see the NGINX welcome page.

Press `Ctrl+C` to stop port forwarding.

---

# 9. Useful Kubernetes Commands

Current context:

```bash
kubectl config current-context
```

List nodes:

```bash
kubectl get nodes
```

List all pods:

```bash
kubectl get pods -A
```

Describe a pod:

```bash
kubectl describe pod <pod-name>
```

View logs:

```bash
kubectl logs <pod-name>
```

List services:

```bash
kubectl get svc
```

List deployments:

```bash
kubectl get deployments
```

---

# 10. Useful Kind Commands

List clusters:

```bash
kind get clusters
```

Export cluster kubeconfig:

```bash
kind export kubeconfig --name dev
```

Delete the cluster:

```bash
kind delete cluster --name dev
```

Recreate the cluster:

```bash
kind create cluster \
  --name dev \
  --image kindest/node:v1.33.1
```

---

# Common Docker Compose Commands

Start services:

```bash
docker compose up -d
```

View running services:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs -f
```

Restart services:

```bash
docker compose restart
```

Stop services:

```bash
docker compose stop
```

Remove services and networks:

```bash
docker compose down
```

---

# Cleanup

Delete the Kubernetes cluster:

```bash
kind delete cluster --name dev
```

Remove test Compose files:

```bash
rm -rf ~/compose-test
```

---

# Next Steps

To make the local cluster more closely resemble EKS or GKE, consider installing:

* Helm
* NGINX Ingress Controller
* Cilium
* MetalLB
* cert-manager

These can be added later without recreating the cluster.
