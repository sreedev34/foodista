pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-cred'
        DOCKER_IMAGE          = 'sreedev34/foodista'
        SONAR_SERVER_NAME     = 'sonarqube_env' 
        SONAR_SCANNER_TOOL    = 'sonar-scanner'     
        GIT_CREDENTIALS_ID    = 'github-cred'
        MANIFEST_REPO         = 'https://github.com/sreedev34/foodista'
        MANIFEST_BRANCH       = 'main'
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool "${SONAR_SCANNER_TOOL}"
                    withSonarQubeEnv("${SONAR_SERVER_NAME}") {
                        sh "${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=foodista-key \
                            -Dsonar.projectName='foodista-project' \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=*/venv/,/.venv/,/_pycache_/,/.pytest_cache/,/static/*"
                    }
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin
                            
                            docker build -t ${DOCKER_IMAGE}:${env.BUILD_NUMBER} -t${DOCKER_IMAGE}:latest .
                            
                            docker push ${DOCKER_IMAGE}:${env.BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage('Update Manifest Repo') {
            steps {
                dir('manifest-workspace') {
                    sh "git clone ${MANIFEST_REPO} . || true"
                    withCredentials([usernamePassword(credentialsId: "${GIT_CREDENTIALS_ID}", usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                        sh """
                            git remote set-url origin https://\${GH_USER}:\${GH_TOKEN}@github.com/sreedev34/foodista.git
                            git pull --rebase origin ${MANIFEST_BRANCH}
                            
                            sed -i '/name: foodista/{n;s|image:.*|image: ${DOCKER_IMAGE}:${env.BUILD_NUMBER}|}' k8s/Deployment.yaml
                            
                            git config user.name "sreedev34"
                            git config user.email "sreedevpattiam@gmail.com"
                            git commit -am "chore(manifest): update foodista image tag to ${env.BUILD_NUMBER} [skip ci]"
                            git push origin ${MANIFEST_BRANCH}
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline executed successfully! fast-api-chatbot image built, pushed, and manifest repo updated.'
        }
        failure {
            echo 'Pipeline failed. Please review the stage logs above.'
        }
    }
}