# ☸️ Introduction to Kubernetes and KIND Setup on AWS EC2

A beginner-friendly hands-on guide for introducing **Kubernetes orchestration** and creating a local Kubernetes cluster on an **AWS EC2 Ubuntu 22.04 LTS** instance using **KIND (Kubernetes IN Docker)**.

---

## 📌 Overview

Docker provides an efficient way to package and run applications as containers. However, managing multiple containers across microservices becomes increasingly difficult as applications grow.

**Kubernetes** provides container orchestration capabilities for:

* ⚡ Auto-scaling
* 🩺 Self-healing
* 🔎 Service discovery
* 🔄 Rolling updates and rollbacks
* 💾 Storage orchestration
* 🔐 Secrets management
* 🌐 Load balancing
* 🚀 Automated deployments
* 📈 High availability

This lab creates a Kubernetes cluster containing:

```text
AWS EC2
└── Ubuntu 22.04 LTS
    ├── Docker
    └── KIND Cluster
        ├── Control Plane
        ├── Worker 1
        ├── Worker 2
        └── Worker 3
```

---

## 🏢 Industry Use Case

A startup operating a microservices-based application may have:

```text
Frontend
React / Angular + NGINX
        │
        ▼
Backend Services
Node.js / Java / Python
        │
        ▼
Database
MySQL / PostgreSQL
```

As the number of containers increases, manual Docker management becomes difficult.

Kubernetes provides a centralized platform for deploying, scaling, networking, monitoring, and recovering containerized workloads.

### Docker Alone vs Kubernetes

| Docker                          | Kubernetes                       |
| ------------------------------- | -------------------------------- |
| Runs containers                 | Orchestrates containers          |
| Manual scaling                  | Automated scaling                |
| Manual recovery                 | Self-healing                     |
| Basic networking                | Service discovery                |
| Manual deployments              | Rolling deployments              |
| Individual container management | Cluster-wide workload management |

---

## ☸️ Kubernetes Architecture Overview

A Kubernetes cluster generally contains a **control plane** and **worker nodes**.

```text
                    Kubernetes Cluster
                           │
             ┌─────────────┴─────────────┐
             │                           │
      Control Plane                 Worker Nodes
             │                  ┌────────┼────────┐
      ┌──────┼──────┐            │        │        │
      │      │      │          Worker1  Worker2  Worker3
   API Server Scheduler
      │
 Controller Manager
      │
     etcd
```

### Control Plane Components

* **API Server** — exposes the Kubernetes API.
* **Scheduler** — determines where workloads should run.
* **Controller Manager** — maintains the desired cluster state.
* **etcd** — stores Kubernetes cluster state and configuration.

### Worker Node Components

* **kubelet** — communicates with the control plane and manages workloads.
* **kube-proxy** — provides networking functionality.
* **Container Runtime** — runs containers.

### Common CNI Plugins

Kubernetes networking can be implemented using CNI plugins such as:

* Flannel
* Calico
* Weave Net
* Cilium

---

# 🏗️ Cluster Types

### Single-Node Clusters

Useful for development and learning:

* KIND
* Minikube

### Multi-Node Self-Managed Clusters

Common tools include:

* kubeadm

### Managed Kubernetes

Cloud providers offer managed Kubernetes services:

* Amazon EKS
* Google Kubernetes Engine
* Azure Kubernetes Service

---

# ☁️ Prerequisites

## AWS EC2

Recommended configuration:

* **AMI:** Ubuntu 22.04 LTS
* **Instance Type:** `t2.medium` recommended
* **Storage:** At least 20 GB recommended
* **Security Group:** Allow SSH access on port `22`
* Allow application/ingress ports `80` and `443` when external access is required.

SSH into the EC2 instance:

```bash
ssh -i <key.pem> ubuntu@<EC2_PUBLIC_IP>
```

Update packages:

```bash
sudo apt update
sudo apt upgrade -y
```

---

# 🐳 Install Docker

