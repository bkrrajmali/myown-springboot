pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        ACR_SERVER = 'democontainerregi.azurecr.io'
        IMAGE_NAME = 'petclinic'
        IMAGE_TAG  = "latest"
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
                        target/
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
    }

    post {
        always {
            sh 'docker image prune -f || true'
        }
    }
}