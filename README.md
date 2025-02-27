# ☁️ MultiCloud, DevOps & AI Challenge - Day 2

This repository contains the implementation for Day 2 of the MultiCloud, DevOps & AI Challenge, focusing on Docker and Kubernetes deployment of a CloudMart application.

## Overview

The challenge involves deploying a containerized application (CloudMart) with frontend and backend components on a Kubernetes cluster using Amazon EKS. The application is first containerized using Docker, then the images are stored in Amazon ECR before being deployed to a Kubernetes cluster.

## Architecture

- **Docker Containers**: Lightweight, portable containers for the frontend and backend applications
- **Amazon ECR**: Registry for Docker images
- **Amazon EKS**: Managed Kubernetes service for container orchestration
- **CloudMart Application**: Full-stack application with:
  - Backend: Node.js API service
  - Frontend: React web application

## Prerequisites

- AWS Account with appropriate permissions
- EC2 instance with AWS CLI configured
- Basic knowledge of Docker and Kubernetes

## Part 1: Docker Implementation

### Step 1: Install Docker on EC2

### **Install Docker on EC2**
```bash
sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo docker run hello-world
sudo systemctl enable docker
docker --version
sudo usermod -a -G docker $(whoami)
newgrp docker
```
```bash
sudo usermod -a -G docker $(whoami)
newgrp docker
```
### Step 2: Create Docker image for CloudMart
- Create and Download Backend folder
```bash
mkdir -p challenge-day2/backend && cd challenge-day2/backend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-backend.zip
unzip cloudmart-backend.zip
```
**Create env files (Environmental variables needed to host the APP)**
```bash
nano .env
```
```bash
PORT=5000
AWS_REGION=us-east-1
BEDROCK_AGENT_ID=<your-bedrock-agent-id>
BEDROCK_AGENT_ALIAS_ID=<your-bedrock-agent-alias-id>
OPENAI_API_KEY=<your-openai-api-key>
OPENAI_ASSISTANT_ID=<your-openai-assistant-id>
```
**Create Dockerfile for backend (Blue print of how we want the image to be created)**
```bash
FROM node:18
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["npm", "start"]
```
- Create and Download Frontend folder
  ```bash
cd ..
mkdir frontend && cd frontend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-frontend.zip
unzip cloudmart-frontend.zip
```
```bash
nano Dockerfile
```
```bash
FROM node:16-alpine
WORKDIR /app
RUN npm install -g serve
COPY --from=build /app/dist /app
ENV PORT=5001
ENV NODE_ENV=production
EXPOSE 5001
CMD ["serve", "-s", ".", "-l", "5001"]

```
## Part 2: Kubernetes
### Step 1: Cluster Setup on AWS Elastic Kubernetes Services (EKS) 
- Create a user named eksuser with Admin privileges and authenticate with it
  ```bash
aws configure
```
- Install the CLI tool eksctl
  ```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo cp /tmp/eksctl /usr/bin
eksctl version
```
- Install the CLI tool kubectl - to interact with cluster
   ```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
kubectl version --short --client
```
- Create an EKS Cluster
  ```bash
eksctl create cluster \
  --name cloudmart \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 1 \
  --with-oidc \
  --managed
```
- Connect to the EKS cluster using the kubectl configuration
    ```bash
aws eks update-kubeconfig --name cloudmart
```
- Verify Cluster Connectivity
    ```bash
kubectl get svc
kubectl get nodes
```
![Capture d’écran 2025-02-26 173002](https://github.com/user-attachments/assets/39698fe8-030c-421a-946d-2b4f8b98cc27)

- Create a Role & Service Account to provide pods access to services used by the application (DynamoDB, Bedrock, etc).-  to allow the cluster to interact with aws services
```bash
eksctl create iamserviceaccount \
  --cluster=cloudmart \
  --name=cloudmart-pod-execution-role \
  --role-name CloudMartPodExecutionRole \
  --attach-policy-arn=arn:aws:iam::aws:policy/AdministratorAccess\
  --region us-east-1 \
  --approve
```
### Step 2: Backend Deployment on Kubernetes
- Create an ECR Repository for the Backend and upload the Docker image to it
  ```bash
Repository name: cloudmart-backend
```
- Switch to backend folder
 ```bash
cd ../..
cd challenge-day2/backend
```
-  build your Docker image with ECR view push commands from AWS Console
-  Create a Kubernetes deployment file (YAML) for the Backend - will use the image from ECR
   ```bash
cd ../..
cd challenge-day2/backend
nano cloudmart-backend.yaml
```
  ```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudmart-backend-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudmart-backend-app
  template:
    metadata:
      labels:
        app: cloudmart-backend-app
    spec:
      serviceAccountName: cloudmart-pod-execution-role
      containers:
      - name: cloudmart-backend-app
        image: public.ecr.aws/l4c0j8h9/cloudmart-backend:latest // replace with image url from ECR in AWS
        env:
        - name: PORT
          value: "5000"
        - name: AWS_REGION
          value: "us-east-1"
        - name: BEDROCK_AGENT_ID
          value: "xxxxxx"
        - name: BEDROCK_AGENT_ALIAS_ID
          value: "xxxx"
        - name: OPENAI_API_KEY
          value: "xxxxxx"
        - name: OPENAI_ASSISTANT_ID
          value: "xxxx"
---

apiVersion: v1
kind: Service
metadata:
  name: cloudmart-backend-app-service
spec:
  type: LoadBalancer
  selector:
    app: cloudmart-backend-app
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
```
- Deploy the Backend on Kubernetes
   ```bash
kubectl apply -f cloudmart-backend.yaml
```
- Monitor the status of objects being created and obtain the public IP generated for the API
   ```bash
kubectl get pods
kubectl get deployment
kubectl get service
```
![Capture d’écran 2025-02-26 185016](https://github.com/user-attachments/assets/c41b3cda-a130-4b30-9993-4ddfdbc75f43)

### Step 3: Frontend Deployment on Kubernetes
   ```bash
cd ../challenge-day2/frontend
nano .env
```
  ```bash
VITE_API_BASE_URL=http://<your_url_kubernetes_api>:5000/api
```
- Create an ECR Repository for the Frontend and upload the Docker image to it
    ```bash
Repository name: cloudmart-frontend
```
-  build your Docker image with ECR view push commands from AWS Console
    ```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudmart-frontend-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudmart-frontend-app
  template:
    metadata:
      labels:
        app: cloudmart-frontend-app
    spec:
      serviceAccountName: cloudmart-pod-execution-role
      containers:
      - name: cloudmart-frontend-app
        image: public.ecr.aws/l4c0j8h9/cloudmart-frontend:latest //replace with yours
---

apiVersion: v1
kind: Service
metadata:
  name: cloudmart-frontend-app-service
spec:
  type: LoadBalancer
  selector:
    app: cloudmart-frontend-app
  ports:
    - protocol: TCP
      port: 5001
      targetPort: 5001
```
- Deploy the Frontend on Kubernetes
     ```bash
     kubectl apply -f cloudmart-frontend.yaml
```
     ```bash
   kubectl get pods
kubectl get deployment
kubectl get service  
```
![Capture d’écran 2025-02-26 190627](https://github.com/user-attachments/assets/2c283a6b-9259-44e2-926e-5abb2c4a7eb6)

- Access the App with address of load balencer - http://a88bf6a00308549008c33b7eba75fd72-389695630.us-east-1.elb.amazonaws.com:5001/
  ![Capture d’écran 2025-02-27 005835](https://github.com/user-attachments/assets/bb094288-013c-4485-aa66-fdef2aa862d1)

  - Acces the backend with  http://a88bf6a00308549008c33b7eba75fd72-389695630.us-east-1.elb.amazonaws.com:5001/Admin to modify the backend and create new products

![Capture d’écran 2025-02-27 010144](https://github.com/user-attachments/assets/5f2ee570-7c50-46be-acdf-0bc58af3d5a7)

  
![Capture d’écran 2025-02-27 010554](https://github.com/user-attachments/assets/2e5c8e7c-0dc5-47ae-bce5-faf5977eb522)