Install Docker:

```bash
sudo apt install docker.io -y
```

Start Docker:

```bash
sudo systemctl start docker
```

Enable Docker at boot:

```bash
sudo systemctl enable docker
```

Verify the installation:

```bash
docker --version
```

Add the current user to the Docker group:

```bash
sudo usermod -aG docker $USER
```

Apply the group change:

```bash
newgrp docker
```

Verify Docker without `sudo`:

```bash
docker ps
```

---

# 🧩 Install KIND

Download KIND:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
```

Make KIND executable:

```bash
chmod +x ./kind
```

Move KIND into the system path:

```bash
sudo mv ./kind /usr/local/bin/kind
```

Verify the installation:

```bash
kind version
```

---

# ⚙️ Create KIND Configuration

Create the configuration file:

```bash
nano config.yml
```

Use the following configuration:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
    image: kindest/node:v1.28.0
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP

  - role: worker
    image: kindest/node:v1.28.0

  - role: worker
    image: kindest/node:v1.28.0

  - role: worker
    image: kindest/node:v1.28.0
```

Save the file and exit.

---

# 🚀 Create the Kubernetes Cluster

Create the cluster using the configuration:

```bash
kind create cluster --name devops-cluster --config config.yml
```

The resulting topology:

```text
devops-cluster
│
├── control-plane
│
├── worker
├── worker
└── worker
```

KIND uses Docker containers as Kubernetes nodes, making it possible to run a multi-node Kubernetes cluster on a single EC2 instance.

---

# 🌐 Why Map Ports 80/443?

The configuration maps:

```text
EC2 Host Port 80  → KIND Control Plane Port 80
EC2 Host Port 443 → KIND Control Plane Port 443
```

The mapping is placed on the **control-plane node** to provide a straightforward entry point for HTTP/HTTPS traffic into the KIND cluster.

This becomes particularly useful when testing:

* NGINX
* Ingress Controllers
* Web applications
* HTTP/HTTPS routing

The worker nodes do not require direct host-port mappings for this setup.

---

# 🔧 Install kubectl

Download the latest stable `kubectl`:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Install it:

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Verify:

```bash
kubectl version --client
```

---

# 🔍 Verify the Cluster

Check cluster information:

```bash
kubectl cluster-info --context kind-devops-cluster
```

Expected output should show Kubernetes control-plane and CoreDNS endpoints.

Check all nodes:

```bash
kubectl get nodes
```

Expected structure:

```text
NAME                             STATUS   ROLES           AGE   VERSION
devops-cluster-control-plane     Ready    control-plane   ...   v1.28.0
devops-cluster-worker            Ready    <none>          ...   v1.28.0
devops-cluster-worker2           Ready    <none>          ...   v1.28.0
devops-cluster-worker3           Ready    <none>          ...   v1.28.0
```

Check KIND clusters:

```bash
kind get clusters
```

Check Kubernetes namespaces:

```bash
kubectl get namespaces
```

---

# 🧪 Test Kubernetes Workloads

Deploy a simple NGINX application:

```bash
kubectl create deployment nginx --image=nginx
```

Check the deployment:

```bash
kubectl get deployments
```

Check pods:

```bash
kubectl get pods
```

Expose the deployment:

```bash
kubectl expose deployment nginx --port=80 --type=NodePort
```

Check the service:

```bash
kubectl get service nginx
```

---

# 🔄 KIND in a CI/CD Workflow

KIND can be useful for Kubernetes development and testing within CI/CD pipelines.

A typical workflow:

```text
GitHub
   │
   ▼
CI/CD Pipeline
   │
   ├── Build Application
   ├── Build Docker Image
   ├── Run Tests
   ├── Create KIND Cluster
   ├── Deploy Kubernetes Manifests
   └── Run Integration Tests
```

Production workloads generally use managed or production-grade Kubernetes clusters such as Amazon EKS rather than KIND.

---

# 💡 Tips & Notes

