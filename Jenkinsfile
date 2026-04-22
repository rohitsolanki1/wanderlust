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
    }
}
