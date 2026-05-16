# CI/CD Pipeline Setup using GitHub, Jenkins, DockerHub, and AWS EKS

## Project Overview

This project demonstrates the implementation of a complete CI/CD (Continuous Integration and Continuous Deployment) pipeline using:

* GitHub
* Jenkins
* Docker
* DockerHub
* AWS EKS (Elastic Kubernetes Service)
* Kubernetes

The objective of this project is to automate the software delivery lifecycle from source code commit to deployment on a Kubernetes cluster.

Whenever code is pushed to the GitHub repository, Jenkins automatically triggers the pipeline, builds the Docker image, pushes it to DockerHub, and deploys the application to AWS EKS.

---

# Architecture Diagram

```text
Developer Push Code
        ↓
GitHub Repository
        ↓
GitHub Webhook Trigger
        ↓
Jenkins Pipeline
        ↓
Docker Image Build
        ↓
Push Image to DockerHub
        ↓
Deploy to AWS EKS
        ↓
Kubernetes Service (LoadBalancer)
        ↓
Application Accessible via Browser
```

---

# Technologies Used

| Technology   | Purpose                    |
| ------------ | -------------------------- |
| GitHub       | Source Code Management     |
| Jenkins      | CI/CD Automation           |
| Groovy       | Jenkins Pipeline Scripting |
| Docker       | Containerization           |
| DockerHub    | Container Registry         |
| AWS EKS      | Kubernetes Cluster         |
| Kubernetes   | Container Orchestration    |
| kubectl      | Kubernetes CLI             |
| eksctl       | EKS Cluster Management     |
| AWS CLI      | AWS Configuration          |
| Python Flask | Sample Application         |

---

# Infrastructure Setup

## AWS EC2 Instance

An EC2 instance was launched to host Jenkins and required DevOps tools.

### EC2 Configuration

| Setting       | Value          |
| ------------- | -------------- |
| AMI           | Ubuntu 22.04   |
| Instance Type | c7i-flex.large |
| Storage       | 25 GB          |
| Region        | eu-north-1     |

### Security Group Rules

| Port | Purpose            |
| ---- | ------------------ |
| 22   | SSH Access         |
| 8080 | Jenkins            |
| 80   | Application Access |
| 443  | HTTPS              |

---

# Java Installation

Jenkins latest version requires Java 21.

## Install Java 21

```bash
sudo apt update
sudo apt install openjdk-21-jdk -y
```

## Verify Java Version

```bash
java -version
```

Expected Output:

```text
openjdk version "21"
```

---

# Jenkins Installation

## Add Jenkins Repository

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
/usr/share/keyrings/jenkins-keyring.asc > /dev/null
```

```bash
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null
```

## Install Jenkins

```bash
sudo apt update
sudo apt install jenkins -y
```

## Start Jenkins

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

## Verify Jenkins Status

```bash
sudo systemctl status jenkins
```

---

# Docker Installation

## Install Docker

```bash
sudo apt install docker.io -y
```

## Enable Docker

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

## Give Docker Access

```bash
sudo usermod -aG docker ubuntu
sudo usermod -aG docker jenkins
```

## Restart Jenkins

```bash
sudo systemctl restart jenkins
```

---

# AWS CLI Installation

## Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

```bash
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
```

## Configure AWS CLI

```bash
aws configure
```

---

# kubectl Installation

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

```bash
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

---

# eksctl Installation

```bash
curl --silent --location \
"https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
| tar xz -C /tmp
```

```bash
sudo mv /tmp/eksctl /usr/local/bin
```

---

# AWS EKS Cluster Setup

## Create EKS Cluster

```bash
eksctl create cluster \
--name devops-cluster \
--region eu-north-1 \
--nodegroup-name workers \
--node-type t3.small \
--nodes 1 \
--managed
```

## Verify Nodes

```bash
kubectl get nodes
```

Expected Output:

```text
STATUS: Ready
```

---

# Jenkins Plugin Installation

Installed Jenkins plugins:

* Git Plugin
* GitHub Integration Plugin
* Docker Pipeline Plugin
* Kubernetes Plugin
* Pipeline Plugin
* Credentials Binding Plugin
* Blue Ocean Plugin

---

# Application Setup

## Project Structure

```text
myapp/
├── app.py
├── requirements.txt
├── Dockerfile
├── deployment.yaml
├── service.yaml
└── Jenkinsfile
```

---

# Flask Application

## app.py

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return "CI/CD Pipeline on AWS EKS Successful"

app.run(host='0.0.0.0', port=5000)
```

---

# Docker Configuration

## Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

---

# Kubernetes Deployment

## deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: myapp-deployment

spec:
  replicas: 2

  selector:
    matchLabels:
      app: myapp

  template:
    metadata:
      labels:
        app: myapp

    spec:
      containers:
      - name: myapp
        image: sagarsalve49/myapp:latest

        ports:
        - containerPort: 5000
```

---

# Kubernetes Service

## service.yaml