### Use KIND for Learning and Testing

KIND is excellent for:

* Kubernetes learning
* Local development
* CI testing
* Kubernetes manifest validation
* Application integration testing

### Use Managed Kubernetes for Production

Production environments can use services such as:

* Amazon EKS
* Google Kubernetes Engine
* Azure Kubernetes Service

### Recommended Next Steps

1. Deploy an NGINX application.
2. Create Kubernetes Deployments and Services.
3. Install an Ingress Controller.
4. Configure DNS and HTTPS.
5. Add ConfigMaps and Secrets.
6. Implement Kubernetes health checks.
7. Integrate Kubernetes with Jenkins or another CI/CD platform.
8. Explore Helm.
9. Add monitoring with Prometheus and Grafana.
10. Explore autoscaling with the Horizontal Pod Autoscaler.

---

# 🛠️ Troubleshooting

## Docker Permission Denied

Error:

```text
permission denied while trying to connect to the Docker daemon
```

Add the current user to the Docker group:

```bash
sudo usermod -aG docker $USER
```

Then reload the group:

```bash
newgrp docker
```

Verify:

```bash
docker ps
```

---

## KIND Image Download Failure

If the Kubernetes node image cannot be downloaded, verify internet connectivity:

```bash
curl -I https://kind.sigs.k8s.io
```

Pull the required image manually:

```bash
docker pull kindest/node:v1.28.0
```

Then recreate the cluster:

```bash
kind create cluster --name devops-cluster --config config.yml
```

---

## Port 80 or 443 Already in Use

Check which process is using the port:

```bash
sudo ss -ltnp | grep ':80'
```

```bash
sudo ss -ltnp | grep ':443'
```

Stop the conflicting service or change the `hostPort` values in `config.yml`.

---

## kubectl Context Issues

List available contexts:

```bash
kubectl config get-contexts
```

Switch to the KIND cluster:

```bash
kubectl config use-context kind-devops-cluster
```

Verify:

```bash
kubectl cluster-info --context kind-devops-cluster
```

---

## Cluster Creation Problems

List existing KIND clusters:

```bash
kind get clusters
```

Delete the existing cluster:

```bash
kind delete cluster --name devops-cluster
```

Create it again:

```bash
kind create cluster --name devops-cluster --config config.yml
```

---

# 📚 Key Kubernetes Concepts

| Concept       | Purpose                                 |
| ------------- | --------------------------------------- |
| Pod           | Smallest deployable Kubernetes workload |
| Deployment    | Manages replicated application Pods     |
| Service       | Provides stable network access to Pods  |
| Namespace     | Provides logical resource separation    |
| ConfigMap     | Stores non-sensitive configuration      |
| Secret        | Stores sensitive configuration          |
| Ingress       | Routes external HTTP/HTTPS traffic      |
| Node          | Machine running Kubernetes workloads    |
| Control Plane | Manages cluster state                   |
| kubelet       | Manages Pods on worker nodes            |
| kubectl       | CLI for interacting with Kubernetes     |

---

# 🎯 Learning Outcome

This lab demonstrates how Kubernetes can transform container management from individual Docker commands into **automated cluster orchestration**.

The KIND cluster provides a practical environment for understanding:

```text
Containers
    ↓
Kubernetes
    ↓
Pods
    ↓
Deployments
    ↓
Services
    ↓
Ingress
    ↓
CI/CD
    ↓
Scalable Applications 🚀
```

---

# 🙏 Credits

Thanks to **miseacademy and Hafiz Muhammad Umair Munir** for the learning resources and guidance.

---

## ⭐ Next Challenge

Build on this foundation by deploying a complete microservices application with:

```text
React / Angular
       ↓
NGINX / Ingress
       ↓
Node.js / Java / Python
       ↓
MySQL / PostgreSQL
       ↓
Kubernetes + CI/CD
```

**From running containers to orchestrating production-grade workloads. ☸️🚀**
