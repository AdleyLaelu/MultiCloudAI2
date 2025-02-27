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

```bash
sudo yum update -y
sudo yum install docker -y


# Project Title

A brief description of what this project does and who it's for
MultiCloud, DevOps & AI Challenge - Day 2

📌 Project Overview

This project focuses on deploying the CloudMart application using Docker and Amazon EKS (Elastic Kubernetes Service) to provide a scalable and reliable cloud-native architecture. The application consists of:

Backend: A Node.js-based API

Frontend: A React-based web application

Both services are containerized with Docker, stored in Amazon ECR (Elastic Container Registry), and deployed into AWS EKS (Elastic Kubernetes Service) using Kubernetes.

📌 Project Architecture

1️⃣ Containerization with Docker

The frontend and backend applications are containerized using Docker.

Docker images are pushed to Amazon ECR (Elastic Container Registry).

2️⃣ Kubernetes Deployment with Amazon EKS

A managed Kubernetes cluster (EKS) is created to orchestrate the deployment.

Backend and frontend pods are deployed in separate Deployments.

Each service is exposed via LoadBalancer Services.

📌 Prerequisites

Before setting up the project, ensure you have:

An AWS account with AdministratorAccess permissions.

An EC2 instance with Amazon Linux 2 (or similar).

AWS CLI, eksctl, and kubectl installed.

Docker installed and running.

📌 1️⃣ Setup & Installations

Install Docker

sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
docker --version
sudo usermod -a -G docker $(whoami)
newgrp docker

📌 2️⃣ Creating Docker Images

Backend

Create and enter the backend folder

mkdir -p challenge-day2/backend && cd challenge-day2/backend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-backend.zip
unzip cloudmart-backend.zip

Create .env file

nano .env

Contents:

PORT=5000
AWS_REGION=us-east-1
BEDROCK_AGENT_ID=<your-bedrock-agent-id>
BEDROCK_AGENT_ALIAS_ID=<your-bedrock-agent-alias-id>
OPENAI_API_KEY=<your-openai-api-key>
OPENAI_ASSISTANT_ID=<your-openai-assistant-id>

Create Dockerfile

nano Dockerfile

Contents:

FROM node:18
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["npm", "start"]

Frontend

Create and enter the frontend folder

cd ..
mkdir frontend && cd frontend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-frontend.zip
unzip cloudmart-frontend.zip

Create Dockerfile

nano Dockerfile

Contents:

FROM node:16-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:16-alpine
WORKDIR /app
RUN npm install -g serve
COPY --from=build /app/dist /app
ENV PORT=5001
ENV NODE_ENV=production
EXPOSE 5001
CMD ["serve", "-s", ".", "-l", "5001"]

📌 3️⃣ Kubernetes Deployment

Create an EKS Cluster

eksctl create cluster \
  --name cloudmart \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 1 \
  --with-oidc \
  --managed

Connect to EKS Cluster

aws eks update-kubeconfig --name cloudmart

Verify Cluster Connectivity

kubectl get svc
kubectl get nodes

Create IAM Role & Service Account

eksctl create iamserviceaccount \
  --cluster=cloudmart \
  --name=cloudmart-pod-execution-role \
  --role-name CloudMartPodExecutionRole \
  --attach-policy-arn=arn:aws:iam::aws:policy/AdministratorAccess\
  --region us-east-1 \
  --approve

📌 4️⃣ Deploying Backend

Create Kubernetes YAML Deployment

nano backend/cloudmart-backend.yaml

Contents:

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
        image: public.ecr.aws/l4c0j8h9/cloudmart-backend:latest
        env:
        - name: PORT
          value: "5000"
        - name: AWS_REGION
          value: "us-east-1"
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

Apply Deployment

kubectl apply -f backend/cloudmart-backend.yaml

📌 5️⃣ Deploying Frontend

Create Kubernetes YAML Deployment

nano frontend/cloudmart-frontend.yaml

Contents:

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
        image: public.ecr.aws/l4c0j8h9/cloudmart-frontend:latest
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

Apply Deployment

kubectl apply -f frontend/cloudmart-frontend.yaml

Verify Deployment

kubectl get pods
kubectl get deployment
kubectl get service

sudo systemctl start docker
sudo docker run hello-world
sudo systemctl enable docker
docker --version
sudo usermod -a -G docker $(whoami)
newgrp docker
