pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/bhupatipradhan/spring-jenkins-poc.git'
            }
        }
        stage('Build') {
            steps {
                bat '''
                    :: Kill the old process first so Maven can delete the target folder
                    FOR /F "tokens=5" %%T IN ('netstat -ano ^| findstr :9090') DO taskkill /F /PID %%T || ver>nul
                    
                    mvn clean package -DskipTests
                '''
            }
        }
        stage('Test') {
            steps {
                bat 'mvn test'
            }
        }
        stage('Deploy') {
            environment {
                JENKINS_NODE_COOKIE = 'dontKillMe'
            }
            steps {
                bat '''
                    :: Start the new application
                    start java -jar target\\demo-0.0.1-SNAPSHOT.jar --server.port=9090
                '''
            }
        }
    }
}