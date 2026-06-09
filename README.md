# Wanderlust App - Local Jenkins + Kubernetes + ArgoCD Deployment

This repository contains the Wanderlust MERN application and the DevOps pipeline for a local Kubernetes deployment with Jenkins, ArgoCD, a local Docker registry, and MongoDB hosted inside the cluster.


## Repo structure

- `Jenkinsfile` - main Jenkins pipeline for building images, pushing to registry, updating manifest tags, and enabling ArgoCD sync.
- `backend/` - Node.js backend application.
- `frontend/` - Vite + React frontend application.
- `kubernetes/` - Kubernetes manifests for backend, frontend, MongoDB, Redis, and cluster deployment.
- `argocd-application.yaml` - ArgoCD Application manifest syncing `kubernetes/`.
- `namespace.yaml` - namespace manifest for `wanderlust`.

## Deployment architecture

Your setup uses:
- Jenkins on a local machine or local host
- Kubernetes on the host cluster
- local Docker registry reachable from Jenkins and the cluster
- ArgoCD installed in Kubernetes for GitOps deployment
- MongoDB deployed inside Kubernetes
- Backend configured to connect to `mongodb://mongo-service/wanderlust`

## Prerequisites

1. Docker Desktop or Minikube running Kubernetes.
2. `kubectl` configured for your cluster.
3. Jenkins installed and able to run shell commands against the cluster.
4. ArgoCD installed on the cluster.
5. Local Docker registry accessible from Jenkins and Kubernetes.
6. MongoDB deployed in the `wanderlust` namespace.

## Step 1: Create namespace

```bash
kubectl apply -f namespace.yaml
```

## Step 2: Deploy MongoDB

The backend uses `MONGODB_URI="mongodb://mongo-service/wanderlust"` from `backend/.env.docker`.

Apply MongoDB manifest:

```bash
kubectl apply -f kubernetes/mongodb.yaml
```

Verify MongoDB is running:

```bash
kubectl get pods -n wanderlust
kubectl get svc -n wanderlust
```

Expected service names:
- `mongo-service`
- `backend-service`
- `frontend-service`

## Step 3: Deploy Redis (required by backend)

If your cluster needs Redis, apply the Redis manifest:

```bash
kubectl apply -f kubernetes/redis.yaml
```

Verify Redis:

```bash
kubectl get pods -n wanderlust
kubectl get svc -n wanderlust
```

## Step 4: Configure local Docker registry

This repo is configured for a local registry at `172.17.10.133:32100`.

Start a local registry:

```bash
docker run -d --name local-registry -p 32100:5000 --restart always registry:2
```

If Docker Desktop is used, add the registry host as an insecure registry:

- Docker Desktop → Settings → Docker Engine → add `"insecure-registries": ["172.17.10.133:32100"]`

## Step 5: Build and push Docker images

From the repository root:

```bash
docker build -t 172.17.10.133:32100/wanderlust-backend:latest ./backend
docker build -t 172.17.10.133:32100/wanderlust-frontend:latest ./frontend

docker push 172.17.10.133:32100/wanderlust-backend:latest
docker push 172.17.10.133:32100/wanderlust-frontend:latest
```

## Step 6: Deploy backend and frontend

```bash
kubectl apply -f kubernetes/backend.yaml
kubectl apply -f kubernetes/frontend.yaml
```

Verify deployments:

```bash
kubectl rollout status deployment/backend-deployment -n wanderlust
kubectl rollout status deployment/frontend-deployment -n wanderlust
kubectl get pods -n wanderlust
kubectl get svc -n wanderlust
```

## Step 7: Access the app locally

This repo uses NodePorts configured in the Kubernetes manifests:
- Frontend: `NodePort 31000`
- Backend: `NodePort 31100`

Open the frontend at:

```bash
http://localhost:31000
```

If the backend is needed directly:

```bash
http://localhost:31100
```

## Step 8: Configure ArgoCD

Apply the ArgoCD Application manifest:

```bash
kubectl apply -f argocd-application.yaml -n argocd
```

Sync the app using the ArgoCD CLI:

```bash
argocd app sync wanderlust-app
```

## Step 9: Configure Jenkins pipeline

Create a Jenkins pipeline job using this repository and select `Jenkinsfile`.

Ensure Jenkins has access to:
- the Git repository
- the local Docker registry at `172.17.10.133:32100`
- `kubectl` credentials for the cluster
- Git push permissions for manifest updates

## MongoDB access and commands

### 1. Verify MongoDB service and connect from inside the cluster

```bash
kubectl exec -n wanderlust deployment/mongo-deployment -- mongo --eval 'db.getMongo().getDBs()'
```

### 2. Open an interactive Mongo shell

```bash
kubectl exec -it -n wanderlust deployment/mongo-deployment -- mongo
```

Then run inside the shell:

```js
use wanderlust
show collections
db.posts.find().pretty()
db.users.find().pretty()
```

### 3. Port-forward MongoDB to localhost

```bash
kubectl port-forward -n wanderlust svc/mongo-service 27017:27017
```

Then connect from your local machine:

```bash
mongo --host localhost --port 27017
```

### 4. Example MongoDB commands

```js
use wanderlust
show collections
db.posts.find().pretty()
db.users.find().pretty()
db.posts.insertOne({ title: 'test', description: 'ok' })
db.posts.updateOne({ title: 'test' }, { $set: { title: 'updated' } })
db.posts.deleteOne({ title: 'updated' })
```

## Backend MongoDB configuration

In `backend/.env.docker`, the backend is configured with:

```bash
MONGODB_URI="mongodb://mongo-service/wanderlust"
REDIS_URL="redis://redis-service:6379"
PORT=8080
```

The backend image copies `.env.docker` to `.env` during build.

## Validation

Make sure all core components are ready:

```bash
kubectl get pods -n wanderlust
kubectl get svc -n wanderlust
kubectl get deployment -n wanderlust
```

If the backend or frontend fails to start, inspect logs:

```bash
kubectl logs -n wanderlust deployment/backend-deployment
kubectl logs -n wanderlust deployment/frontend-deployment
```

## Notes for your resume

- Local deployment uses Jenkins + Kubernetes + ArgoCD + MongoDB.
- The repo now contains a clean local GitOps path with a single `Jenkinsfile`.
- MongoDB is configured inside Kubernetes and the backend connects using the service `mongo-service`.
- The README includes exact commands for deploying and accessing MongoDB.
