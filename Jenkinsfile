pipeline {
    agent any
    stages {
        stage('Checkout') { steps { git branch: 'main', url: 'https://github.com/bhupatipradhan/spring-jenkins-poc.git' } }
        stage('Build')    { steps { sh 'mvn clean package -DskipTests' } }
        stage('Test')     { steps { sh 'mvn test' } }
        stage('Deploy')   { steps { sh 'nohup java -jar target/demo-0.0.1-SNAPSHOT.jar &' } }
    }
}
