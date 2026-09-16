pipeline {
    agent any
    tools {
        maven 'maven'
    }
    
    stages {
        stage ('Checkout from Git') 
        {
            steps {
                git branch: 'prod' , url: 'https://github.com/bkrrajmali/springbootjavapp.git'
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
    }
}