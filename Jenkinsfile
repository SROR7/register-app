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
        JENKINS_API_TOKEN = credentials('JENKINS_API_TOKEN')
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

                    def imageTag = "${RELEASE}-${env.BUILD_NUMBER}"

                    def dockerImage = docker.build("${DOCKER_IMAGE}:${imageTag}")

                    docker.withRegistry('https://index.docker.io/v1/', 'docker') {
                        dockerImage.push(imageTag)
                        dockerImage.push("latest")
                    }

                    env.IMAGE_TAG = imageTag
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh """
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image \
                    ${DOCKER_IMAGE}:${IMAGE_TAG} \
                    --no-progress --scanners vuln \
                    --exit-code 0 \
                    --severity HIGH,CRITICAL \
                    --format table
                    """
                }
            }
        }

        stage('Cleanup') {
            steps {
                script {
                    sh "docker rmi ${DOCKER_IMAGE}:${IMAGE_TAG} || true"
                    sh "docker rmi ${DOCKER_IMAGE}:latest || true"
                }
            }
        }

        stage("Trigger CD Pipeline") {
            steps {
                script {
                    sh """
                    curl -v -k \
                    --user SROR:${JENKINS_API_TOKEN} \
                    -X POST \
                    -H 'cache-control: no-cache' \
                    -H 'content-type: application/x-www-form-urlencoded' \
                    --data 'IMAGE_TAG=${IMAGE_TAG}' \
                    'http://ec2-13-60-104-242.eu-north-1.compute.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'
                    """
                }
            }
        }
    }
}
