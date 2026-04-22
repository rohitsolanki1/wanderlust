pipeline {
    agent any
    
    environment {
        REGISTRY = "localhost:32100"   // 🔁 replace with your HOST IP
        REPO = "rohitsolanki1/Devops-Mega-Project-Jenkins-ArgoCD-EKS.git"
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
                    url: "git@github.com:${REPO}",
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

        // ✅ FIXED: use service URL instead of localhost
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

        // ✅ NEW: Update Kubernetes manifests
        stage("Update K8s Manifests"){
            steps{
                sh """
                sed -i 's|wanderlust-backend:.*|wanderlust-backend:${DOCKER_TAG}|' kubernetes/backend.yaml
                sed -i 's|wanderlust-frontend:.*|wanderlust-frontend:${DOCKER_TAG}|' kubernetes/frontend.yaml
                """
            }
        }

        // ✅ NEW: Push changes so ArgoCD can deploy
        stage("Push Changes to Git"){
            steps{
                withCredentials([usernamePassword(credentialsId: 'jenkins-github', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh """
                    git config user.email "jenkins@local"
                    git config user.name "jenkins"
                    git add kubernetes/backend.yaml kubernetes/frontend.yaml
                    git commit -m "Update image tag to ${DOCKER_TAG}" || true
                    git push https://$USER:$PASS@github.com/${REPO} HEAD:main
                    """
                }
            }
        }
    }
}
