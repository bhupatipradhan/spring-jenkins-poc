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
                bat 'mvnw.cmd clean package -DskipTests'
            }
        }
        stage('Test') {
            steps {
                bat 'mvnw.cmd test'
            }
        }
        stage('Deploy') {
            steps {
                bat 'start java -jar target\\demo-0.0.1-SNAPSHOT.jar'
            }
        }
    }
}