# 🚀 Secure Two-Tier Deployment: Flask + MySQL on Kubernetes (KIND)

> A hands-on DevOps PoC demonstrating how to deploy a scalable, secure, two-tier application (Flask + MySQL) on a self-managed Kubernetes cluster using **KIND (Kubernetes IN Docker)** on an AWS EC2 instance — avoiding the cost of a managed service like EKS.

📚 Learned via **MiseAcademy**, guided by **Hafiz Muhammad Umair Munir** 🙏

---

## 📌 Industry Use Case

A fintech startup is building a personal expense tracking platform with:
- 🐍 A **Flask** web app (frontend + API)
- 🗄️ A **MySQL** database (user & transaction data)

**Goal:** Build a Proof of Concept (PoC) on AWS that proves scalability, high availability, and security — while staying cost-effective.

---

## 🎯 Objective

Deploy a two-tier Flask + MySQL app on Kubernetes (KIND) on AWS EC2 with:

| Requirement | Solution |
|---|---|
| 💰 Cost-effectiveness | KIND on EC2 (avoids EKS costs) |
| ⚡ Rapid Deployment | CI/CD with GitHub integration |
| 🔐 Security | Credentials managed via Kubernetes Secrets |

---

## 🛠️ DevOps Engineer Responsibilities

- **Cluster Setup** — Provision an AWS EC2 instance and configure KIND to simulate a Kubernetes cluster
- **Application Deployment** — Write Kubernetes manifests (Deployments, Services, Secrets, PersistentVolumes) for Flask & MySQL
- **Scalability & Monitoring** — Implement Horizontal Pod Autoscaling (HPA) and readiness/liveness probes

---

## 🏗️ Architecture

- **Frontend/API:** Flask web application 🐍
- **Backend:** MySQL database 🗄️
- **Orchestration:** Kubernetes (via KIND) ☸️
- **Host:** AWS EC2 (Ubuntu 22.04, t2.medium) ☁️

---

## 📁 Project Structure

```
two-tier-app/
│
├── app.py                # Flask app
├── requirements.txt      # Dependencies
├── Dockerfile             # Docker image
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
├── kind-config.yaml       # KIND cluster config
│
└── README.md               # Documentation / setup steps
```

---

## 🔗 Source Code

Two-Tier Flask App: `https://github.com/Umair1012/two-tier-app.git`

---

## 🚦 Step-by-Step Implementation

### 1️⃣ Setup EC2 Instance (Ubuntu t2.medium)

- Login to **AWS Console → EC2**
- Launch Instance:
  - Name: `kind-two-tier-ec2`
  - AMI: `Ubuntu 22.04 LTS`
  - Type: `t2.medium` (2 vCPU, 4GB RAM)
  - Security Group: Allow SSH (22), HTTP (80), Custom TCP 5000 (Flask), MySQL (3306)

```bash
ssh -i mykey.pem ubuntu@<EC2_PUBLIC_IP>
```

### 2️⃣ Install Docker

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
docker --version
```

Install MySQL client dependencies:

```bash
sudo apt-get update
sudo apt-get install -y python3-dev default-libmysqlclient-dev build-essential pkg-config
```

### 3️⃣ Install KIND and kubectl

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

### 4️⃣ Create KIND Cluster Config

`kind-config.yaml` — 1 control-plane node + 2 worker nodes with port mappings:

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
      - containerPort: 30007     # Flask NodePort
        hostPort: 30007
        protocol: TCP

  - role: worker
    image: kindest/node:v1.28.0

  - role: worker
    image: kindest/node:v1.28.0
```

Create the cluster:

```bash
kind create cluster --name two-tier-cluster --config kind-config.yaml
kubectl cluster-info
kubectl get nodes
```

### 5️⃣ Create Namespaces

```bash
kubectl create namespace flask-app
kubectl create namespace mysql-db
kubectl get ns
```

### 6️⃣ Clone the Application

```bash
git clone https://github.com/Umair1012/two-tier-app.git
cd two-tier-app
```

