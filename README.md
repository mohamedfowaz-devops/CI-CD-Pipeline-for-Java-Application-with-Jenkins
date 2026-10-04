# Java CI/CD Pipeline with Jenkins, Docker, AWS ECR and Ansible

## Overview

This project demonstrates a CI/CD pipeline for a Java application using Jenkins, Maven, Docker, AWS ECR, and Ansible.

The pipeline automates the process of building the Java application, creating a Docker image, pushing the image to Amazon ECR, and deploying the application to multiple EC2 instances using Ansible.

## Tools & Technologies

* Java
* Maven
* Git & GitHub
* Jenkins
* Docker
* AWS EC2
* AWS ECR
* Ansible
* Linux
* Shell Scripting

## CI/CD Workflow

```text
GitHub
   ↓
Jenkins
   ↓
Maven Build & Test
   ↓
Docker Build
   ↓
Push Docker Image to AWS ECR
   ↓
Ansible Deployment
   ↓
App Server 1
   ↓
Health Check
   ↓
App Server 2
   ↓
Health Check
```

## Project Structure

```text
CI-CD-Pipeline-for-Java-Application-with-Jenkins/
│
├── java-cicd/
│   ├── src/
│   ├── pom.xml
│   ├── Dockerfile
│   └── Jenkinsfile
│
├── ansible/
│   ├── inventory
│   └── deploy.yml
│
└── README.md
```

## Pipeline Stages

### 1. Checkout

Jenkins checks out the application source code from GitHub.

### 2. Build and Test

Maven is used to build the Java application and execute unit tests.

### 3. Docker Build

The application is packaged into a Docker image.

```text
java-cicd-demo:<BUILD_NUMBER>
```

### 4. Push to AWS ECR

The Docker image is tagged and pushed to an Amazon ECR repository.

```text
AWS ECR
└── java-cicd-demo
```

### 5. Ansible Deployment

Jenkins triggers the Ansible playbook to deploy the Docker image to the application servers.

### 6. Rolling Deployment

Ansible deploys the application one server at a time.

```text
App Server 1
     ↓
Deploy
     ↓
Health Check
     ↓
App Server 2
     ↓
Deploy
     ↓
Health Check
```

This approach avoids updating both application servers simultaneously.

## Deployment

The deployment is performed using Ansible. The inventory contains the application server details, while the playbook handles the application deployment and health checks.

Example command:

```bash
ansible-playbook -i inventory deploy.yml
```

## AWS Components

* **EC2** – Jenkins and application servers
* **ECR** – Docker image repository
* **IAM** – Access control for AWS resources

## Key Learning

This project provided hands-on experience with:

* Jenkins declarative pipelines
* Maven build and testing
* Docker image creation
* AWS ECR image management
* Ansible-based deployments
* Linux server administration
* Rolling application deployments
* Application health checks
* CI/CD automation

