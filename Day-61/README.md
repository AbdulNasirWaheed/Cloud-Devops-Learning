# 🚀 Deploy Your First App on Kubernetes with KIND (Nginx Example)

A hands-on walkthrough for deploying, scaling, updating, and rolling back a containerized NGINX application on a local Kubernetes cluster running inside Docker (KIND), set up on an AWS EC2 instance.

> 📚 Learning credit: **MiseAcademy** and **Hafiz Muhammad Umair Munir**

---

## 📖 Table of Contents

- [Industry Scenario](#-industry-scenario--use-case)
- [Core Concepts](#-core-kubernetes-concepts)
- [Prerequisites](#-prerequisites)
- [EC2 Setup](#-1-setup-ec2-instance)
- [Install Docker, kubectl, KIND](#-2-install-docker-and-kind)
- [Create the KIND Cluster](#-3-create-the-kind-cluster)
- [Deploy NGINX](#-4-deploy-the-nginx-application)
- [Expose via Service](#-5-expose-deployment-as-a-service)
- [Scale the App](#-6-scale-the-application)
- [Rolling Updates & Rollbacks](#-7-rolling-updates--rollbacks)
- [Cleanup](#-cleanup)
- [Conclusion](#-conclusion)

---

## 🏢 Industry Scenario & Use Case

A startup building a microservices-based web app needs:
- 📈 Scalable deployments across multiple services
- 🌍 High availability for end-users
- ⚡ Rapid updates without downtime

**Challenge:** Manually managing containers with Docker doesn't scale and is error-prone.
**Solution:** Use **Kubernetes (K8s)** with **KIND** to orchestrate containers automatically.

---

## 🧩 Core Kubernetes Concepts

| Concept | What it does |
|---|---|
| **Deployment** | Manages replicas of pods, ensures desired state (replica count, version), enables auto-scaling, rolling updates, and rollbacks |
| **Pod** | Smallest deployable unit; runs one or more containers; ephemeral (recreated automatically by a Deployment) |
| **Service** | Stable network entry point (IP/DNS) to a set of pods |

### 🔌 Service Types

| Type | Description | Analogy |
|---|---|---|
| **ClusterIP** | Internal-only access (default) | ☎️ Office phone extension |
| **NodePort** | Exposes a port on every node for external access | 📞 Direct office number |
| **LoadBalancer** | Integrates with cloud load balancers (AWS/GCP/Azure) | 🧑‍💼 Receptionist routing calls |

### 🔄 Rolling Updates & Rollbacks

- **Rolling Update** — gradually replaces pods with a new version, zero downtime (e.g. `nginx:1.20` → `nginx:1.22`)
- **Rollback** — instantly reverts to the previous working version if an update fails

---

## ✅ Prerequisites

- AWS account with permission to launch EC2 instances
- Basic familiarity with the Linux command line
- SSH key pair for EC2 access

---

## 🖥️ 1. Setup EC2 Instance

- **OS:** Ubuntu 22.04 LTS
- **Instance type:** t2.medium
- **Security group:** Allow SSH (port 22); optionally HTTP/HTTPS

```bash
ssh -i path/to/key.pem ubuntu@<EC2_PUBLIC_IP>
sudo apt update && sudo apt upgrade -y
```

---

## 🐳 2. Install Docker and KIND

### Install Docker
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker
docker --version
```

### Install kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
kubectl version --client
```

### Install KIND
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.21.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

---

## ☸️ 3. Create the KIND Cluster

`config.yml` — 1 control-plane node + 3 worker nodes, with ports 80/443 mapped:

```yaml
# config.yml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  # Control plane node
  - role: control-plane
    image: kindest/node:v1.28.0
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP

  # Worker nodes
  - role: worker
    image: kindest/node:v1.28.0
  - role: worker
    image: kindest/node:v1.28.0
  - role: worker
    image: kindest/node:v1.28.0
```

```bash
# Create cluster
kind create cluster --name nginx-cluster --config config.yml

# Verify
kubectl cluster-info --context kind-nginx-cluster
kubectl get nodes

# Delete cluster (if needed)
kind delete cluster --name nginx-cluster
```

---

## 📦 4. Deploy the NGINX Application

`deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      containers:
      - name: hello-container
        image: nginx:latest
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods
```

> 🧹 To fully reset Docker before recreating the cluster:
> ```bash
> kind delete cluster --name nginx-cluster
> docker system prune -af
> ```

---

## 🌐 5. Expose Deployment as a Service

`service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  selector:
    app: hello-app
  type: NodePort
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30007
```

```bash
kubectl apply -f service.yaml
kubectl get svc

# Access the app (binds to all network interfaces, not just localhost)
kubectl port-forward --address 0.0.0.0 svc/hello-service 8080:80
```

Open **http://localhost:8080** in a browser. 🎉

---

## 📈 6. Scale the Application

```bash
kubectl scale deployment hello-deployment --replicas=5
kubectl get pods
```

---

## 🔄 7. Rolling Updates & Rollbacks

**Update the container image:**
```bash
kubectl set image deployment/hello-deployment hello-container=nginx:alpine
kubectl rollout status deployment/hello-deployment
```

**Roll back if something breaks:**
```bash
kubectl rollout undo deployment/hello-deployment
```

**Check history:**
```bash
kubectl rollout history deployment/hello-deployment
```

> ⚠️ **Note:** If a Deployment was created with `kubectl apply`, rolling back via `kubectl rollout undo` won't update the `kubectl.kubernetes.io/last-applied-configuration` annotation. This can cause unexpected diffs on future `kubectl apply` runs — prefer re-applying your source YAML when possible.

---

## 🧹 Cleanup

```bash
kind delete cluster --name nginx-cluster
docker system prune -af
```

---

## 🏁 Conclusion

This project covers:

- 🏢 The industry relevance of Kubernetes for startups managing microservices
- 🧩 Core concepts: Deployment, Pod, Service, Rolling Updates & Rollbacks
- ☁️ Cloud setup: launching an EC2 instance and installing Docker
- ☸️ Local cluster setup with KIND for testing and development
- 🛠️ Hands-on deployment of an NGINX app — with scaling, rolling updates, and rollback

**✅ Outcome:** A fully functional local Kubernetes environment ready to deploy, scale, and manage containerized applications like a real-world DevOps engineer.

---

### 🙏 Credits

Tutorial content and guidance by **MiseAcademy** and **Hafiz Muhammad Umair Munir**.
