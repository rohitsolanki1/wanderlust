pipeline {
    agent any
    
    environment {
        REGISTRY = "localhost:32100"   // 🔁 replace with your HOST IP (NOT localhost)
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
                    echo "VITE_API_PATH=http://localhost:31100" > .env.docker
                    """
                }
            }
        }

        stage("Docker: Build Images"){
            steps{
                sh """
                docker build -t ${REGISTRY}/wanderlust-backend:${DOCKER_TAG} ./backend
                docker tag ${REGISTRY}/wanderlust-backend:${DOCKER_TAG} ${REGISTRY}/wanderlust-backend:latest

                docker build -t ${REGISTRY}/wanderlust-frontend:${DOCKER_TAG} ./frontend
                docker tag ${REGISTRY}/wanderlust-frontend:${DOCKER_TAG} ${REGISTRY}/wanderlust-frontend:latest
                """
            }
        }
        
        stage("Docker: Push to Local Registry"){
            steps{
                sh """
                docker push ${REGISTRY}/wanderlust-backend:${DOCKER_TAG}
                docker push ${REGISTRY}/wanderlust-backend:latest

                docker push ${REGISTRY}/wanderlust-frontend:${DOCKER_TAG}
                docker push ${REGISTRY}/wanderlust-frontend:latest
                """
            }
        }
    }
}
