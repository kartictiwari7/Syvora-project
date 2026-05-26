# 🚀  DevOps Project — Node.js Web Server + CI/CD + Kubernetes + Monitoring

## 👤 Author
**Name:** Kartikey Tiwari  
**Role:** DevOps Engineer  
**Date:** November 2025  

---

## 📘 Overview
This project demonstrates the complete DevOps lifecycle — from code to deployment to monitoring — using modern cloud-native tools.  
It includes:
- A **Node.js backend** with full **CRUD APIs**
- A **database (PostgreSQL)** connected via Docker Compose
- Automated **CI/CD pipeline** that builds and pushes Docker images
- **Kubernetes deployment** using Helm and Terraform
- **Prometheus** monitoring and **alerting**

This README explains every step to let the reviewer **build, deploy, and monitor** the system with minimal effort.

---

## 🧱 Project Architecture

project-root/
│
├── backend/                     # Node.js API source
│   ├── index.js
│   ├── db.js
│   ├── package.json
│   ├── Dockerfile
│   └── .env.example
│
├── docker-compose.yml            # Runs App + Database locally
│
├── .github/workflows/ci.yml      # CI pipeline (GitHub Actions)
│
├── helm/                         # Helm Chart for Kubernetes deployment
│   ├── templates/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   └── values.yaml
│
├── terraform/                    # Infrastructure setup (optional)
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
│
├── monitoring/
│   ├── servicemonitor.yaml
│   ├── prometheus-rules.yaml
│   └── grafana-dashboard.json
│
└── README.md

---

## 🧠 Features Implemented

| Category | Feature | Description |
|-----------|----------|-------------|
| **Backend** | CRUD APIs | Create, Read, Update, Delete user data |
| **Database** | MongoDB | Persistent data storage |
| **Dockerization** | `Dockerfile` + `docker-compose.yml` | App + DB run locally in containers |
| **CI/CD** | GitHub Actions | Auto build & push image to Docker Hub on main branch push |
| **Infrastructure** | Terraform | Optional IaC setup for Kubernetes cluster |
| **Deployment** | Helm Charts | Deploy Node.js backend to Kubernetes |
| **Monitoring** | Prometheus + Alertmanager | Collects metrics + triggers alerts |
| **Alerts** | Latency & Uptime | Warns if response > threshold or service down |

---

## ⚙️ 1. Web Server Setup

### **Backend**
A simple **Express.js** app with CRUD APIs for a `users` table.

Example endpoints:
GET    /api/users
POST   /api/users
PUT    /api/users/:id
DELETE /api/users/:id

Run locally:
cd backend
npm install
npm start

### **Database**
Using MongoDB via Docker Compose.

docker-compose up -d

Access:
- API → http://localhost:5000
- Database → localhost:27017

---

## 🐳 2. Dockerization

### **Dockerfile**
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["npm", "start"]

### **docker-compose.yml**
version: "3.8"

services:
  mongo:
    image: mongo:7.0
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: "${MONGO_INITDB_ROOT_USERNAME}"
      MONGO_INITDB_ROOT_PASSWORD: "${MONGO_INITDB_ROOT_PASSWORD}"
      MONGO_INITDB_DATABASE: "${MONGO_INITDB_DATABASE}"
    volumes:
      - mongo-data:/data/db
    ports:
      - "27017:27017"
    healthcheck:
      test: ["CMD", "mongo", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    depends_on:
      - mongo
    environment:
      # Backend reads the connection string from process.env.MONGO_URI
      MONGO_URI: "${MONGO_URI}"
      PORT: "${PORT:-5000}"
      NODE_ENV: "${NODE_ENV:-production}"
    ports:
      - "5000:5000"
    restart: unless-stopped

volumes:
  mongo-data:

Run both containers:
docker-compose up -d

---

## ⚙️ 3. CI/CD Setup

Using **GitHub Actions** for CI.

### `.github/workflows/ci.yml`
name: CI Pipeline

on:
  push:
    branches: [ "main" ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Log in to Docker Hub
        run: echo "${{ secrets.DOCKER_HUB_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_HUB_USERNAME }}" --password-stdin

      - name: Build Docker image
        run: docker build -t ${{ secrets.DOCKER_HUB_USERNAME }}/node-crud:latest ./backend

      - name: Push Docker image
        run: docker push ${{ secrets.DOCKER_HUB_USERNAME }}/node-crud:latest

✅ This pipeline automatically:
- Builds the app image
- Pushes to **Docker Hub** on every `main` branch push

---

## ☸️ 4. Kubernetes Deployment

### **Helm Chart Installation**
helm install node-crud ./helm -n app --create-namespace

Verify:
kubectl get pods -n app
kubectl get svc -n app

Access app:
http://<Node-IP>:30080

---

## 🧰 5. Infrastructure Setup (Terraform)

Terraform creates a Kubernetes cluster (EKS / kind / local).

Example `terraform/main.tf` (simplified):
provider "aws" {
  region = "us-east-1"
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  cluster_name = "demo-cluster"
  cluster_version = "1.29"
  vpc_id = var.vpc_id
  subnet_ids = var.subnet_ids
}

---

## 📊 6. Monitoring with Prometheus

Prometheus scrapes metrics from `/metrics` endpoint exposed by the Node.js backend using **prom-client**.

**Key Metrics:**
- `http_request_duration_seconds`
- `http_requests_total`
- `process_cpu_seconds_total`

### **ServiceMonitor**
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: backend-servicemonitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-crud
  endpoints:
    - port: http
      path: /metrics
      interval: 15s

### **Alert Rules**
groups:
- name: backend-alerts
  rules:
  - alert: BackendDown
    expr: up{job="backend"} == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Backend is Down"
      description: "No metrics received for 2 minutes."

  - alert: HighLatency
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High Response Latency"
      description: "95th percentile response > 1s for last 5 mins."

---

## 🧪 Verification Checklist

| ✅ Task | Status |
|---------|---------|
| CRUD APIs working | ✔️ |
| Dockerized application | ✔️ |
| Docker Compose for local dev | ✔️ |
| CI/CD pipeline builds & pushes image | ✔️ |
| Kubernetes deployment via Helm | ✔️ |
| Prometheus & Alertmanager configured | ✔️ |
| Alerting for downtime & latency | ✔️ |
| README documentation | ✔️ |

---

## 📝 Clarifications & Design Choices

- **Language:** Node.js chosen for simplicity and native Prometheus integration (`prom-client`).
- **Database:** PostgreSQL used for strong consistency and Docker support.
- **CI/CD:** GitHub Actions selected for easy integration and transparency.
- **Kubernetes:** Helm used to standardize deployment templates.
- **Monitoring:** Prometheus chosen as open-source, industry-standard metrics system.
- **IaC:** Terraform preferred for reproducibility and modular infra setup.

If any alternative approach was required (e.g., Python Flask backend), similar steps apply — the architecture remains the same.

---

## ✅ Reviewer Notes

- Run locally using `docker-compose up`  
- CI automatically builds images on code push  
- Deploy to Kubernetes using `helm install`  
- Observe metrics in Prometheus and alerts in Alertmanager  

This README ensures **minimal setup effort** and full traceability from code → deploy → monitor.

---

## 🏁 Conclusion

This project demonstrates a complete **DevOps workflow** covering:
- Application development  
- Containerization  
- Continuous integration  
- Kubernetes orchestration  
- Observability and alerting  

> Designed to be modular, scalable, and easy to replicate in real-world production environments.
