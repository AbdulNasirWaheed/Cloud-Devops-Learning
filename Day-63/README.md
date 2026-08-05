# Two-Tier Flask App on Kubernetes (KIND) — AWS EC2 PoC

A cost-effective proof-of-concept for deploying a **two-tier application** (Flask frontend + MySQL backend) on a **local Kubernetes cluster using KIND (Kubernetes IN Docker)**, running entirely on a single AWS EC2 instance.

## 📌 Overview

This project demonstrates how a startup or small team can prototype a multi-tier, cloud-native application on Kubernetes **without** paying for a managed service like AWS EKS. KIND provides a lightweight, disposable cluster that's ideal for development, testing, and learning — before migrating to production-grade infrastructure.

**Stack:**
- **Frontend:** Flask web application
- **Backend:** MySQL database
- **Orchestration:** Kubernetes (via KIND)
- **Host:** Single Ubuntu EC2 instance (t2.medium)

## 🏗️ Project Structure

```
two-tier-app/
│
├── app.py                # Flask app
├── requirements.txt      # Python dependencies
├── Dockerfile            # Docker image for Flask app
│
k8s-manifests/
│
├── namespaces/
│   ├── flask-app.yaml
│   └── mysql-db.yaml
│
├── flask-app/
│   ├── deployment.yaml
│   └── service.yaml
│
└── mysql-db/
    ├── deployment.yaml
    └── service.yaml
│
├── kind-config.yaml      # KIND cluster config
│
└── README.md
```

## ✅ Prerequisites

- AWS account with permission to launch EC2 instances
- Ubuntu 22.04 LTS EC2 instance (t2.medium recommended: 2 vCPU / 4GB RAM)
- Security group allowing: SSH (22), HTTP (80), Flask (5000 / 30007), MySQL (3306, if needed)
- A Docker Hub account (to push/pull custom images)

## 🚀 Setup Guide

### 1. Launch EC2 Instance
Launch an Ubuntu 22.04 t2.medium instance, then connect:
```bash
ssh -i mykey.pem ubuntu@<EC2_PUBLIC_IP>
```

### 2. Install Docker
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
docker --version
```

Install MySQL client build dependencies:
```bash
sudo apt-get update
sudo apt-get install -y python3-dev default-libmysqlclient-dev build-essential pkg-config
```

### 3. Install kubectl and KIND

**kubectl:**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
kubectl version --client
```

**KIND:**
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

### 4. Clone This Repository
```bash
git clone https://github.com/Umair1012/two-tier-app.git
cd two-tier-app
```

> 💡 Always clone the repo **first** and run all subsequent commands from inside the project folder — this keeps your `kind-config.yaml` and manifests version-controlled and reproducible.

### 5. Create the KIND Cluster

The cluster config (`kind-config.yaml`) defines 1 control-plane node + 2 worker nodes, with port mappings for HTTP/HTTPS and the Flask NodePort:

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
      - containerPort: 30007
        hostPort: 30007
        protocol: TCP
  - role: worker
    image: kindest/node:v1.28.0
  - role: worker
    image: kindest/node:v1.28.0
```

Create the cluster from inside the cloned project folder:
```bash
kind create cluster --name two-tier-cluster --config kind-config.yaml
kubectl cluster-info
kubectl get nodes
```

### 6. Create Namespaces
```bash
kubectl create namespace flask-app
kubectl create namespace mysql-db
kubectl get ns
```

### 7. Build & Push the Flask Docker Image
```bash
cd app
docker build -t <dockerhub-username>/flask-two-tier:latest .
docker push <dockerhub-username>/flask-two-tier:latest
cd ..
```

(Optional) Pull and re-tag the official MySQL image:
```bash
docker pull mysql:8
docker tag mysql:8 <dockerhub-username>/mysql:8
docker push <dockerhub-username>/mysql:8
```

### 8. Apply Kubernetes Manifests
```bash
kubectl apply -f pod.yml
kubectl apply -f deployment.yml
kubectl apply -f service.yml
```

Check status:
```bash
kubectl get pods -n flask-app
kubectl get pods -n mysql-db
kubectl get svc -n flask-app
kubectl get svc -n mysql-db
```

### 9. Access the Application
```bash
kubectl port-forward svc/flask-service 5000:5000 -n flask-app
```

Then open in your browser:
```
http://<EC2_PUBLIC_IP>:5000
```

You should see the Flask app running and connected to MySQL. 🎉

## 🧹 Tearing Down

```bash
kind delete cluster --name two-tier-cluster
```

## 📖 Notes

- This setup is intended for **development, PoC, and learning purposes** — not production. MySQL root passwords are hardcoded in plain manifests here for simplicity; use Kubernetes Secrets in any real deployment.
- KIND clusters are ephemeral by design — re-running `kind create cluster` recreates everything from scratch, so keeping all manifests inside this repo (rather than ad-hoc files in `$HOME`) makes the environment fully reproducible.
- To move toward production, the natural next step is migrating this same manifest structure to a managed Kubernetes service such as **AWS EKS**.

## 🏁 Conclusion

This PoC shows how a small team can practice multi-tier Kubernetes deployment patterns — namespaces, deployments, services — at minimal cost, using a single EC2 instance and KIND, before committing to production-grade managed Kubernetes infrastructure.