Repo contains:
- `app/` → Flask frontend
- `mysql/` → MySQL backend

### 7️⃣ Build & Push Docker Images

**Flask image** (`Dockerfile`):

```dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    gcc \
    default-libmysqlclient-dev \
    pkg-config \
    python3-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

```bash
cd app
docker build -t <dockerhub-username>/flask-two-tier:latest .
docker push <dockerhub-username>/flask-two-tier:latest
cd ..
```

**MySQL image** (optional, official image works fine):

```bash
docker pull mysql:8
docker tag mysql:8 <dockerhub-username>/mysql:8
docker push <dockerhub-username>/mysql:8
```

### 8️⃣ Kubernetes Manifests

<details>
<summary>📄 Pod Manifests (<code>pod.yml</code>)</summary>

```yaml
# MySQL Pod
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
  namespace: mysql-db
spec:
  containers:
  - name: mysql
    image: mysql:8
    env:
      - name: MYSQL_ROOT_PASSWORD
        value: rootpassword
    ports:
      - containerPort: 3306
---
# Flask Pod
apiVersion: v1
kind: Pod
metadata:
  name: flask-pod
  namespace: flask-app
spec:
  containers:
  - name: flask
    image: <dockerhub-username>/flask-two-tier:latest
    ports:
      - containerPort: 5000
```

</details>

<details>
<summary>📄 Deployment Manifests (<code>deployment.yml</code>)</summary>

```yaml
# MySQL Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
  namespace: mysql-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        env:
          - name: MYSQL_ROOT_PASSWORD
            value: rootpassword
        ports:
        - containerPort: 3306
---
# Flask Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-deployment
  namespace: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask
  template:
    metadata:
      labels:
        app: flask
    spec:
      containers:
      - name: flask
        image: <dockerhub-username>/flask-two-tier:latest
        ports:
        - containerPort: 5000
```

</details>

<details>
<summary>📄 Service Manifests (<code>service.yml</code>)</summary>

```yaml
# MySQL Service
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: mysql-db
spec:
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
  type: ClusterIP
---
# Flask Service
apiVersion: v1
kind: Service
metadata:
  name: flask-service
  namespace: flask-app
spec:
  selector:
    app: flask
  ports:
    - port: 5000
      targetPort: 5000
      nodePort: 30007
  type: NodePort
```

</details>

### 9️⃣ Apply Kubernetes Resources

```bash
kubectl apply -f pod.yml
kubectl apply -f deployment.yml
kubectl apply -f service.yml

kubectl get pods -n flask-app
kubectl get pods -n mysql-db
kubectl get svc -n flask-app
kubectl get svc -n mysql-db
```

### 🔟 Access the Flask Application

```bash
kubectl port-forward svc/flask-service 5000:5000 -n flask-app
```

Open in browser: `http://<EC2_PUBLIC_IP>:5000` 🌐

You should see your two-tier Flask application connected to MySQL! ✅

---

## ✅ Conclusion

This documentation demonstrated how a DevOps engineer can:

- 🖥️ Launch an Ubuntu EC2 instance
- 🐳 Install Docker and KIND for a Kubernetes PoC
- ☸️ Create a KIND multi-node cluster
- 🗂️ Organize resources with namespaces (`flask-app`, `mysql-db`)
- 🚀 Deploy a two-tier Flask application with separate pods, deployments, and services
- 🔍 Validate the deployment and access the application

This PoC provides a **cost-effective and secure** way to practice multi-tier Kubernetes deployments — enabling startups to experiment with CI/CD, scaling, and container orchestration before migrating to production-grade managed services like **AWS EKS**.

---

## 🙏 Credits

Big thanks to **MiseAcademy** and **Hafiz Muhammad Umair Munir** for the guidance on this project! 🎓

---

## 🏷️ Tags

`#DevOps` `#Kubernetes` `#AWS` `#Docker` `#Flask` `#MySQL` `#KIND` `#CloudComputing` `#CI_CD`
