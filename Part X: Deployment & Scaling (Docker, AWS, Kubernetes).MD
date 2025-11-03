# 🚀 Flask Mastery Series – Part X: Deployment & Scaling (Docker, AWS, Kubernetes)

---

## 📘 Table of Contents

1. [Introduction](#introduction)
2. [Deployment Environments Overview](#deployment-environments-overview)
3. [Preparing Flask for Production](#preparing-flask-for-production)
4. [Containerization with Docker](#containerization-with-docker)
5. [Docker Compose for Multi-Service Apps](#docker-compose-for-multi-service-apps)
6. [Gunicorn + Nginx Stack](#gunicorn--nginx-stack)
7. [Environment Configuration & Secrets Management](#environment-configuration--secrets-management)
8. [CI/CD Pipelines for Deployment](#cicd-pipelines-for-deployment)
9. [Cloud Deployment (AWS, GCP, Azure)](#cloud-deployment-aws-gcp-azure)

   * [AWS Elastic Beanstalk](#aws-elastic-beanstalk)
   * [AWS ECS / Fargate](#aws-ecs--fargate)
   * [GCP Cloud Run](#gcp-cloud-run)
10. [Kubernetes Deployment](#kubernetes-deployment)

    * [Manifest & Secrets Setup](#manifest--secrets-setup)
    * [Horizontal Scaling & Load Balancing](#horizontal-scaling--load-balancing)
11. [Monitoring & Logging in Production](#monitoring--logging-in-production)
12. [Autoscaling & Cost Optimization](#autoscaling--cost-optimization)
13. [Disaster Recovery & Backup Strategies](#disaster-recovery--backup-strategies)
14. [Deployment Checklist](#deployment-checklist)
15. [Next Steps](#next-steps)

---

## 🧠 Introduction

This final part of the Flask Mastery Series covers how to take your Flask app from development to production. You’ll learn Docker-based deployments, CI/CD pipelines, AWS/GCP setups, and scaling strategies using Kubernetes.

---

## 🌍 Deployment Environments Overview

### Common environments

* **Development:** Local with `flask run`, hot reloading, debug enabled.
* **Staging:** Replica of production, used for QA testing.
* **Production:** Stable release, served by WSGI server (Gunicorn/uWSGI).

### Deployment goals

✅ Consistency (same app across environments)
✅ Scalability (horizontal & vertical)
✅ Reliability (monitoring & fault tolerance)

---

## 🧱 Preparing Flask for Production

Checklist:

* Disable debug mode (`DEBUG=False`).
* Serve with WSGI (Gunicorn, uWSGI) — not the built-in Flask dev server.
* Use environment variables for configuration.
* Configure logging and error tracking (Sentry, Datadog, ELK).
* Use reverse proxy (Nginx) for HTTPS and static files.

Example production entry:

```bash
gunicorn -w 4 -b 0.0.0.0:5000 app:create_app()
```

---

## 🐳 Containerization with Docker

Example `Dockerfile`:

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:create_app()"]
```

Build and run:

```bash
docker build -t flask-app .
docker run -p 5000:5000 flask-app
```

---

## 🧩 Docker Compose for Multi-Service Apps

Example with PostgreSQL + Redis:

```yaml
version: '3.8'
services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/app
      REDIS_URL: redis://cache:6379/0
    depends_on:
      - db
      - cache
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
  cache:
    image: redis:7
```

Run:

```bash
docker-compose up -d
```

---

## 🧠 Gunicorn + Nginx Stack

Nginx acts as a reverse proxy in front of Gunicorn.

**nginx.conf:**

```nginx
server {
    listen 80;
    server_name _;

    location /static/ {
        alias /app/static/;
        expires 30d;
    }

    location / {
        proxy_pass http://web:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Compose snippet:

```yaml
nginx:
  image: nginx:alpine
  ports:
    - "80:80"
  volumes:
    - ./nginx.conf:/etc/nginx/conf.d/default.conf
  depends_on:
    - web
```

---

## 🔑 Environment Configuration & Secrets Management

Use `.env` or environment variables for sensitive config.

```bash
SECRET_KEY=supersecret
DATABASE_URL=postgresql://user:pass@db:5432/app
JWT_SECRET_KEY=jwtsecret
```

Load via:

```py
from dotenv import load_dotenv
load_dotenv()
```

For production, use:

* **AWS Secrets Manager**
* **GCP Secret Manager**
* **Vault by HashiCorp**

---

## ⚙️ CI/CD Pipelines for Deployment

Automate testing and deployment with GitHub Actions.

Example `.github/workflows/deploy.yml`:

```yaml
name: Deploy to AWS ECS
on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Build Docker image
        run: docker build -t flask-app:latest .
      - name: Push to ECR
        run: |
          aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <account>.dkr.ecr.ap-south-1.amazonaws.com
          docker tag flask-app:latest <account>.dkr.ecr.ap-south-1.amazonaws.com/flask-app:latest
          docker push <account>.dkr.ecr.ap-south-1.amazonaws.com/flask-app:latest
      - name: Deploy to ECS
        run: aws ecs update-service --cluster flask-cluster --service flask-service --force-new-deployment
```

---

## ☁️ Cloud Deployment (AWS, GCP, Azure)

### AWS Elastic Beanstalk

```bash
eb init -p docker flask-app
 eb create flask-env
 eb deploy
```

### AWS ECS / Fargate

* Run containers on AWS-managed cluster.
* Configure autoscaling, secrets, and logs via CloudWatch.

### GCP Cloud Run

```bash
gcloud run deploy flask-app --source . --region asia-south1 --allow-unauthenticated
```

Cloud Run auto-scales containers and bills per request.

---

## ☸️ Kubernetes Deployment

### Manifest & Secrets Setup

`deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask
          image: flask-app:latest
          ports:
            - containerPort: 5000
          envFrom:
            - secretRef:
                name: flask-secrets
---
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  type: LoadBalancer
  selector:
    app: flask-app
  ports:
    - port: 80
      targetPort: 5000
```

`secrets.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: flask-secrets
type: Opaque
data:
  SECRET_KEY: c3VwZXJzZWNyZXQ=
```

Deploy:

```bash
kubectl apply -f secrets.yaml -f deployment.yaml
```

### Horizontal Scaling & Load Balancing

Use Horizontal Pod Autoscaler (HPA):

```bash
kubectl autoscale deployment flask-app --cpu-percent=70 --min=2 --max=10
```

Use NGINX Ingress for routing and TLS termination.

---

## 📊 Monitoring & Logging in Production

Tools:

* **Prometheus + Grafana** — metrics
* **ELK (Elasticsearch, Logstash, Kibana)** — logs
* **Loki + Grafana** — lightweight logging
* **Sentry** — error tracking

Integrate health checks (`/healthz`) and readiness probes for load balancers.

---

## 📈 Autoscaling & Cost Optimization

* Use auto-scaling groups (ECS/Kubernetes) based on CPU, memory, or custom metrics.
* Use **spot instances** or **serverless** (Cloud Run/Lambda) for cost efficiency.
* Scale DB with read replicas or managed services (RDS, Cloud SQL).

---

## 🧯 Disaster Recovery & Backup Strategies

* Schedule DB backups via RDS or pg_dump.
* Store backups on S3/Cloud Storage with lifecycle rules.
* Implement multi-zone deployments for failover.

---

## ✅ Deployment Checklist

* [ ] Flask app containerized and running with Gunicorn.
* [ ] Environment variables & secrets stored securely.
* [ ] CI/CD pipeline automated with testing & deploy stages.
* [ ] Monitoring, logging, and alerts configured.
* [ ] Horizontal scaling and health checks tested.
* [ ] Backups and recovery plan in place.
* [ ] TLS/HTTPS configured via Nginx/Ingress.

---

## 🏁 Next Steps

🎓 You’ve completed the **Flask Mastery Series**!
Next: Explore **Advanced Flask Patterns** like microservices with gRPC, WebSockets (Flask-SocketIO), and event-driven architectures.

---

*Prepared for the Flask Mastery Series — exportable in `.md`, `.pdf`, and `.docx` formats.*
