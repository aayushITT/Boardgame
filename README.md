---

# BoardgameListing WebApp – CI/CD Deployment on AWS EKS

---
description: |
  BoardgameListing Web Application is a full-stack platform that lists board games and user reviews. 
  While anyone can browse games and reviews, users must log in to add new games or submit reviews. 
  Managers have extended privileges to edit or delete reviews.

  This project demonstrates a secure, automated, and monitored CI/CD pipeline on AWS, 
  integrating Jenkins, SonarQube, Nexus, Docker, Trivy, EKS, Prometheus, and Grafana 
  to ensure reliable application deployment and observability.
---

## Technologies Used

* **CI/CD:** Jenkins
* **Code Quality & Security:** SonarQube
* **Artifact Repository:** Nexus
* **Container Security:** Trivy
* **Container Registry:** AWS ECR
* **Orchestration:** Kubernetes on AWS EKS
* **Monitoring & Visualization:** Prometheus, Grafana
* **Cloud Infrastructure:** AWS EC2, IAM, Network Load Balancer (NLB)

---

## Architecture Overview

### 1. EC2 Instances

* **Main Web Server:** SSH access and orchestration hub
* **Jenkins Server (t3.medium):** Pipeline automation
* **SonarQube Server (t3.medium):** Static code analysis and quality gate
* **Nexus Server (t3.medium):** Artifact storage (JAR/TAR files)
* **Monitoring Server (t3.medium):** Prometheus and Grafana dashboards

### 2. AWS ECR

* Stores Docker images built from the application code
* Images are pulled for deployment in EKS

### 3. AWS EKS Cluster

* Managed Kubernetes control plane
* Worker Node Group with 2 EC2 nodes
* Integrated with AWS ECR for container image deployment

### 4. Monitoring Layer

* Prometheus + Grafana dashboards for real-time monitoring
* Metrics collected via **node-exporter** and **kube-state-metrics**

---

## End-to-End Setup

### 1. Jenkins Setup

* Installed Jenkins on EC2 (port 8080) and essential plugins
* Configured credentials for GitHub, SonarQube, Nexus, AWS ECR
* Installed Trivy for container vulnerability scanning

### 2. SonarQube Setup

* Deployed on a separate EC2 instance (port 9000)
* Configured webhook to send Quality Gate results to Jenkins
* Pipeline halts if the Quality Gate fails

### 3. Nexus Repository Setup

* Installed Nexus on EC2 (port 8081)
* Created dedicated Maven hosted repositories for application artifacts
* Jenkins pipeline uploads JAR files to Nexus after successful build

### 4. AWS EKS Cluster Setup

* Provisioned EKS cluster via AWS console/CLI
* Configured IAM roles and RBAC for Jenkins
* Created a managed node group with two EC2 worker nodes
* Installed `kubectl` and AWS CLI on Jenkins EC2
* Updated `aws-auth` ConfigMap to authorize nodes

---

## CI/CD Pipeline Overview (Jenkinsfile)

### Pipeline Stages

1. **Git Checkout:** Clone code from GitHub using PAT (Personal Access Token)
2. **Code Analysis (SonarQube):** Scan code for quality issues, security vulnerabilities, and code smells
3. **Quality Gate Check:** Halt pipeline if the Quality Gate fails
4. **Build:** Run `mvn clean install` to package the application into a JAR
5. **Artifact Upload to Nexus:** Store JAR in Nexus repository
6. **Docker Build:** Build Docker image and tag with `IMAGE_NAME:BUILD_NUMBER`
7. **Trivy Security Scan:** Run container security scan and generate vulnerability reports
8. **AWS ECR Login:** Authenticate Jenkins with AWS ECR
9. **Push Docker Image:** Push scanned Docker image to ECR and clean up local image
10. **Deploy to EKS:** Update Kubernetes manifest with image info and apply deployment using `kubectl`
11. **Post-Build Notifications:** Send email notifications for success or failure

---

## Monitoring & Observability

* Prometheus scrapes cluster and node metrics
* Grafana visualizes dashboards for pods, nodes, and application performance
* Metrics are continuously monitored to ensure stable deployment

---

## Workflow Summary

1. Developer pushes code to GitHub
2. Jenkins pipeline triggers automatically
3. SonarQube scans and checks Quality Gate
4. Maven builds and artifacts pushed to Nexus
5. Docker image built and scanned via Trivy
6. Image pushed to AWS ECR
7. Kubernetes deployment applied to EKS cluster
8. Prometheus + Grafana monitor application and cluster health
9. Email notifications sent to team

---
