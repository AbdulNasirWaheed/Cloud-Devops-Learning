# 🚀 Secure Two-Tier Deployment on Kubernetes with ConfigMap & Secrets

A hands-on DevOps project deploying a **Flask + MySQL** two-tier architecture on a **KIND (Kubernetes IN Docker)** cluster running on an **AWS EC2** instance — using Kubernetes **Secrets** and **ConfigMaps** to manage credentials securely.

---

## 📌 Industry Use Case

A fintech startup is building a personal expense-tracking platform with a Flask web app (frontend + API) and a MySQL database (user & transaction data). The goal: a proof-of-concept on AWS that proves **scalability**, **high availability**, and **security**, while staying **cost-effective**.

---

## 🎯 Objective

Deploy a two-tier Flask + MySQL app on Kubernetes (KIND) in AWS EC2 with:

- 💰 **Cost-effectiveness** — KIND on EC2 to avoid EKS costs
- ⚡ **Rapid deployment** — CI/CD-ready structure with GitHub integration
- 🔐 **Security** — Credentials managed with Kubernetes Secrets, not hardcoded

---

## 🏗️ Architecture

```
                ┌─────────────────────┐
                │   AWS EC2 (Ubuntu)  │
                │  ┌───────────────┐  │
                │  │  KIND Cluster │  │
                │  │               │  │
                │  │  Control Plane│  │
                │  │  Worker Node 1│  │
                │  │  Worker Node 2│  │
                │  └───────────────┘  │
                └─────────────────────┘
                          │
        ┌─────────────────┴─────────────────┐
        │                                     │
┌───────▼────────┐                  ┌────────▼────────┐
│  flask-app NS   │                  │  mysql-db NS    │
│                 │                  │                 │
│  Flask Deploy   │ ───MySQL_HOST──▶ │  MySQL Deploy   │
│  Flask Service  │   (ClusterIP)    │  MySQL Service  │
│  (NodePort)     │                  │  (ClusterIP)    │
└─────────────────┘                  └─────────────────┘
```

Each layer (frontend/backend) is isolated into its own **namespace** for better management, security, and separation of concerns.

---

## 📁 Project Structure

```
two-tier-app/
│
├── app.py                     # Flask app
├── requirements.txt           # Python dependencies
├── Dockerfile                 # Docker image definition
├── .env                       # Environment variables (MySQL creds, DB name, etc.)
│
├── k8s-manifests/
│   ├── namespaces/
│   │   ├── flask-app.yaml
│   │   └── mysql-db.yaml
│   ├── flask-app/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── mysql-db/
│       ├── deployment.yaml
│       └── service.yaml
│
├── kind-config.yaml            # KIND cluster config
└── README.md                   # This file
```

---

## 🔑 Secrets vs ConfigMaps

| Feature       | ConfigMap (Non-sensitive)                          | Secret (Sensitive)                          |
|---------------|------------------------------------------------------|-----------------------------------------------|
| **Purpose**   | Store non-confidential config (DB name, app mode)   | Store sensitive info (passwords, API keys, TLS certs) |
| **Encoding**  | Plain text                                           | Base64-encoded                                 |
| **Use Case**  | App configuration, DB names, URLs                   | Credentials, tokens, certificates              |
| **Security**  | Not encrypted by default                            | Designed for confidential data                 |

**Best Practice:** Use `ConfigMap` for DB name/host/port. Use `Secret` for usernames/passwords.

### What is `secretKeyRef`?

`secretKeyRef` injects a specific key from a Kubernetes Secret into a Pod as an environment variable — keeping sensitive data out of YAML files and version control entirely.

```yaml
env:
  - name: MYSQL_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: MYSQL_ROOT_PASSWORD
```

✅ Security — sensitive data never touches your Git repo
✅ Separation of concerns — devs reference secrets without knowing their values
✅ Flexibility — update the Secret once, no Deployment edits needed
✅ Compliance — supports GDPR/ISO/HIPAA-style requirements

---

## 🛠️ Step-by-Step Setup

### 1️⃣ Launch EC2 Instance

- AMI: Ubuntu 22.04 LTS
- Instance type: `t2.medium` (2 vCPU, 4GB RAM)
- Security group: allow SSH (22), HTTP (80), Custom TCP 5000 (Flask), MySQL (3306 if needed)

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

### 3️⃣ Install KIND and kubectl

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl

# KIND
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### 4️⃣ Create the KIND Cluster

`kind-config.yaml`:

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

```bash
kind create cluster --name two-tier-cluster --config kind-config.yaml
kubectl cluster-info
kubectl get nodes
```

### 5️⃣ Create Namespaces

```bash
kubectl create namespace flask-app
kubectl create namespace mysql-db
```

### 6️⃣ Build & Push Docker Image

```bash
docker build -t <dockerhub-username>/flask-two-tier:latest .
docker push <dockerhub-username>/flask-two-tier:latest
```

### 7️⃣ Create ConfigMap & Secret

```bash
kubectl create configmap mysql-config \
  --from-env-file=.env \
  --namespace=mysql-db

kubectl create secret generic mysql-secret \
  --from-env-file=.env \
  --namespace=mysql-db
```

> ⚠️ **Important:** Secrets and ConfigMaps are namespace-scoped. If your Flask app (in `flask-app` namespace) also needs these values via `envFrom`, you must create matching copies in that namespace too.

### 8️⃣ Apply Manifests

```bash
kubectl apply -f k8s-manifests/ --recursive
```

### 9️⃣ Verify Deployment

```bash
kubectl get pods -n flask-app
kubectl get pods -n mysql-db
kubectl get svc -n flask-app
kubectl get svc -n mysql-db
```

### 🔟 Access the Application

```bash
kubectl port-forward svc/flask-service 5000:5000 -n flask-app --address 0.0.0.0
```

Then open: `http://<EC2_PUBLIC_IP>:5000`

---

## 🐛 Common Issues & Fixes

| Error | Cause | Fix |
|---|---|---|
| `did not find expected key` / `did not find expected '-' indicator` | YAML indentation mismatch | Ensure consistent 2-space indentation per nesting level; align list items exactly |
| `unknown field spec.template.spec.envFrom` | `envFrom`/`env` placed outside the container block | Nest under `containers[].envFrom`, not `spec.template.spec` |
| `MYSQL_USER="root" ... cannot be used for root user` | `MYSQL_USER=root` set alongside root password vars | Remove `MYSQL_USER`; only use `MYSQL_ROOT_PASSWORD` for root |
| `secret "mysql-secret" not found` (CreateContainerConfigError) | Secret/ConfigMap created in a different namespace | Create matching Secret/ConfigMap in the same namespace as the consuming Deployment |
| `pod is not running. Current status=Pending` | Pod stuck scheduling — often due to the above config errors | Run `kubectl describe pod <name> -n <namespace>` and check the **Events** section |

---

## ✅ Key Takeaways

- 💰 **Cost-Effective** — KIND on EC2 avoids EKS costs while simulating production
- 🔐 **Secure** — Secrets and ConfigMaps protect sensitive credentials and configs
- 🗂️ **Organized** — Namespaces separate app and database layers
- 📈 **Scalable** — Containerized workloads can be scaled horizontally
- 🔮 **Future-Ready** — Foundation for CI/CD, monitoring, and migration to EKS

---

## 🙏 Acknowledgements

This project was completed as a hands-on learning exercise, with guidance and structure from **Miles Academy** and **Hafiz Muhammad Umair Munir**.

---

## 📄 License

This project is for educational/learning purposes.
