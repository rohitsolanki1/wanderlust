pipeline {
    agent any
    
    stages {

        stage("Workspace cleanup"){
            steps{
                cleanWs()
            }
        }
        
        stage('Git: Code Checkout') {
            steps {
                git branch: 'main',
                    url: 'git@github.com:rohitsolanki1/Devops-Mega-Project-Jenkins-ArgoCD-EKS.git',
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

        // ✅ Keep env generation (needed for your Dockerfile)
        stage("Create Env File"){
            steps{
                dir('frontend'){
                    sh '''
                    echo "VITE_API_URL=https://your-api" > .env.docker
                    '''
                }
            }
        }

        stage("Docker: Build Images"){
            steps{
                sh """
                docker build -t thatgeekcontainer/wanderlust-backend-beta:${DOCKER_TAG} ./backend
                docker build -t thatgeekcontainer/wanderlust-frontend-beta:${DOCKER_TAG} ./frontend
                """
            }
        }
        
        stage("Docker: Push to DockerHub"){
            steps{
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh """
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push thatgeekcontainer/wanderlust-backend-beta:${DOCKER_TAG}
                    docker push thatgeekcontainer/wanderlust-frontend-beta:${DOCKER_TAG}
                    """
                }
            }
        }
    }
}
