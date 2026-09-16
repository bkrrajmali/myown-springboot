pipeline {
    agent any
    tools {
        maven 'maven'
    }
    
    stages {
        stage ('Checkout from Git') 
        {
            steps {
                git branch: 'prod' , url: 'https://github.com/bkrrajmali/myown-springboot.git'
            }
        }
        stage ('Validate with Maven') 
        {
            steps {
                sh 'mvn validate'
            }
        }
        stage ('Compile with Maven')
        {
            steps {
                sh 'mvn compile'
            }
        }
        stage ('Test with Maven')
        {
            steps {
                sh 'mvn test'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''mvn sonar:sonar \
                        -Dsonar.organization=bkrrajmali \
                        -Dsonar.projectKey=myown-springboot \
                        -Dsonar.projectName=myown-springboot \
                        -Dsonar.java.binaries=target/classes'''
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
                [artifactId: 'spring-framework-petclinic', classifier: '', file: 'target/petclinic.war', type: 'war'],
                [artifactId: 'spring-framework-petclinic', classifier: '', file: 'pom.xml', type: 'pom']
            ]
        )
    }
}
stage('Trivy Artifact Scan') {
    steps {
        sh 'trivy fs --scanners vuln --format table -o trivy-war-report.txt target/'
    }
}
    }
}