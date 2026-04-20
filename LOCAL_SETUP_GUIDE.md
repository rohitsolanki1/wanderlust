# Local Kubernetes Setup Guide - Complete DevOps Stack

## Complete Guide to Host Wanderlust Application on Local Machine with Jenkins, ArgoCD, SonarQube, and Kubernetes

This guide provides step-by-step instructions to set up the entire DevOps pipeline on your local machine using a local Kubernetes cluster (Minikube/Docker Desktop) along with Jenkins, SonarQube, and ArgoCD.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Local Kubernetes Setup](#local-kubernetes-setup)
3. [Docker Registry Setup](#docker-registry-setup)
4. [Jenkins Setup](#jenkins-setup)
5. [SonarQube Setup](#sonarqube-setup)
6. [ArgoCD Setup](#argocd-setup)
7. [Application Deployment](#application-deployment)
8. [CI/CD Pipeline Configuration](#cicd-pipeline-configuration)
9. [Monitoring Setup (Optional)](#monitoring-setup-optional)
10. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### System Requirements
- **OS**: Windows 10/11 with WSL2 enabled OR macOS or Linux
- **RAM**: Minimum 16GB (8GB minimum, but 16GB recommended)
- **Disk Space**: 50GB available
- **CPU**: 4+ cores recommended

### Required Software
- **Docker**: Latest version installed and running
- **kubectl**: Command-line tool for Kubernetes
- **Helm**: Package manager for Kubernetes (v3.x)
- **Git**: For cloning repositories

### Installation Verification
```bash
# Check Docker installation
docker --version
docker run hello-world

# Check kubectl installation
kubectl version --client

# Check Helm installation
helm version

# Check Git installation
git --version
```

---

## Local Kubernetes Setup

You have two options for local Kubernetes:

### Option 1: Docker Desktop (Recommended for Windows)

#### Step 1: Enable Kubernetes in Docker Desktop

1. Open **Docker Desktop** application
2. Go to **Settings** → **Kubernetes**
3. Check the box **"Enable Kubernetes"**
4. Click **"Apply & Restart"**
5. Wait for Kubernetes to start (2-3 minutes)

#### Step 2: Verify Kubernetes Installation

```bash
# Check cluster status
kubectl cluster-info

# Check nodes
kubectl get nodes

# Expected output:
# NAME             STATUS   ROLES           AGE   VERSION
# docker-desktop   Ready    control-plane   2m    v1.xx.x
```

#### Step 3: Set Resource Limits (Optional but Recommended)

```bash
# For Docker Desktop on Windows, edit config in Docker settings
# Set:
# - CPUs: 4
# - Memory: 8GB
# - Swap: 1GB
```

---

### Option 2: Minikube (Alternative)

#### Step 1: Install Minikube

```bash
# Download and install Minikube
# Windows (using chocolatey):
choco install minikube

# macOS (using brew):
brew install minikube

# Linux:
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

#### Step 2: Start Minikube Cluster

```bash
# Start cluster with sufficient resources
minikube start --cpus=4 --memory=8192 --disk-size=50g

# Enable ingress addon
minikube addons enable ingress

# Get Minikube IP (you'll need this later)
minikube ip
```

#### Step 3: Configure kubectl Context

```bash
# Minikube automatically configures kubectl
# Verify:
kubectl get nodes
```

---

## Docker Registry Setup

### Option 1: Docker Hub (Recommended for Beginners)

1. Create account at [Docker Hub](https://hub.docker.com)
2. Log in locally:
```bash
docker login
# Enter your Docker Hub username and password
```

### Option 2: Local Docker Registry

For completely offline setup, use a local registry container:

```bash
# Start local registry
docker run -d \
  --name local-registry \
  -p 5000:5000 \
  --restart always \
  registry:2

# Verify registry is running
curl http://localhost:5000/v2/

# Tag images for local registry
docker tag <image> localhost:5000/<image>

# Push to local registry
docker push localhost:5000/<image>

# Configure Docker Desktop to use insecure registry:
# For Docker Desktop: Settings → Docker Engine → add:
# "insecure-registries": ["localhost:5000"]
```

---

## Jenkins Setup

### Step 1: Create Jenkins Namespace

```bash
kubectl create namespace jenkins
kubectl config set-context --current --namespace=jenkins
```

### Step 2: Create Persistent Volume for Jenkins

Create a file named `jenkins-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv
  namespace: jenkins
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data/jenkins
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  volumeName: jenkins-pv
```

Apply it:
```bash
kubectl apply -f jenkins-pvc.yaml
```

### Step 3: Deploy Jenkins using Helm

```bash
# Add Jenkins Helm repository
helm repo add jenkinsci https://charts.jenkins.io
helm repo update

# Create Jenkins values file
cat > jenkins-values.yaml << 'EOF'
controller:
  adminPassword: "admin123"
  
  persistence:
    enabled: true
    existingClaim: jenkins-pvc
    size: 20Gi
  
  installPlugins:
    - kubernetes:latest
    - docker:latest
    - git:latest
    - ghprb:latest
    - blueocean:latest
    - sonar:latest
    - owasp-dependency-check:latest
    - pipeline-stage-view:latest
    - credentials:latest
    - credentials-binding:latest
    - email-ext:latest
    - greenballs:latest
  
  JCasC:
    enabled: true
    configScripts:
      basic-security: |
        jenkins:
          securityRealm:
            saml:
              binding: "urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
          disableRememberMe: false

agent:
  enabled: true
  defaultsProviderTemplate: ""
  podCapacity: 10
  image: "jenkins/inbound-agent"
  tag: "3.296-1"
EOF

# Install Jenkins
helm install jenkins jenkinsci/jenkins -f jenkins-values.yaml -n jenkins

# Wait for Jenkins to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=jenkins -n jenkins --timeout=300s
```

### Step 4: Access Jenkins

```bash
# Get Jenkins service IP (for minikube, use 'minikube service jenkins')
kubectl get svc -n jenkins

# For Docker Desktop, Jenkins runs on localhost:8080
# For Minikube, get IP with:
minikube service jenkins -n jenkins

# Get initial admin password
kubectl exec -it -n jenkins $(kubectl get pods -n jenkins -l app.kubernetes.io/name=jenkins -o jsonpath='{.items[0].metadata.name}') -- cat /run/secrets/additional/chart-admin-passwd
```

**Access Jenkins**:
- URL: `http://localhost:8080` (Docker Desktop) or `http://<minikube-ip>:8080`
- Username: `admin`
- Password: Use the password from above

### Step 5: Configure Jenkins Credentials

1. Go to **Jenkins Dashboard** → **Manage Jenkins** → **Credentials**
2. Add Docker Hub credentials:
   - Click **New credentials**
   - Kind: **Username with password**
   - Username: Your Docker Hub username
   - Password: Your Docker Hub password
   - ID: `docker-credentials`
3. Add Git credentials (if using private repo):
   - Kind: **SSH Key** or **Username with password**
   - ID: `git-credentials`

---

## SonarQube Setup

### Step 1: Create SonarQube Namespace

```bash
kubectl create namespace sonarqube
kubectl config set-context --current --namespace=sonarqube
```

### Step 2: Create PostgreSQL Database for SonarQube

Create `sonarqube-db.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
  namespace: sonarqube
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data/postgres
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: sonarqube
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  volumeName: postgres-pv
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: sonarqube
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        env:
        - name: POSTGRES_DB
          value: sonarqube
        - name: POSTGRES_USER
          value: sonarqube
        - name: POSTGRES_PASSWORD
          value: sonarqube123
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: sonarqube
spec:
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: postgres
```

Apply:
```bash
kubectl apply -f sonarqube-db.yaml
```

### Step 3: Deploy SonarQube

Create `sonarqube-deployment.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: sonarqube-pv
  namespace: sonarqube
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data/sonarqube
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sonarqube-pvc
  namespace: sonarqube
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  volumeName: sonarqube-pv
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sonarqube
  namespace: sonarqube
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sonarqube
  template:
    metadata:
      labels:
        app: sonarqube
    spec:
      containers:
      - name: sonarqube
        image: sonarqube:lts-community
        env:
        - name: sonar.jdbc.url
          value: jdbc:postgresql://postgres:5432/sonarqube
        - name: sonar.jdbc.username
          value: sonarqube
        - name: sonar.jdbc.password
          value: sonarqube123
        ports:
        - containerPort: 9000
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
        volumeMounts:
        - name: sonarqube-storage
          mountPath: /opt/sonarqube/data
        livenessProbe:
          httpGet:
            path: /api/system/health
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
      volumes:
      - name: sonarqube-storage
        persistentVolumeClaim:
          claimName: sonarqube-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: sonarqube
  namespace: sonarqube
spec:
  ports:
  - port: 9000
    targetPort: 9000
    nodePort: 30090
  type: NodePort
  selector:
    app: sonarqube
```

Apply:
```bash
kubectl apply -f sonarqube-deployment.yaml

# Wait for SonarQube to be ready (takes 2-5 minutes)
kubectl logs -f deployment/sonarqube -n sonarqube
```

### Step 4: Access SonarQube

```bash
# Get SonarQube service IP/Port
kubectl get svc -n sonarqube

# For Docker Desktop: http://localhost:30090
# For Minikube: http://<minikube-ip>:30090
```

**Initial Login**:
- Username: `admin`
- Password: `admin`

**Change Default Password**:
1. Log in with `admin/admin`
2. Go to **Administration** → **Security** → **Users**
3. Change admin password

**Create SonarQube Token**:
1. Go to **My Account** → **Security** → **Tokens**
2. Generate a token (name: `jenkins-token`)
3. Copy this token (needed for Jenkins integration)

---

## ArgoCD Setup

### Step 1: Create ArgoCD Namespace

```bash
kubectl create namespace argocd
```

### Step 2: Install ArgoCD

```bash
# Install ArgoCD using official manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be ready
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

### Step 3: Access ArgoCD UI

```bash
# Port forward to access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8443:443

# Or expose as LoadBalancer/NodePort
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

**Access URL**: `https://localhost:8443` (accept self-signed certificate)

### Step 4: Get Initial Password

```bash
# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d

# Username: admin
# Password: [from above command]
```

### Step 5: Change Default Password

```bash
# Using ArgoCD CLI
argocd account update-password \
  --account admin \
  --new-password "your-new-password" \
  --server localhost:8443 \
  --insecure

# Or use ArgoCD UI:
# Go to Settings → Accounts → admin → Change Password
```

---

## Application Deployment

### Step 1: Prepare Application for Local Deployment

#### Create Local Kubernetes Files

Create `local-k8s-deployment.yaml`:

```yaml
---
# MongoDB Deployment
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data/mongodb
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:latest
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: "admin"
        - name: MONGO_INITDB_ROOT_PASSWORD
          value: "admin123"
        volumeMounts:
        - name: mongodb-storage
          mountPath: /data/db
      volumes:
      - name: mongodb-storage
        persistentVolumeClaim:
          claimName: mongodb-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb
  namespace: default
spec:
  ports:
  - port: 27017
    targetPort: 27017
  selector:
    app: mongodb
---
# Redis Deployment
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data/redis
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:latest
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-storage
          mountPath: /data
      volumes:
      - name: redis-storage
        persistentVolumeClaim:
          claimName: redis-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: default
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
---
# Backend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: <your-docker-registry>/backend:latest
        # Replace with your image: docker.io/yourusername/backend:latest
        # or localhost:5000/backend:latest for local registry
        ports:
        - containerPort: 8080
        env:
        - name: MONGODB_URI
          value: "mongodb://admin:admin123@mongodb:27017/wanderlust?authSource=admin"
        - name: REDIS_URL
          value: "redis://redis:6379"
        - name: PORT
          value: "8080"
        - name: ENVIRONMENT
          value: "production"
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /api/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: default
spec:
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080
  type: NodePort
  selector:
    app: backend
---
# Frontend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: <your-docker-registry>/frontend:latest
        # Replace with your image: docker.io/yourusername/frontend:latest
        # or localhost:5000/frontend:latest for local registry
        ports:
        - containerPort: 80
        env:
        - name: VITE_API_URL
          value: "http://backend:8080"
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 300m
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: default
spec:
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30000
  type: NodePort
  selector:
    app: frontend
```

Save this file as `kubernetes/local-k8s-deployment.yaml`

### Step 2: Build Docker Images Locally

```bash
# Navigate to project root
cd /path/to/DevOps-Project

# Build backend image
docker build -t <your-docker-registry>/backend:latest ./backend
# Examples:
# docker build -t docker.io/yourusername/backend:latest ./backend
# docker build -t localhost:5000/backend:latest ./backend

# Build frontend image
docker build -t <your-docker-registry>/frontend:latest ./frontend
# docker build -t docker.io/yourusername/frontend:latest ./frontend
# docker build -t localhost:5000/frontend:latest ./frontend

# Verify images
docker images | grep -E 'backend|frontend'
```

### Step 3: Push Images to Registry

```bash
# For Docker Hub:
docker push <your-docker-registry>/backend:latest
docker push <your-docker-registry>/frontend:latest

# For local registry (localhost:5000):
docker push localhost:5000/backend:latest
docker push localhost:5000/frontend:latest

# Verify images in registry
curl http://localhost:5000/v2/_catalog  # For local registry
```

### Step 4: Update Kubernetes Manifests

In `kubernetes/local-k8s-deployment.yaml`, replace:
- `<your-docker-registry>/backend:latest` with your actual registry
- `<your-docker-registry>/frontend:latest` with your actual registry

### Step 5: Deploy Application

```bash
# Apply Kubernetes manifests
kubectl apply -f kubernetes/local-k8s-deployment.yaml

# Verify deployments
kubectl get deployments
kubectl get pods
kubectl get svc

# Check deployment status
kubectl describe deployment backend
kubectl describe deployment frontend

# View logs
kubectl logs -f deployment/backend
kubectl logs -f deployment/frontend
```

### Step 6: Access Application

```bash
# Get service information
kubectl get svc

# For Docker Desktop (localhost access):
# Frontend: http://localhost:30000
# Backend API: http://localhost:30080

# For Minikube:
minikube service frontend
minikube service backend
```

---

## CI/CD Pipeline Configuration

### Step 1: Create Jenkins Pipeline Job

1. Go to **Jenkins Dashboard**
2. Click **New Item**
3. Enter name: `wanderlust-ci-cd-pipeline`
4. Select **Pipeline**
5. Click **OK**

### Step 2: Configure Pipeline

In the Pipeline section, add the following Jenkinsfile script:

```groovy
pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = credentials('docker-credentials')
        DOCKER_IMAGE_BACKEND = 'yourusername/backend'
        DOCKER_IMAGE_FRONTEND = 'yourusername/frontend'
        // Use these for local registry:
        // DOCKER_IMAGE_BACKEND = 'localhost:5000/backend'
        // DOCKER_IMAGE_FRONTEND = 'localhost:5000/frontend'
        GIT_REPO = 'https://github.com/yourusername/wanderlust.git'
        SONAR_HOST = 'http://sonarqube:9000'
        SONAR_TOKEN = credentials('sonarqube-token')
    }

    stages {
        stage('Checkout') {
            steps {
                echo '========== Checking out code =========='
                git branch: 'main', url: '${GIT_REPO}'
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                echo '========== Running Trivy Filesystem Scan =========='
                sh '''
                    trivy filesystem . --severity HIGH,CRITICAL --exit-code 0
                '''
            }
        }

        stage('OWASP Dependency Check - Backend') {
            steps {
                echo '========== Running OWASP Dependency Check for Backend =========='
                sh '''
                    cd backend
                    dependency-check.sh --project "wanderlust-backend" --scan . --format HTML --out ../dependency-check-report.html
                    cd ..
                '''
            }
        }

        stage('OWASP Dependency Check - Frontend') {
            steps {
                echo '========== Running OWASP Dependency Check for Frontend =========='
                sh '''
                    cd frontend
                    dependency-check.sh --project "wanderlust-frontend" --scan . --format HTML --out ../dependency-check-report-frontend.html
                    cd ..
                '''
            }
        }

        stage('SonarQube Code Analysis - Backend') {
            steps {
                echo '========== Running SonarQube Analysis for Backend =========='
                sh '''
                    cd backend
                    sonar-scanner \
                        -Dsonar.projectKey=wanderlust-backend \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=${SONAR_HOST} \
                        -Dsonar.login=${SONAR_TOKEN}
                    cd ..
                '''
            }
        }

        stage('SonarQube Code Analysis - Frontend') {
            steps {
                echo '========== Running SonarQube Analysis for Frontend =========='
                sh '''
                    cd frontend
                    sonar-scanner \
                        -Dsonar.projectKey=wanderlust-frontend \
                        -Dsonar.sources=src \
                        -Dsonar.host.url=${SONAR_HOST} \
                        -Dsonar.login=${SONAR_TOKEN}
                    cd ..
                '''
            }
        }

        stage('Build Backend') {
            steps {
                echo '========== Building Backend Docker Image =========='
                sh '''
                    docker build -t ${DOCKER_IMAGE_BACKEND}:${BUILD_NUMBER} ./backend
                    docker tag ${DOCKER_IMAGE_BACKEND}:${BUILD_NUMBER} ${DOCKER_IMAGE_BACKEND}:latest
                '''
            }
        }

        stage('Build Frontend') {
            steps {
                echo '========== Building Frontend Docker Image =========='
                sh '''
                    docker build -t ${DOCKER_IMAGE_FRONTEND}:${BUILD_NUMBER} ./frontend
                    docker tag ${DOCKER_IMAGE_FRONTEND}:${BUILD_NUMBER} ${DOCKER_IMAGE_FRONTEND}:latest
                '''
            }
        }

        stage('Push Images to Registry') {
            steps {
                echo '========== Pushing Docker Images =========='
                sh '''
                    echo "${DOCKER_REGISTRY_PSW}" | docker login -u "${DOCKER_REGISTRY_USR}" --password-stdin
                    docker push ${DOCKER_IMAGE_BACKEND}:${BUILD_NUMBER}
                    docker push ${DOCKER_IMAGE_BACKEND}:latest
                    docker push ${DOCKER_IMAGE_FRONTEND}:${BUILD_NUMBER}
                    docker push ${DOCKER_IMAGE_FRONTEND}:latest
                '''
            }
        }

        stage('Deploy to ArgoCD') {
            steps {
                echo '========== Triggering ArgoCD Deployment =========='
                sh '''
                    # Update image tags in deployment manifests
                    sed -i "s|image: ${DOCKER_IMAGE_BACKEND}.*|image: ${DOCKER_IMAGE_BACKEND}:${BUILD_NUMBER}|g" kubernetes/local-k8s-deployment.yaml
                    sed -i "s|image: ${DOCKER_IMAGE_FRONTEND}.*|image: ${DOCKER_IMAGE_FRONTEND}:${BUILD_NUMBER}|g" kubernetes/local-k8s-deployment.yaml
                    
                    # Apply updated manifests (for manual deployment)
                    kubectl apply -f kubernetes/local-k8s-deployment.yaml
                    
                    # Or use ArgoCD CLI:
                    # argocd app sync wanderlust --server localhost:8443 --insecure
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                echo '========== Verifying Deployment =========='
                sh '''
                    kubectl rollout status deployment/backend -n default
                    kubectl rollout status deployment/frontend -n default
                    kubectl get pods -n default
                '''
            }
        }
    }

    post {
        success {
            echo '========== Build Successful =========='
        }
        failure {
            echo '========== Build Failed =========='
        }
        always {
            echo '========== Cleaning up =========='
            sh 'docker logout'
        }
    }
}
```

### Step 3: Advanced Options Configuration

Add these settings in Jenkins job configuration:

**Build Triggers**:
- ☑ GitHub hook trigger for GITScm polling
- OR: Poll SCM - `H H * * *` (daily)

**Pipeline**:
- Definition: **Pipeline script from SCM**
- SCM: **Git**
- Repository URL: Your Git repo
- Branches to build: `main` (or your default branch)
- Script Path: `Jenkinsfile` (if in root) or `path/to/Jenkinsfile`

---

## Monitoring Setup (Optional)

### Step 1: Install Prometheus and Grafana

```bash
# Add Prometheus Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring

# Install Grafana
helm install grafana grafana/grafana -n monitoring
```

### Step 2: Access Grafana

```bash
# Get Grafana admin password
kubectl get secret -n monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode

# Port forward to Grafana
kubectl port-forward -n monitoring svc/grafana 3000:80

# Access: http://localhost:3000
# Username: admin
# Password: [from above]
```

### Step 3: Add Prometheus Data Source in Grafana

1. Go to **Configuration** → **Data Sources**
2. Click **Add data source**
3. Select **Prometheus**
4. URL: `http://prometheus-kube-prometheus-prometheus:9090`
5. Click **Save & Test**

### Step 4: Import Dashboards

1. Go to **Dashboards** → **Import**
2. Import ID: `6417` (Kubernetes Cluster Monitoring)
3. Select Prometheus as data source

---

## Troubleshooting

### Issue 1: Kubernetes Pods Not Starting

**Symptoms**: Pods stuck in `Pending` or `CrashLoopBackOff` state

**Solutions**:
```bash
# Check pod status
kubectl describe pod <pod-name>

# Check pod logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # For crashed pods

# Check node status
kubectl get nodes
kubectl describe node <node-name>

# Check resource usage
kubectl top nodes
kubectl top pods
```

### Issue 2: Docker Image Pull Failures

**Symptoms**: `ImagePullBackOff` errors

**Solutions**:
```bash
# Verify image exists in registry
docker image ls

# Check Docker credentials
kubectl get secrets -n <namespace>

# Create Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=docker.io \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email>

# Reference in deployment
imagePullSecrets:
  - name: regcred
```

### Issue 3: SonarQube Not Starting

**Symptoms**: SonarQube pod stuck in init state

**Solutions**:
```bash
# Check logs
kubectl logs -f deployment/sonarqube -n sonarqube

# Verify PostgreSQL is running
kubectl get pod -n sonarqube | grep postgres

# Check PostgreSQL connection
kubectl exec -it <postgres-pod> -n sonarqube -- \
  psql -U sonarqube -d sonarqube -h localhost

# Increase resource limits in deployment
# Set: resources.limits.memory: 2Gi
```

### Issue 4: Jenkins Cannot Connect to Kubernetes

**Symptoms**: Jenkins agents failing to spawn

**Solutions**:
```bash
# Create service account for Jenkins
kubectl create serviceaccount jenkins
kubectl create rolebinding jenkins --clusterrole=edit --serviceaccount=default:jenkins

# Get service account token
kubectl get secret $(kubectl get secret -o name | grep jenkins-token) -o jsonpath='{.data.token}' | base64 --decode

# Configure in Jenkins:
# Manage Jenkins → Manage Nodes and Clouds → Kubernetes
# Set Kubernetes URL: https://kubernetes.default.svc.cluster.local
# Disable certificate verification or add CA cert
```

### Issue 5: Cannot Access Services

**Symptoms**: Cannot reach frontend/backend from browser

**Solutions**:
```bash
# Check service health
kubectl get svc
kubectl describe svc <service-name>

# Test service connectivity from pod
kubectl exec -it <pod-name> -- curl http://<service-name>:<port>

# Port forward for local access
kubectl port-forward svc/<service-name> <local-port>:<container-port>

# For Minikube, use:
minikube service <service-name>

# Check Ingress (if using)
kubectl get ingress
kubectl describe ingress <ingress-name>
```

### Issue 6: Persistent Volume Issues

**Symptoms**: PVC stuck in `Pending` state

**Solutions**:
```bash
# Check PV and PVC status
kubectl get pv
kubectl get pvc

# Create directories for hostPath volumes (for local clusters)
mkdir -p /mnt/data/{mongodb,redis,jenkins,sonarqube,postgres}
sudo chmod -R 777 /mnt/data/

# For Minikube:
minikube ssh
sudo mkdir -p /mnt/data/{mongodb,redis,jenkins,sonarqube,postgres}
sudo chmod -R 777 /mnt/data/
```

### Issue 7: Jenkins Pipeline Build Failure

**Symptoms**: Build fails during Docker build/push stages

**Solutions**:
```bash
# Verify Docker daemon is accessible
docker ps

# Check Docker credentials in Jenkins
# Manage Jenkins → Credentials → System → Global Credentials

# Test Docker login manually
docker login -u <username> -p <password>
docker push <image>

# For local registry, ensure it's running
docker ps | grep registry

# Check Docker build logs
docker build -t <image> .
```

### Issue 8: ArgoCD Application Sync Failure

**Symptoms**: ArgoCD app stuck in `OutOfSync` state

**Solutions**:
```bash
# Check ArgoCD logs
kubectl logs -f deployment/argocd-application-controller -n argocd

# Manually sync application
argocd app sync <app-name> --server localhost:8443 --insecure

# Check application status
argocd app get <app-name>

# Verify Git repository connectivity
argocd repo list
argocd repo add <repo-url> --username <user> --password <pass>
```

### Issue 9: Out of Memory/Disk Space

**Symptoms**: Pods evicted or stuck in pending

**Solutions**:
```bash
# Check resource usage
kubectl top nodes
kubectl top pods

# Clean up unused resources
docker system prune -a
kubectl delete pod --field-selector status.phase=Failed
kubectl delete pod --field-selector status.phase=Succeeded

# Increase Docker Desktop resources
# Settings → Resources → CPU, Memory, Disk
```

### Issue 10: SonarQube Analysis Fails

**Symptoms**: Jenkins build fails in SonarQube stage

**Solutions**:
```bash
# Verify SonarQube is accessible from Jenkins pod
kubectl exec -it <jenkins-pod> -- curl http://sonarqube:9000

# Check SonarQube logs
kubectl logs -f deployment/sonarqube -n sonarqube

# Verify SonarQube token in Jenkins credentials
# Manage Jenkins → Credentials → sonarqube-token

# Test analysis manually
cd backend
sonar-scanner \
  -Dsonar.projectKey=wanderlust-backend \
  -Dsonar.sources=. \
  -Dsonar.host.url=http://localhost:30090 \
  -Dsonar.login=<token>
```

---

## Useful Commands Reference

```bash
# Kubernetes Commands
kubectl get pods -A                          # List all pods
kubectl get svc -A                           # List all services
kubectl logs -f <pod-name>                   # View pod logs
kubectl exec -it <pod-name> -- /bin/bash     # Access pod terminal
kubectl describe pod <pod-name>              # Details about pod
kubectl delete pod <pod-name>                # Delete pod
kubectl apply -f <file.yaml>                 # Apply configuration
kubectl delete -f <file.yaml>                # Delete configuration

# Docker Commands
docker ps                                     # List running containers
docker logs <container-id>                    # View container logs
docker build -t <image> .                    # Build image
docker push <image>                          # Push image to registry
docker run -it <image> /bin/bash              # Run container

# Jenkins
curl http://localhost:8080                    # Jenkins UI
# Default creds: admin / admin123

# SonarQube
curl http://localhost:30090                   # SonarQube UI
# Default creds: admin / admin

# ArgoCD
kubectl port-forward -n argocd svc/argocd-server 8443:443
# URL: https://localhost:8443
# Default creds: admin / [check secret]

# Helm Commands
helm repo add <repo> <url>                   # Add Helm repo
helm install <release> <chart>               # Install chart
helm upgrade <release> <chart>               # Upgrade release
helm uninstall <release>                     # Uninstall release
```

---

## Next Steps

1. **Test the complete pipeline**: Commit code → Jenkins builds → ArgoCD deploys
2. **Set up monitoring**: Use Prometheus + Grafana for cluster monitoring
3. **Configure backup**: Regular backup of PersistentVolumes
4. **Security hardening**: RBAC, network policies, pod security policies
5. **Load testing**: Use tools like JMeter or Locust to test application
6. **Documentation**: Document your setup and customizations

---

## Support & Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [SonarQube Documentation](https://docs.sonarqube.org/)
- [Docker Documentation](https://docs.docker.com/)

---

**Last Updated**: April 2026
**Status**: Complete Guide Ready for Local Setup
