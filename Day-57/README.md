# 🧱 Three-Tier App — Multibranch CI/CD Pipeline

Automated CI/CD pipeline for a three-tier web application (**React + Node.js + MongoDB**), built with **Jenkins Multibranch Pipeline**, **Docker Compose**, and deployed across **Development**, **Staging**, and **Production** environments on **AWS EC2**.

---

## 📖 Overview

This project simulates a real-world DevOps workflow at a fast-growing tech company. Every Git branch maps to a live environment, and every push triggers a fully automated build → tag → push → deploy pipeline — no manual intervention required (aside from approval gates for staging/production).

```
Git Push → GitHub Webhook → Jenkins Multibranch Pipeline → Build & Push Images → Deploy via Docker Compose
```

---

## 🏗️ Architecture

| Component            | Technology                          |
|----------------------|--------------------------------------|
| Frontend             | React                                |
| Backend              | Node.js                              |
| Database             | MongoDB                              |
| CI/CD Engine         | Jenkins (Multibranch Pipeline)       |
| Containerization     | Docker + Docker Compose              |
| Image Registries     | AWS ECR · Docker Hub · GHCR          |
| Deployment Server    | Ubuntu EC2                            |
| Secrets Management   | Jenkins Credentials Store            |

---

## 🌱 Branch → Environment Mapping

| Branch | Environment  | URL Example            | Image Tag              |
|--------|-------------|--------------------------|-------------------------|
| `dev`  | Development | `http://ec2-ip:3001`     | `dev-BUILD_NUMBER`      |
| `stg`  | Staging     | `http://ec2-ip:4001`     | `stg-BUILD_NUMBER`      |
| `prod` | Production  | `http://ec2-ip:5001`     | `prod-BUILD_NUMBER`     |

> Staging and Production deployments require **manual approval** in Jenkins before rolling out.

---

## 📦 Port Mapping

| Service   | Container Port | Host Port |
|-----------|-----------------|-----------|
| Frontend  | 3000            | 3000      |
| Backend   | 5000            | 5000      |
| Database  | 27017           | 27017     |

---

## 🛠️ Setup Guide

### 1. Clone the Repository
```bash
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>
```

### 2. Launch an EC2 Instance
- OS: Ubuntu 22.04
- Instance type: `t2.micro` (or larger)
- Open inbound ports: `22, 8080, 3001, 5001, 27017`

```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

### 3. Install Jenkins
```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install -y jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

### 4. Install Docker, Docker Compose & AWS CLI
```bash
sudo apt install -y docker.io docker-compose
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# AWS CLI v2
sudo apt install unzip curl -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

### 5. Configure AWS CLI
```bash
aws configure
```
Provide your IAM user's Access Key ID, Secret Access Key, region (e.g. `us-east-1`), and output format (`json`). The IAM user needs ECR push/pull permissions.

### 6. Create a Jenkins Multibranch Pipeline
1. Jenkins dashboard → **New Item** → **Multibranch Pipeline**
2. Configure **Branch Sources** with your GitHub repo URL + credentials (GitHub token)
3. Set scan period (e.g. every 1 min) or use a webhook trigger
4. In GitHub → **Settings → Webhooks**, add: `http://<jenkins-url>/github-webhook/`

### 7. Add the `Jenkinsfile`
Place a `Jenkinsfile` at the repo root that:
- Checks out source
- Logs into Docker Hub
- Builds & tags `frontend`/`backend` images per branch (`dev-`, `stg-`, `prod-` + build number)
- Pushes images to the registry
- Writes a `.env` file for Docker Compose
- Requires manual approval before deploying to `stg`/`prod`
- Runs `docker compose up -d --remove-orphans` to deploy
- Cleans up local images after deployment

### 8. Create Environment-Specific `docker-compose` Files
| File                        | Frontend Port | Backend Port |
|------------------------------|---------------|--------------|
| `docker-compose.dev.yml`     | 3001          | 5001         |
| `docker-compose.stg.yml`     | 4001          | 6001         |
| `docker-compose.prod.yml`    | 80            | 443          |

---

## 🔐 Security Notes
- Store Docker Hub, AWS, and GitHub credentials in **Jenkins Credentials Store** — never hardcode secrets in the `Jenkinsfile` or `docker-compose.yml`.
- Restrict EC2 security group inbound rules to only the ports actually needed publicly.
- Use manual approval gates for `stg` and `prod` to prevent accidental production deploys.

---

## ✅ Key Benefits

- Fully automated builds and deployments across dev, staging, and production
- Redundant, multi-registry image storage (Docker Hub, AWS ECR, GHCR)
- Branch-based image tagging for easy rollback and version tracking
- Scalable foundation — can be extended to Amazon ECS or Kubernetes

---

## 🙏 Credits

Learned and built as part of a hands-on DevOps course by **MiseAcademy**, guided by **Hafiz Muhammad Umair Munir**.

---

## 📄 License

This project is open-sourced for learning purposes. Feel free to fork, adapt, and build on it.
