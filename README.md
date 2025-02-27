# ☁️ MultiCloud, DevOps & AI Challenge - Day 2

# MultiCloud, DevOps & AI Challenge - Day 2

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
sudo systemctl start docker
sudo docker run hello-world
sudo systemctl enable docker
docker --version
sudo usermod -a -G docker $(whoami)
newgrp docker
