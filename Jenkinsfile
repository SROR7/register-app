pipeline {
    agent { label 'jenkins-agent' }

    tools {
        jdk 'java17'
        maven 'Maven3'
    }

    environment {
        APP_NAME = 'register-app-pipeline'
        RELEASE = '1.0.0'
        DOCKER_USER = 'sror'
        DOCKER_IMAGE = "${DOCKER_USER}/${APP_NAME}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'GitHub',
                    url: 'https://github.com/SROR7/register-app.git'
            }
        }

        stage('Build Application') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Test Application') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }

        stage('Build And Push Docker Image') {
            steps {
                script {

                    def docker_image = docker.build("${DOCKER_IMAGE}:${RELEASE}")

                    docker.withRegistry('https://index.docker.io/v1/', 'docker') {

                        docker_image.push("${RELEASE}")
                        docker_image.push("latest")
                    }
                }
            }
        }
    }
}
