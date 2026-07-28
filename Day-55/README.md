# 🚀 Three-Tier App CI/CD Pipeline (Docker + Jenkins + GHCR)

A complete CI/CD pipeline for a **React + Node.js + MongoDB** three-tier application, deployed to a single AWS EC2 instance using **Jenkins**, **Docker Compose**, and **GitHub Container Registry (GHCR)**.

Every push to `main` triggers an automated build → push → deploy cycle. No manual steps required. ⚡

---

## 🏢 Project Scenario

This project simulates a real-world DevOps setup at a startup managing a full-stack application in a single **Dev environment**:

- 📦 Source code hosted on **GitHub**
- 🐳 Docker images built and pushed to **GitHub Container Registry**
- 🔧 **Jenkins** automates the entire CI/CD workflow
- 🖥️ Everything runs on a single **Ubuntu EC2 instance**

---

## 🧰 Tech Stack

| Layer          | Technology            |
|----------------|------------------------|
| Frontend       | React                 |
| Backend        | Node.js / Express     |
| Database       | MongoDB               |
| Containerization | Docker + Docker Compose |
| CI/CD          | Jenkins                |
| Image Registry | GitHub Container Registry (GHCR) |
| Hosting        | AWS EC2 (Ubuntu)       |

---

## 📦 Port Mapping

| Service   | Container Port | Host Port |
|-----------|-----------------|-----------|
| Frontend  | 3000            | 3001      |
| Backend   | 5000            | 5001      |
| MongoDB   | 27017           | (internal only) |

---

## ⚙️ How the Pipeline Works

```
GitHub Push
     │
     ▼
GitHub Webhook triggers Jenkins
     │
     ▼
Jenkins clones the repo
     │
     ▼
Builds Docker images (frontend + backend)
     │
     ▼
Pushes images to GHCR
     │
     ▼
Updates docker-compose.yml with new image tags
     │
     ▼
Redeploys containers via Docker Compose
```

---

## 🛠 Setup Guide

### 1️⃣ Clone the Repo
```bash
git clone https://github.com/<your-username>/deploy-three-tier-app.git
cd deploy-three-tier-app
```

### 2️⃣ Launch an EC2 Instance
- OS: Ubuntu 22.04+
- Recommended: `t3.large` or larger (Docker builds need real RAM/CPU)
- Open inbound ports: `22`, `8080` (Jenkins), `3001` (frontend), `5001` (backend)

### 3️⃣ Install Jenkins
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

### 4️⃣ Install Docker & Docker Compose
```bash
sudo apt install -y docker.io docker-compose-plugin
sudo usermod -aG docker jenkins
sudo usermod -aG docker $USER
sudo systemctl restart jenkins
```

### 5️⃣ Configure Jenkins
- Access Jenkins at `http://<EC2_PUBLIC_IP>:8080`
- Unlock with: `sudo cat /var/lib/jenkins/secrets/initialAdminPassword`
- Install plugins: **Docker Pipeline**, **GitHub Integration**, **Credentials Binding**

### 6️⃣ Add GHCR Credentials to Jenkins
Jenkins → Manage Jenkins → Credentials → Global → Add Credentials
- Type: Username & Password
- ID: `ghcr`
- Username: your GitHub username
- Password: a GitHub Personal Access Token with `write:packages` scope

### 7️⃣ Set Up GitHub Webhook
GitHub → Repo → Settings → Webhooks → Add Webhook
- Payload URL: `http://<EC2_PUBLIC_IP>:8080/github-webhook/`
- Content type: `application/json`
- Trigger: **Push events**

### 8️⃣ Create the Jenkins Pipeline Job
Create a new **Pipeline** job pointing at this repo's `Jenkinsfile`.

---

## 🔑 Environment Variables

**`backend/.env`** *(never commit real secrets here — pass via `docker-compose.yml` instead)*
```
MONGO_URI=mongodb://mongodb:27017/three-tier-db
```

**`frontend/.env`** *(safe to commit — baked into the public JS bundle at build time)*
```
REACT_APP_API_BASE_URL=http://<EC2_PUBLIC_IP>:5001
```

> ⚠️ **Important:** React environment variables are baked into the bundle at `docker build` time, not read at container runtime. Any change to `frontend/.env` requires a fresh image build and redeploy.

---

## 🚀 Deploying

Simply push to `main`:
```bash
git add .
git commit -m "your change"
git push origin main
```
Jenkins picks it up automatically via the webhook, builds, pushes to GHCR, and redeploys.

---

## 🌐 Accessing the App

| Service   | URL |
|-----------|-----|
| Frontend  | `http://<EC2_PUBLIC_IP>:3001` |
| Backend   | `http://<EC2_PUBLIC_IP>:5001` |
| MongoDB   | internal only via `mongodb://mongodb:27017` |

---

## 🧠 Lessons Learned Building This

- 🔡 Docker image names **must be lowercase** — no exceptions
- 🧹 `.dockerignore` matters — excluding the wrong file can silently break your build
- 🏷️ Always pin a Docker Compose **project name** (`-p`) to avoid orphaned containers holding ports hostage
- 🔐 `sudo docker` and plain `docker` can have completely separate credential stores — add your user to the `docker` group
- ⚛️ React (CRA) environment variables are compiled at **build time**, not runtime — restarting a container never picks up a `.env` change
- 🌍 EC2 instances get a **new public IP** on every stop/start unless you attach an Elastic IP
- 🔍 Always check container logs (`docker logs <container>`) before assuming a networking issue

---

## 🙏 Credits

Built as part of a hands-on DevOps learning project. Special thanks to **MIS Academy** and **Hafiz Muhammad Umair Munir** for the guided project structure that made this real-world CI/CD practice possible.

---

## 📄 License

This project is for educational purposes.
