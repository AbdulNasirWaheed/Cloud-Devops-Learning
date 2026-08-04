# Flask Application on Kubernetes with KIND (AWS EC2)

A hands-on DevOps proof-of-concept demonstrating how to deploy a containerized Flask application on a Kubernetes cluster running via KIND (Kubernetes IN Docker), hosted on a single AWS EC2 instance.

## 📌 Project Overview

A growing startup needs to demonstrate Kubernetes orchestration for application deployment while keeping infrastructure costs low. This project simulates that scenario by deploying a simple Flask app on a lightweight, local Kubernetes environment — ideal for development and PoC work before scaling to a managed service like AWS EKS.

### Why KIND?
- Lightweight and easy to set up
- Perfect for development and proof-of-concept work
- Avoids the overhead of managing a full production Kubernetes cluster early on
- Great for testing deployment strategies before moving to production infrastructure

## 🎯 Objective

- Launch an Ubuntu EC2 instance (t2.medium)
- Install Docker and KIND
- Set up a Kubernetes cluster inside the EC2 instance
- Create and use a Kubernetes namespace for resource isolation
- Deploy a Flask application on the cluster
- Practice scaling, rolling updates, and rollbacks

## 📁 Project Structure

```
flask-app/
│
├── app.py                # Flask application
├── requirements.txt      # Python dependencies
├── Dockerfile            # Docker image definition
│
├── deployment.yaml       # Kubernetes Deployment manifest
├── service.yaml          # Kubernetes Service manifest
├── namespace.yaml        # Kubernetes Namespace manifest
│
├── kind-config.yaml      # KIND cluster configuration
│
└── README.md             # This file
```

## 🧠 Key Kubernetes Concepts Used

- **Namespace** — isolates resources (pods, services, deployments) so multiple apps can run in the same cluster without conflicts.
- **Selector, Labels & Template** — the selector tells a Deployment which pods to manage; the pod template defines how each pod should look, and its labels must match the selector.
- **Deployment** — manages long-running application pods (e.g., the Flask app running 24/7).
- **Job** *(not used here, but relevant)* — for one-off tasks that run to completion, like a database migration script.
- **DaemonSet** *(not used here, but relevant)* — ensures one pod runs on every node, commonly used for log collectors, monitoring agents, or networking components.

## 🚀 Step-by-Step Setup

### 1. Launch EC2 Instance
- AMI: Ubuntu 22.04 LTS
- Instance type: t2.medium (2 vCPU, 4GB RAM)
- Security group: allow SSH (22), HTTP (80), and Custom TCP (5000 for Flask)

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

### 3. Install kubectl and KIND
```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
kubectl version --client

# KIND
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

### 4. Create the KIND Cluster
`kind-config.yaml` defines a 1 control-plane + 2 worker node cluster with port mappings for 80, 443, and the Flask NodePort (30007):

```bash
kind create cluster --name flask-cluster --config kind-config.yaml
kubectl cluster-info
kubectl get nodes
```

### 5. Create the Kubernetes Namespace
```bash
kubectl apply -f namespace.yaml
kubectl get ns
```

### 6. Build and Push the Docker Image
```bash
docker login
docker build -t <dockerhub-username>/flask-kind:latest .
docker push <dockerhub-username>/flask-kind:latest
```

> **Tip:** If your image is only needed locally for KIND (no registry push required), load it directly into the cluster instead:
> ```bash
> kind load docker-image <dockerhub-username>/flask-kind:latest --name flask-cluster
> ```
> and set `imagePullPolicy: IfNotPresent` in your deployment manifest.

### 7. Deploy to Kubernetes
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get pods -n flask-namespace
kubectl get svc -n flask-namespace
```

### 8. Access the Application
```bash
kubectl port-forward --address 0.0.0.0 svc/flask-service 5000:5000 -n flask-namespace
```

Then open in your browser:
```
http://<EC2_PUBLIC_IP>:5000
```

Expected output:
```
Hello from Flask running on Kubernetes with KIND!
```

## 🔄 Scaling, Rolling Updates & Rollback

**Scale the deployment:**
```bash
kubectl scale deployment flask-deployment --replicas=5 -n flask-namespace
```

**Update the container image (rolling update):**
```bash
kubectl set image deployment/flask-deployment flask=<dockerhub-username>/flask-kind:v2 -n flask-namespace
kubectl rollout status deployment/flask-deployment -n flask-namespace
```

**Rollback if something goes wrong:**
```bash
kubectl rollout undo deployment/flask-deployment -n flask-namespace
```

## 🐛 Common Issues & Fixes

| Issue | Cause | Fix |
|---|---|---|
| `ImagePullBackOff` | Image tag typo or image not accessible to the cluster | Verify tag spelling; use `kind load docker-image` to load it locally |
| `error: unable to find container named "X"` | Container name in `kubectl set image` doesn't match the deployment's container name | Check `containers[].name` in `deployment.yaml` and match exactly |
| Port-forward exits unexpectedly | Underlying pod was terminated/replaced (e.g. during scaling or rollout) | Restart the port-forward command after the new pod is `Running` |
| Pods stuck in `Terminating` | Grace period / KIND container runtime delay | Wait ~30-60s; force delete with `--grace-period=0 --force` if truly stuck |

## ✅ Conclusion

This project demonstrates a complete, cost-effective workflow for practicing Kubernetes orchestration on a single EC2 instance — from cluster creation and namespace isolation to containerized deployment, scaling, and rolling updates. It provides a solid foundation before migrating to a production-grade managed service like **AWS EKS**.

## 🙏 Acknowledgements

Special thanks to **Miseacademy** and **Hafiz Muhammad Umair Munir** for the guidance throughout this learning project.