```yaml
apiVersion: v1
kind: Service

metadata:
  name: myapp-service

spec:
  type: LoadBalancer

  selector:
    app: myapp

  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000
```

---

# Jenkins Pipeline Configuration

## Jenkinsfile

```groovy
pipeline {

    agent any

    environment {

        DOCKER_IMAGE = "sagarsalve49/myapp"
        DOCKER_TAG = "${BUILD_NUMBER}"

    }

    triggers {
        githubPush()
    }

    stages {

        stage('Clone Code') {

            steps {

                git branch: 'main',
                url: 'https://github.com/Sagar-salve-49/FinacPlus-Task.git'
            }
        }

        stage('Build Docker Image') {

            steps {

                sh 'docker build -t $DOCKER_IMAGE:$DOCKER_TAG .'

                sh 'docker tag $DOCKER_IMAGE:$DOCKER_TAG $DOCKER_IMAGE:latest'
            }
        }

        stage('Push Docker Image') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

                    docker push $DOCKER_IMAGE:$DOCKER_TAG

                    docker push $DOCKER_IMAGE:latest
                    '''
                }
            }
        }

        stage('Deploy to EKS') {

            steps {

                sh '''
                aws eks update-kubeconfig \
                --region eu-north-1 \
                --name devops-cluster

                kubectl apply -f deployment.yaml

                kubectl apply -f service.yaml
                '''
            }
        }

        stage('Verify Deployment') {

            steps {

                sh '''
                kubectl get pods

                kubectl get svc
                '''
            }
        }
    }

    post {

        success {

            echo 'Pipeline Executed Successfully'
        }

        failure {

            echo 'Pipeline Failed'
        }
    }
}
```

---

# GitHub Webhook Configuration

GitHub webhook was configured to automatically trigger Jenkins builds whenever code is pushed.

## Webhook URL

```text
http://JENKINS_PUBLIC_IP:8080/github-webhook/
```

---

# CI/CD Pipeline Workflow

## Step-by-Step Flow

1. Developer pushes code to GitHub
2. GitHub webhook triggers Jenkins pipeline
3. Jenkins clones latest source code
4. Docker image is built automatically
5. Docker image is pushed to DockerHub
6. Jenkins deploys updated image to AWS EKS
7. Kubernetes updates running pods
8. Application becomes accessible through AWS LoadBalancer

---

# Deployment Verification

## Verify Kubernetes Pods

```bash
kubectl get pods
```

## Verify Services

```bash
kubectl get svc
```

## Verify Nodes

```bash
kubectl get nodes
```

---

# Security Best Practices Implemented

* Jenkins credentials used instead of hardcoded passwords
* DockerHub credentials securely stored in Jenkins
* AWS access configured using AWS CLI
* Kubernetes deployment isolated within EKS cluster
* LoadBalancer used for secure external access
* Latest Java 21 used for Jenkins compatibility

---

# Error Handling and Troubleshooting

## Common Issues Handled

### Java Compatibility Issue

Resolved by installing Java 21.

### Docker Permission Issue

```bash
sudo usermod -aG docker jenkins
```

### EKS Cluster Creation Failure

Resolved using:

```bash
--node-type t3.small
--nodes 1
```

### Kubernetes Access Issue

Verified using:

```bash
kubectl get nodes
```

---

# Screenshots

## Jenkins Dashboard

(Add Screenshot Here)

---

## Successful Jenkins Pipeline

(Add Screenshot Here)

---

## DockerHub Repository

(Add Screenshot Here)

---

## Kubernetes Pods Running

(Add Screenshot Here)

---

## Kubernetes Service / LoadBalancer

(Add Screenshot Here)

---

## Application Running on Browser

(Add Screenshot Here)

---

# Final Output

Application successfully deployed on AWS EKS and accessible through AWS LoadBalancer URL.

Example:

```text
CI/CD Pipeline on AWS EKS Successful
```

---

# Key Achievements

✅ Complete CI/CD Pipeline Automation
✅ GitHub Integration with Jenkins
✅ Docker Image Build and Push Automation
✅ AWS EKS Kubernetes Deployment
✅ Kubernetes LoadBalancer Integration
✅ Groovy-Based Jenkins Pipeline
✅ Real Production-Style DevOps Workflow
✅ Automated Continuous Deployment

---

# Future Enhancements

* Helm Charts Integration
* ArgoCD GitOps Deployment
* SonarQube Code Scanning
* Trivy Container Security Scanning
* Prometheus Monitoring
* Grafana Dashboards
* Slack Notifications
* HTTPS with Ingress Controller

---

# Conclusion

This project successfully demonstrates the implementation of a scalable and automated CI/CD pipeline using GitHub, Jenkins, DockerHub, and AWS EKS.

The entire software delivery lifecycle was automated, reducing manual intervention and ensuring faster, reliable, and consistent deployments in a cloud-native Kubernetes environment.

---

# Author

Sagar Salve
DevOps Engineer

