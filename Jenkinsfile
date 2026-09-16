pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        ACR_SERVER       = 'democontainerregi.azurecr.io'
        IMAGE_NAME       = 'petclinic'
        IMAGE_TAG        = "${BUILD_NUMBER}"
        DEPLOYMENT_NAME  = 'petclinic'
        EMAIL_FROM       = 'bkrraj2021@gmail.com'
        EMAIL_RECIPIENTS = 'bkrraj@gmail.com'
    }

    stages {

        stage('Checkout from Git') {
            steps {
                git branch: 'prod',
                    url: 'https://github.com/bkrrajmali/myown-springboot.git'
            }
        }

        stage('Validate with Maven') {
            steps {
                sh 'mvn validate'
            }
        }

        stage('Compile with Maven') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test with Maven') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        mvn sonar:sonar \
                            -Dsonar.organization=bkrrajmali \
                            -Dsonar.projectKey=myown-springboot \
                            -Dsonar.projectName=myown-springboot \
                            -Dsonar.java.binaries=target/classes
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package with Maven') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }

        stage('Upload to Nexus') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: '192.168.0.228:8081',
                    groupId: 'org.springframework.samples',
                    version: '6.2.5',
                    repository: 'maven-releases',
                    credentialsId: 'nexus-creds',
                    artifacts: [
                        [
                            artifactId: 'spring-framework-petclinic',
                            classifier: '',
                            file: 'target/petclinic.war',
                            type: 'war'
                        ],
                        [
                            artifactId: 'spring-framework-petclinic',
                            classifier: '',
                            file: 'pom.xml',
                            type: 'pom'
                        ]
                    ]
                )
            }
        }

        stage('Trivy Artifact Scan') {
            steps {
                sh '''
                    trivy fs \
                        --scanners vuln \
                        --format table \
                        -o trivy-war-report.txt \
                        .
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-war-report.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t $ACR_SERVER/$IMAGE_NAME:$IMAGE_TAG \
                                 -t $ACR_SERVER/$IMAGE_NAME:latest .
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image \
                        --severity HIGH,CRITICAL \
                        --ignore-unfixed \
                        --exit-code 0 \
                        --format table \
                        -o trivy-image-report.txt \
                        $ACR_SERVER/$IMAGE_NAME:$IMAGE_TAG
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-image-report.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Push to ACR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'acr-creds',
                                                  usernameVariable: 'ACR_USER',
                                                  passwordVariable: 'ACR_PASS')]) {
                    sh '''
                        echo "$ACR_PASS" | docker login $ACR_SERVER -u "$ACR_USER" --password-stdin
                        docker push $ACR_SERVER/$IMAGE_NAME:$IMAGE_TAG
                        docker push $ACR_SERVER/$IMAGE_NAME:latest
                        docker logout $ACR_SERVER
                    '''
                }
            }
        }

        stage('Deploy to AKS') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        sed -i "s|$IMAGE_NAME:latest|$IMAGE_NAME:$IMAGE_TAG|g" k8s/deployment.yaml
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                    '''
                }
            }
        }

        stage('Verify Deployment Rollout') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl rollout status deployment/$DEPLOYMENT_NAME --timeout=300s
                        kubectl get pods -l app=$IMAGE_NAME
                        kubectl get svc
                    '''
                }
            }
        }
    }

    post {
        success {
            script {
                echo "Deployment verified successfully. Sending success email via Brevo API."
                withCredentials([string(credentialsId: 'brevo-api-key', variable: 'BREVO_API_KEY')]) {
                    sh """
                    curl --fail -s -X POST https://api.brevo.com/v3/smtp/email \\
                      -H "api-key: \$BREVO_API_KEY" \\
                      -H "Content-Type: application/json" \\
                      -d '{
                        "sender": {"email": "${EMAIL_FROM}"},
                        "to": [{"email": "${EMAIL_RECIPIENTS}"}],
                        "subject": "SUCCESS: Jenkins Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        "textContent": "Good news!\\n\\nThe pipeline ${env.JOB_NAME} build #${env.BUILD_NUMBER} completed successfully, and the deployment ${DEPLOYMENT_NAME} rolled out successfully to AKS.\\n\\nBuild URL: ${env.BUILD_URL}"
                      }'
                    """
                }
            }
        }
        failure {
            script {
                echo "Pipeline or deployment verification failed. Sending failure email via Brevo API."
                withCredentials([string(credentialsId: 'brevo-api-key', variable: 'BREVO_API_KEY')]) {
                    sh """
                    curl --fail -s -X POST https://api.brevo.com/v3/smtp/email \\
                      -H "api-key: \$BREVO_API_KEY" \\
                      -H "Content-Type: application/json" \\
                      -d '{
                        "sender": {"email": "${EMAIL_FROM}"},
                        "to": [{"email": "${EMAIL_RECIPIENTS}"}],
                        "subject": "FAILED: Jenkins Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        "textContent": "The pipeline ${env.JOB_NAME} build #${env.BUILD_NUMBER} FAILED.\\n\\nThis could be due to a build/deploy step failing, or the deployment ${DEPLOYMENT_NAME} failing to roll out successfully in AKS (check the Verify Deployment Rollout stage logs).\\n\\nBuild URL: ${env.BUILD_URL}\\nConsole Log: ${env.BUILD_URL}console"
                      }'
                    """
                }
            }
        }
        always {
            sh 'docker image prune -f || true'
            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }
    }
}