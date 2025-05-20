# Deploying a Static Flask Web App on AWS EKS with CI/CD, Trivy, Prometheus & Grafana

##This project demonstrates:

1:Static web app with Flask and HTML.

2:Containerize the app with Docker and push it to Docker Hub.

3:CI/CD cretion with GitHub Action Workflows.

4:Deploying the Application on AWS EKS Cluster.

5:Monitor the Application and System metrics with Prometheus and Grafana.

6:Check for the Image Scanned Vulnerabilities from Trivy.


# Project Tree Structure:

```bash
eks-static-app/
├── app.py                 # app.py of flask application
├── index.html             # HTML details for static page 
├── requirements.txt       #python flask dependencies.
├── Dockerfile             # Dockerfile for containerizing the app
├── k8s/
│   ├── deployment.yaml    #Deployment manifests file
│   └── service.yaml       #kubernets service (loadbalancer) Manifests file.
├── .github/
│   └── workflows/
│       └── ci-cd.yml      # GitHub Actions workflow
├── README.md              #Clear Description of Readme content
```

# Application on live:

![image](https://github.com/user-attachments/assets/a9108a9b-81ae-4008-a3c0-b560d91127de)

# Build through GitHub Actions workflow:

 
# Step 1: Build the Flask Static Web App

-inside the app.py with using return render_template_string function rendering the index.html page to display the application content.
-below are the contents inside app.py
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

-index.html content with consisting of Header, body and footer in HTML with backgroung and bold contents.

```bash
<!DOCTYPE html>
<html>
<head>
  <style>
    header, footer { background: #76eec6; color: white; padding: 1rem; font-weight: bold; text-align: center; }
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

-Contents written inside the requirements.txt file are utilized while creating docker container.

```bash
flask
```

# Step 2: Dockerize the App

-Below are the details of Docker file and Explanation of Each Line.

Dockerfile

```bash
FROM python:3.9-slim #Getting the image from Python official image slim version.
WORKDIR /app #setting the workdirectory.
COPY requirements.txt . #copy the requirements file to present directory.
RUN pip install -r requirements.txt #run pip install command to install flask app.
COPY . .     # Copy application code to work directory.
EXPOSE 5000 #expose the app to 5000 port number
CMD ["python", "app.py"]  # set CMD for python and app.py for container runtime command
```

#Build & Push to Docker Hub

-Once Dockerfile is written, Container is built with dockerfile details.
-docker image is tagged to docker hub id and pushed to docker hub,

```bash
docker build -t hello-world-app .
docker tag hello-world-app manukumarkc/eks-static-app:latest
docker login
docker push manukumarkc/eks-static-app:latest
```

-Once Docker login is successful, the Docker login credentials are added n Github>Setting>Secrets and Variables>add secrets.

#Step 3: CI/CD with GitHub Actions

-Creating CI-CD workflow to create docker app and push it to DockerHUb and have Trivy for Image Vulnerability Scanning.

.github/workflows/ci-cd.yml

```bash
name: CI/CD Pipeline #Name for the Pipeline.

on:
  push:  #only Push to the Main branc is triggered.
    branches:
      - main

jobs:
  build: #job builds on ubuntu latest image server.
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code #code checkout from Github to run the pipeline.
      uses: actions/checkout@v3

    - name: Install dependencies #Installs all the dependencies and requirements for the application and format checking through Flake.
      run: | 
        python -m pip install --upgrade pip
        pip install flake8
        flake8 app.py

    - name: Set up Docker Buildx  #setting up Docker Buildx for effective platform indedependent image creation.
      uses: docker/setup-buildx-action@v3

    - name: Log in to Docker Hub #Docker login with Credentials inside github secrets repository.
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push Docker image #Docker image build and push it to Docker hub
      run: |
        docker build -t manukumarkc/eks-static-app:latest .
        docker push manukumarkc/eks-static-app:latest

    - name: Trivy Scan #Docker image Vulnerability Scan for the Images created and pushed in Dockerhub,
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: manukumarkc/eks-static-app:latest
```

    #Add GitHub secrets DOCKER_USERNAME and DOCKER_PASSWORD.

# Step 4: Kubernetes Deployment on AWS EKS
-Below is the Command to deploy EKS Cluster with t2.medium configuration, and hosting the application with Manifests files by Manual deployment.

-Create Cluster with eksctl

```bash
eksctl create cluster \
  --name eks-static-cluster \
  --region eu-west-1 \
  --nodes 2 \
  --node-type t2.medium \
  --managed
```

Kubernetes Manifests

-Kubernetes Manifests file for Deployment which pulls the image from Docker Hub latest image and creates Pod and Services inside Deployment.

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

-Service Manifest file of Type Loadbalancer which creates Loadbalancer to expose the app to externally for Public Access:

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

-Manual Deployment of Deployment and Service YAML file for hosting application on EKS Cluster.

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```
Get the public IP: #the LoadBalancer IP is provided with Public access of application,Get the SVC URL and Access the application on browser.

```bash
kubectl get svc eks-static-service
```
# Step 5: Monitoring with Prometheus & Grafana

-Monitoring services are installed for Prometheus and Grafana.
-Below are the commands for installing Prometheus and Grafana.

```bash
Add Helm Repo & Install
```

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

```bash 
kubectl create namespace monitoring #creating a separate namespace for isolation.

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring
```
Access Grafana #accessing the SVC url for Grafana Dashboard.

```bash
kubectl get svc -n monitoring monitoring-grafana
```

    Type: LoadBalancer 

    Default login:

        Username: admin

        Password: prom-operator

# Step 6: Visualize Metrics in Grafana

    Open Grafana from the LoadBalancer IP.

    Login using default credentials.

-Once the Dashboard is visible, there are two newly added Dashboard for Accessing System metrics and Application metrics of EKS Cluster.
-Below is the Reference image for reference:

