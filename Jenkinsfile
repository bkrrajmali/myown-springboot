// pipeline {
//     agent any

//     tools {
//         maven 'maven'
//     }

//     environment {
//         ACR_SERVER       = 'democontainerregi.azurecr.io'
//         IMAGE_NAME       = 'petclinic'
//         IMAGE_TAG        = "latest"
//         DEPLOYMENT_NAME  = 'petclinic'
//         EMAIL_FROM       = 'bkrraj2021@gmail.com'
//         EMAIL_RECIPIENTS = 'bkrraj@gmail.com'
//     }

//     stages {

//         stage('Checkout from Git') {
//             steps {
//                 git branch: 'prod',
//                     url: 'https://github.com/bkrrajmali/myown-springboot.git'
//             }
//         }

//         stage('Validate with Maven') {
//             steps {
//                 sh 'mvn validate'
//             }
//         }

//         stage('Compile with Maven') {
//             steps {
//                 sh 'mvn compile'
//             }
//         }

//         stage('Test with Maven') {
//             steps {
//                 sh 'mvn test'
//             }
//         }

//         stage('SonarQube Analysis') {
//             steps {
//                 withSonarQubeEnv('sonar-server') {
//                     sh '''
//                         mvn sonar:sonar \
//                             -Dsonar.organization=bkrrajmali \
//                             -Dsonar.projectKey=myown-springboot \
//                             -Dsonar.projectName=myown-springboot \
//                             -Dsonar.java.binaries=target/classes
//                     '''
//                 }
//             }
//         }

//         stage('Quality Gate') {
//             steps {
//                 timeout(time: 5, unit: 'MINUTES') {
//                     waitForQualityGate abortPipeline: true
//                 }
//             }
//         }

//         stage('Package with Maven') {
//             steps {
//                 sh 'mvn package -DskipTests'
//             }
//         }

//         stage('Upload to Nexus') {
//             steps {
//                 nexusArtifactUploader(
//                     nexusVersion: 'nexus3',
//                     protocol: 'http',
//                     nexusUrl: '192.168.0.228:8081',
//                     groupId: 'org.springframework.samples',
//                     version: '6.2.5',
//                     repository: 'maven-releases',
//                     credentialsId: 'nexus-creds',
//                     artifacts: [
//                         [
//                             artifactId: 'spring-framework-petclinic',
//                             classifier: '',
//                             file: 'target/petclinic.war',
//                             type: 'war'
//                         ],
//                         [
//                             artifactId: 'spring-framework-petclinic',
//                             classifier: '',
//                             file: 'pom.xml',
//                             type: 'pom'
//                         ]
//                     ]
//                 )
//             }
//         }

//         stage('Trivy Artifact Scan') {
//             steps {
//                 sh '''
//                     trivy fs \
//                         --scanners vuln \
//                         --format table \
//                         -o trivy-war-report.txt \
//                         .
//                 '''
//             }
//             post {
//                 always {
//                     archiveArtifacts artifacts: 'trivy-war-report.txt', allowEmptyArchive: true
//                 }
//             }
//         }

//         stage('Docker Build') {
//             steps {
//                 sh '''
//                     docker build -t $ACR_SERVER/$IMAGE_NAME:$IMAGE_TAG \
//                                  -t $ACR_SERVER/$IMAGE_NAME:latest .
//                 '''
//             }
//         }

//         stage('Trivy Image Scan') {
//             steps {
//                 sh '''
//                     trivy image \
//                         --severity HIGH,CRITICAL \
//                         --ignore-unfixed \
//                         --exit-code 0 \
//                         --format table \
//                         -o trivy-image-report.txt \
//                         $ACR_SERVER/$IMAGE_NAME:$IMAGE_TAG
//                 '''
//             }
//             post {
//                 always {
//                     archiveArtifacts artifacts: 'trivy-image-report.txt', allowEmptyArchive: true
//                 }
//             }
//         }

//         stage('Push to ACR') {
//             steps {
//                 withCredentials([usernamePassword(credentialsId: 'acr-creds',
//                                                   usernameVariable: 'ACR_USER',
//                                                   passwordVariable: 'ACR_PASS')]) {
//                     sh '''
//                         echo "$ACR_PASS" | docker login $ACR_SERVER -u "$ACR_USER" --password-stdin
//                         docker push $ACR_SERVER/$IMAGE_NAME:$IMAGE_TAG
//                         docker push $ACR_SERVER/$IMAGE_NAME:latest
//                         docker logout $ACR_SERVER
//                     '''
//                 }
//             }
//         }

//         stage('Create ACR Pull Secret') {
//             steps {
//                 withCredentials([usernamePassword(credentialsId: 'acr-creds',
//                                                   usernameVariable: 'ACR_USER',
//                                                   passwordVariable: 'ACR_PASS'),
//                                  file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
//                     sh '''
//                         kubectl create secret docker-registry acr-secret \
//                             --docker-server=$ACR_SERVER \
//                             --docker-username="$ACR_USER" \
//                             --docker-password="$ACR_PASS" \
//                             --dry-run=client -o yaml | kubectl apply -f -
//                     '''
//                 }
//             }
//         }

//         stage('Deploy to AKS') {
//             steps {
//                 withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
//                     sh '''
//                         sed -i "s|$IMAGE_NAME:latest|$IMAGE_NAME:$IMAGE_TAG|g" k8s/deployment.yaml
//                         kubectl apply -f k8s/deployment.yaml
//                         kubectl apply -f k8s/service.yaml
//                     '''
//                 }
//             }
//         }
//         stage('Approve Production Deploy') {
//             steps {
//                 timeout(time: 30, unit: 'MINUTES') {
//                 input message: "Deploy ${IMAGE_TAG} to production?", ok: 'Deploy'
//                     }
//                 }
//             }
//         stage('Verify Deployment Rollout') {
//             steps {
//                 withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
//                     sh '''
//                         kubectl rollout status deployment/$DEPLOYMENT_NAME --timeout=300s
//                         kubectl get pods -l app=$IMAGE_NAME
//                         kubectl get svc
//                     '''
//                 }
//             }
//         }
//     }

//     post {
//         success {
//             script {
//                 echo "Deployment verified successfully. Sending success email via Brevo API."
//                 withCredentials([string(credentialsId: 'brevo-api-key', variable: 'BREVO_API_KEY')]) {
//                     sh """
//                     curl --fail -s -X POST https://api.brevo.com/v3/smtp/email \\
//                       -H "api-key: \$BREVO_API_KEY" \\
//                       -H "Content-Type: application/json" \\
//                       -d '{
//                         "sender": {"email": "${EMAIL_FROM}"},
//                         "to": [{"email": "${EMAIL_RECIPIENTS}"}],
//                         "subject": "SUCCESS: Jenkins Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER}",
//                         "textContent": "Good news!\\n\\nThe pipeline ${env.JOB_NAME} build #${env.BUILD_NUMBER} completed successfully, and the deployment ${DEPLOYMENT_NAME} rolled out successfully to AKS.\\n\\nBuild URL: ${env.BUILD_URL}"
//                       }' || true
//                     """
//                 }
//             }
//         }
//         failure {
//             script {
//                 echo "Pipeline or deployment verification failed. Sending failure email via Brevo API."
//                 withCredentials([string(credentialsId: 'brevo-api-key', variable: 'BREVO_API_KEY')]) {
//                     sh """
//                     curl --fail -s -X POST https://api.brevo.com/v3/smtp/email \\
//                       -H "api-key: \$BREVO_API_KEY" \\
//                       -H "Content-Type: application/json" \\
//                       -d '{
//                         "sender": {"email": "${EMAIL_FROM}"},
//                         "to": [{"email": "${EMAIL_RECIPIENTS}"}],
//                         "subject": "FAILED: Jenkins Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER}",
//                         "textContent": "The pipeline ${env.JOB_NAME} build #${env.BUILD_NUMBER} FAILED.\\n\\nThis could be due to a build/deploy step failing, or the deployment ${DEPLOYMENT_NAME} failing to roll out successfully in AKS (check the Verify Deployment Rollout stage logs).\\n\\nBuild URL: ${env.BUILD_URL}\\nConsole Log: ${env.BUILD_URL}console"
//                       }' || true
//                     """
//                 }
//             }
//         }
//         always {
//             sh 'docker image prune -f || true'
//             echo "Pipeline finished with status: ${currentBuild.currentResult}"
//         }
//     }
// }

pipeline {
    agent any

    tools {
        maven 'maven'
    }

    options {
        timeout(time: 45, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }

    environment {
        ACR_SERVER       = 'democontainerregi.azurecr.io'
        IMAGE_NAME       = 'petclinic'
        IMAGE_TAG        = "latest"
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
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
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

        stage('Verify Artifact') {
            steps {
                sh '''
                    test -f target/petclinic.war || { echo "WAR not found"; exit 1; }
                    unzip -l target/petclinic.war | grep -q "WEB-INF/web.xml\\|WEB-INF/classes" \
                        || { echo "WAR looks malformed"; exit 1; }
                    ls -lh target/petclinic.war
                '''
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

        stage('Container Smoke Test') {
            steps {
                sh '''
                    docker rm -f petclinic-test 2>/dev/null || true
                    docker run -d --name petclinic-test -p 9999:8080 \
                        $ACR_SERVER/$IMAGE_NAME:$IMAGE_TAG

                    echo "Waiting for app to start..."
                    for i in $(seq 1 30); do
                        if curl -sf http://localhost:9999/ > /dev/null; then
                            echo "App responded after ${i} attempts"
                            break
                        fi
                        if [ $i -eq 30 ]; then
                            echo "App never became ready"
                            docker logs petclinic-test
                            exit 1
                        fi
                        sleep 5
                    done

                    echo "Checking key endpoints..."
                    curl -sf http://localhost:9999/ > /dev/null          || exit 1
                    curl -sf http://localhost:9999/vets.html > /dev/null || exit 1
                    curl -sf http://localhost:9999/owners/find > /dev/null || exit 1
                    echo "All endpoints OK"
                '''
            }
            post {
                always {
                    sh 'docker rm -f petclinic-test || true'
                }
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

        stage('Approve Production Deploy') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Deploy ${IMAGE_TAG} to production?", ok: 'Deploy'
                }
            }
        }

        stage('Create ACR Pull Secret') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'acr-creds',
                                                  usernameVariable: 'ACR_USER',
                                                  passwordVariable: 'ACR_PASS'),
                                 file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl create secret docker-registry acr-secret \
                            --docker-server=$ACR_SERVER \
                            --docker-username="$ACR_USER" \
                            --docker-password="$ACR_PASS" \
                            --dry-run=client -o yaml | kubectl apply -f -
                    '''
                }
            }
        }

        stage('Validate K8s Manifests') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl apply --dry-run=server -f k8s/deployment.yaml
                        kubectl apply --dry-run=server -f k8s/service.yaml
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
            post {
                failure {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                        sh '''
                            echo "Rollout failed - rolling back"
                            kubectl rollout undo deployment/$DEPLOYMENT_NAME || true
                            kubectl describe pods -l app=$IMAGE_NAME | tail -40
                        '''
                    }
                }
            }
        }

        stage('Post-Deploy Smoke Test') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        IP=$(kubectl get svc petclinic-service \
                             -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

                        if [ -z "$IP" ]; then
                            echo "No external IP assigned yet"
                            exit 1
                        fi
                        echo "Testing against http://$IP/"

                        for i in $(seq 1 20); do
                            if curl -sf http://$IP/ > /dev/null; then
                                echo "Live site responded"
                                break
                            fi
                            if [ $i -eq 20 ]; then
                                echo "Live site never responded"
                                exit 1
                            fi
                            sleep 15
                        done

                        curl -sf http://$IP/vets.html > /dev/null || exit 1
                        curl -sf http://$IP/owners/find > /dev/null || exit 1
                        echo "Production smoke test passed"
                    '''
                }
            }
            post {
                failure {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                        sh 'kubectl rollout undo deployment/$DEPLOYMENT_NAME || true'
                    }
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
                      }' || true
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
                      }' || true
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