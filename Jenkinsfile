pipeline {
    agent any

    tools {
        jdk 'null'
        maven 'maven_3.6.3'
    }

    environment {
        DOCKERHUB_USER = "spkumardocker"
        DOCKERHUB_REPO = "java-aks-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_NAME = "${DOCKERHUB_USER}/${DOCKERHUB_REPO}"
    }

    stages {

        stage("Cloning code from Git Repository") {
            steps {
                git branch: 'main',
                url: 'https://github.com/sharabgithub/devsecops-project.git'
            }
        }

        stage("Maven Build") {
            steps {
                sh '''
                java -version
                mvn -version
                mvn clean package
                '''
            }
        }

        stage("SonarQube Code Scan") {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                    mvn sonar:sonar \
                      -Dsonar.projectKey=java-aks-app \
                      -Dsonar.projectName=java-aks-app
                    '''
                }
            }
        }

        stage("SonarQube Quality Gate") {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Build Docker Image") {
            steps {
                sh '''
                docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                '''
            }
        }

        stage("Trivy HTML Report") {
            steps {
                sh '''
                mkdir -p trivy-report

                trivy image \
                  --format template \
                  --template "@/usr/local/share/trivy/templates/html.tpl" \
                  --output trivy-report/trivy-report.html \
                  ${IMAGE_NAME}:${IMAGE_TAG} || true
                '''
            }
        }

        stage("Publish Trivy Report") {
            steps {
                publishHTML([
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'trivy-report',
                    reportFiles: 'trivy-report.html',
                    reportName: 'Trivy Security Report'
                ])
            }
        }

        stage("Trivy Security Gate") {
            steps {
                sh '''
                trivy image \
                  --ignore-unfixed \
                  --exit-code 0 \
                  --severity CRITICAL,HIGH \
                  ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage("DockerHub Login & Push") {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-cred',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                    echo "$DOCKER_PASS" | docker login \
                      -u "$DOCKER_USER" \
                      --password-stdin

                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest
                    '''
                }
            }
        }

        stage("Deploy to AKS") {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'aks-kubeconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                    mkdir -p ~/.kube
                    cp "$KUBECONFIG_FILE" ~/.kube/config
                    chmod 600 ~/.kube/config

                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml

                    kubectl set image deployment/java-aks-app \
                      java-aks-app=${IMAGE_NAME}:${IMAGE_TAG}

                    kubectl rollout status deployment/java-aks-app --timeout=120s

                    kubectl get pods -o wide
                    kubectl get svc
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline execution completed"
        }

        success {
            echo "CI/CD + DevSecOps pipeline completed successfully 🚀"
        }

        failure {
            echo "Pipeline failed. Check logs"
        }
    }
}