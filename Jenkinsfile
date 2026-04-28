pipeline {
    agent any
    
    environment {
        REGISTRY = "localhost:32100"   // 🔁 replace with your actual HOST IP
    }
    
    stages {

        stage("Workspace cleanup"){
            steps{
                cleanWs()
            }
        }
        
        stage('Git: Code Checkout') {
            steps {
                git branch: 'main',
                    url: "git@github.com:rohitsolanki1/Devops-Mega-Project-Jenkins-ArgoCD-EKS.git",
                    credentialsId: 'jenkins-github'
            }
        }

        stage("Generate Docker Tag") {
            steps {
                script {
                    def commit = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.DOCKER_TAG = "${BUILD_NUMBER}-${commit}"
                    echo "Generated Tag: ${env.DOCKER_TAG}"
                }
            }
        }

        stage("Create Env File"){
            steps{
                dir('frontend'){
                    sh """
                    echo "VITE_API_PATH=localhost:31100" > .env.docker
                    """
                }
            }
        }

        stage("Docker: Build Images"){
            steps{
                sh """
                docker build -t ${REGISTRY}/wanderlust-backend:${DOCKER_TAG} ./backend
                docker build -t ${REGISTRY}/wanderlust-frontend:${DOCKER_TAG} ./frontend
                """
            }
        }
        
        stage("Docker: Push to Local Registry"){
            steps{
                sh """
                docker push ${REGISTRY}/wanderlust-backend:${DOCKER_TAG}
                docker push ${REGISTRY}/wanderlust-frontend:${DOCKER_TAG}
                """
            }
        }

        stage("Update Kubernetes Manifests") {
            steps {
                sh """
                sed -i 's|image: 172.17.10.133:32100/wanderlust-backend:latest|image: ${REGISTRY}/wanderlust-backend:${DOCKER_TAG}|g' kubernetes/backend.yaml
                sed -i 's|image: 172.17.10.133:32100/wanderlust-frontend:latest|image: ${REGISTRY}/wanderlust-frontend:${DOCKER_TAG}|g' kubernetes/frontend.yaml
                """
            }
        }

        stage("Commit and Push Manifests") {
            steps {
                sh """
                git config user.name "rohitsolanki1"
                git config user.email "aryaveer.rohit@gmail.com"
                git add kubernetes/
                git commit -m "Update image tags to ${DOCKER_TAG}" || echo "No changes to commit"
                git push origin main
                """
            }
        }
    }
}
