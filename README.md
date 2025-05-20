# Deploying a Static Flask Web App on AWS EKS with CI/CD, Trivy, Prometheus & Grafana

#This project demonstrates:

✅ Static web app using Flask + HTML
✅ Dockerized application pushed to Docker Hub
✅ CI/CD with GitHub Actions
✅ Deployed to AWS EKS cluster
✅ Monitored with Prometheus + Grafana
✅ Image scanned with Trivy for vulnerabilities
📁 Project Structure

```bash
eks-static-app/
├── app.py                 # Flask app
├── index.html             # HTML content
├── requirements.txt       # Python dependencies
├── Dockerfile             # Docker build config
├── k8s/
│   ├── deployment.yaml    # Kubernetes deployment
│   └── service.yaml       # Kubernetes service (LoadBalancer)
├── .github/
│   └── workflows/
│       └── ci-cd.yml      # GitHub Actions workflow
├── README.md              # This file
```

#🛠️ Step 1: Build the Flask Static Web App

app.py

```bash
from flask import Flask, render_template_string
app = Flask(__name__)
@app.route('/')
def index():
    return render_template_string(open("index.html").read())
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```
index.html

```bash
<!DOCTYPE html>
<html>
<head>
  <style>
    header, footer { background: #007bff; color: white; padding: 1rem; font-weight: bold; text-align: center; }
    main { padding: 2rem; text-align: center; }
  </style>
</head>
<body>
  <header>Hello from EKS App</header>
  <main><h1>Welcome to Kubernetes with Monitoring</h1></main>
  <footer>&copy; 2025 manukumarkc</footer>
</body>
</html>
```

requirements.txt

```bash
flask
```

#🐳 Step 2: Dockerize the App

Dockerfile

```bash
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

#Build & Push to Docker Hub

```bash
docker build -t hello-world-app .
docker tag hello-world-app manukumarkc/eks-static-app:latest
docker login
docker push manukumarkc/eks-static-app:latest
```

    Make sure your Docker Hub repo exists and is public (or configure a secret in Kubernetes for private repos).

#🔄 Step 3: CI/CD with GitHub Actions

.github/workflows/ci-cd.yml

```bash
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install flake8
        flake8 app.py

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push Docker image
      run: |
        docker build -t manukumarkc/eks-static-app:latest .
        docker push manukumarkc/eks-static-app:latest

    - name: Trivy Scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: manukumarkc/eks-static-app:latest
```

    Add GitHub secrets DOCKER_USERNAME and DOCKER_PASSWORD.

#☸️ Step 4: Kubernetes Deployment on AWS EKS
Create Cluster with eksctl

```bash
eksctl create cluster \
  --name eks-static-cluster \
  --region eu-west-1 \
  --nodes 2 \
  --node-type t2.medium \
  --managed
```

Kubernetes Manifests

k8s/deployment.yaml
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eks-static-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: eks-app
  template:
    metadata:
      labels:
        app: eks-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/metrics"
        prometheus.io/port: "5000"
    spec:
      containers:
        - name: eks-app
          image: manukumarkc/eks-static-app:latest
          ports:
            - containerPort: 5000
```
k8s/service.yaml
```bash
apiVersion: v1
kind: Service
metadata:
  name: eks-static-service
spec:
  type: LoadBalancer
  selector:
    app: eks-app
  ports:
    - port: 80
      targetPort: 5000
```
Deploy to EKS
```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```
Get the public IP:
```bash
kubectl get svc eks-static-service
```
📊 Step 5: Monitoring with Prometheus & Grafana
```bash
Add Helm Repo & Install
```

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

```bash
kubectl create namespace monitoring

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring
```
Access Grafana
```bash
kubectl get svc -n monitoring monitoring-grafana
```

    Type: LoadBalancer or NodePort

    Default login:

        Username: admin

        Password: prom-operator

#📈 Step 6: Visualize Metrics in Grafana

    Open Grafana from the LoadBalancer IP.

    Login using default credentials.

    Go to Dashboards > New > Import.

    Use dashboard ID: 1860 (Kubernetes Deployment Overview)

    Select the Prometheus data source (already set).
