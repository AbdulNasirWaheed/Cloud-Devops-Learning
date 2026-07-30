# 🚀 Jenkins Declarative Pipeline with Shared Libraries for Multi-Environment Deployment

A production-style DevOps project demonstrating how to build a **clean, reusable, and maintainable Jenkins Declarative Pipeline** using **Jenkins Shared Libraries**.

Instead of duplicating pipeline code across projects, reusable Groovy functions are stored in a centralized Shared Library repository, making CI/CD pipelines easier to maintain and scale.

---

# 📌 Industry Scenario

You are a DevOps Engineer responsible for deploying a **three-tier application** across multiple environments:

* Development
* Staging
* Production

Each environment has its own EC2 instance and requires independent deployments while sharing the same CI/CD logic.

To reduce duplicated Jenkins code, Shared Libraries are used.

---

# 🏗 Architecture

```text
                GitHub Repository
                       │
               Push to Branch
                       │
                 Jenkins Controller
                       │
         Jenkins Shared Library
                       │
      ┌────────────────┼────────────────┐
      │                │                │
   Build Images     Push Images     Generate .env
      │                │                │
      └────────────────┼────────────────┘
                       │
                Docker Hub Registry
                       │
      ┌────────────────┼────────────────┐
      │                │                │
 Dev EC2          Staging EC2       Production EC2
      │                │                │
 Docker Compose   Docker Compose   Docker Compose
```

---

# ✨ Features

* Jenkins Declarative Pipeline
* Jenkins Shared Libraries
* Dockerized Frontend & Backend
* Docker Hub Integration
* Branch-Based Docker Image Tagging
* Environment-Specific Deployment
* Docker Compose Automation
* Automatic `.env` Generation
* Manual Approval for Staging & Production
* Image Cleanup
* Reusable Groovy Functions

---

# 🛠 Tech Stack

* Jenkins
* Jenkins Shared Libraries
* Groovy
* Docker
* Docker Compose
* Docker Hub
* AWS EC2
* Git
* GitHub
* React
* Node.js
* MongoDB

---

# 📁 Shared Library Structure

```text
jenkins-shared-lib/

├── vars/
│   ├── checkoutSource.groovy
│   ├── dockerLogin.groovy
│   ├── buildImages.groovy
│   ├── pushImages.groovy
│   ├── prepareEnvFile.groovy
│   ├── deployApp.groovy
│   └── cleanupImages.groovy
│
└── src/
```

---

# 🔁 Pipeline Flow

```text
Checkout Source
      │
Docker Login
      │
Build Docker Images
      │
Push Images to Docker Hub
      │
Generate .env
      │
Deploy with Docker Compose
      │
Cleanup Images
```

---

# 🚀 Pipeline Stages

* Checkout Source
* Docker Login
* Build Images
* Push Images
* Prepare Environment File
* Deploy Application
* Cleanup Images

---

# 🏷 Branch-Based Image Tagging

| Branch | Docker Tag        |
| ------ | ----------------- |
| dev    | dev-BUILD_NUMBER  |
| stg    | stg-BUILD_NUMBER  |
| prod   | prod-BUILD_NUMBER |

Example:

```text
username/backend:dev-25
username/frontend:dev-25
```

---

# 🔐 Deployment Strategy

## Development

* Automatic deployment

## Staging

* Manual approval required

## Production

* Manual approval required

---

# 📦 Shared Library Functions

### checkoutSource()

Checks out project source code.

### dockerLogin()

Authenticates with Docker Hub using Jenkins credentials.

### buildImages()

Builds frontend and backend Docker images.

### pushImages()

Pushes images to Docker Hub.

### prepareEnvFile()

Creates the `.env` file with image references.

### deployApp()

Deploys the application using Docker Compose.

### cleanupImages()

Removes local Docker images after deployment.

---

# 📂 Project Structure

```text
project/

├── backend/
├── frontend/
├── docker-compose.dev.yml
├── docker-compose.stg.yml
├── docker-compose.prod.yml
├── Jenkinsfile
└── README.md
```

---

# 📸 Screenshots

Add screenshots here:

* Jenkins Dashboard
* Successful Pipeline
* Shared Library Configuration
* Docker Hub Images
* Running Containers
* EC2 Deployment
* Application UI

---

# 🎯 Key Learning Outcomes

* Jenkins Shared Libraries
* Reusable CI/CD Pipelines
* Jenkins Declarative Pipelines
* Docker Automation
* Docker Hub Integration
* Multi-Environment Deployments
* Branch-Based Deployment Strategy
* Environment-Specific Docker Compose
* Production-Ready Pipeline Design

---

# 📚 Acknowledgements

Special thanks to **miseacademy** and **Hafiz Muhammad Umair Munir** for their outstanding DevOps training and practical, industry-oriented learning resources.

---

# ⭐ If you found this project helpful

Give this repository a ⭐ and feel free to fork it for your own learning!

Happy Learning! 🚀
