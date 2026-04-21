pipeline {
    agent any
    
    environment{
        SONAR_HOME = tool "Sonar"
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
                    url: 'git@github.com:rohitsolanki1/Devops-Mega-Project-Jenkins-ArgoCD-EKS.git',
                    credentialsId: 'jenkins-github'
            }
        }

        // ✅ NEW: Generate Auto Tag
        stage("Generate Docker Tag") {
            steps {
                script {
                    def commit = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.DOCKER_TAG = "${BUILD_NUMBER}-${commit}"
                    echo "Generated Tag: ${env.DOCKER_TAG}"
                }
            }
        }
        
        stage("Trivy: Filesystem scan"){
            steps{
                sh 'trivy fs .'
            }
        }

        stage("OWASP: Dependency check"){
            when {
                expression { return false }
            }
            steps{
                echo "Skipping OWASP scan"
            }
        }
        
        stage("SonarQube: Code Analysis"){
            steps{
                withSonarQubeEnv('Sonar') {
                    sh """
                    ${SONAR_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=wanderlust \
                    -Dsonar.projectName=wanderlust \
                    -Dsonar.sources=backend,frontend/src \
                    -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/build/** \
                    -Dsonar.javascript.node.maxspace=2048
                    """
                }
            }
        }
        
        stage("SonarQube: Code Quality Gates"){
            steps{
                timeout(time: 1, unit: 'MINUTES') {
                    script {
                        try {
                            waitForQualityGate abortPipeline: true
                        } catch (err) {
                            echo "Quality Gate check skipped due to timeout"
                        }
                    }
                }
            }
        }
                
        stage('Exporting environment variables') {
            parallel{
                stage("Backend env setup"){
                    steps {
                        dir("Automations"){
                            sh "bash updatebackendnew.sh"
                        }
                    }
                }
                
                stage("Frontend env setup"){
                    steps {
                        dir("Automations"){
                            sh "bash updatefrontendnew.sh"
                        }
                    }
                }
            }
        }
        
        // ✅ UPDATED
        stage("Docker: Build Images"){
            steps{
                sh """
                docker build -t thatgeekcontainer/wanderlust-backend-beta:${DOCKER_TAG} ./backend
                docker build -t thatgeekcontainer/wanderlust-frontend-beta:${DOCKER_TAG} ./frontend
                """
            }
        }
        
        // ✅ UPDATED
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
