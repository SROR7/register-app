pipeline {
    agent { label 'jenkins-agent' }
    tools {
        jdk 'java17'
        maven 'Maven3'
    }
    environment {
        APP_NAME   = 'register-app-pipeline'
        RELEASE    = '1.0.0'
        DOCKER_USER  = 'sror'
        DOCKER_IMAGE = "sror/register-app-pipeline"
        JENKINS_API_TOKEN = credentials('JENKINS_API_TOKEN')
    }
    stages {

        stage('CleanUp Workspace') {
            steps {
                cleanWs()
            }
        }

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
                    // Build image tag once and store it
                    def imageTag = "${RELEASE}-${env.BUILD_NUMBER}"
                    env.IMAGE_TAG = imageTag

                    def dockerImage = docker.build("${DOCKER_IMAGE}:${imageTag}")
                    docker.withRegistry('https://index.docker.io/v1/', 'docker') {
                        dockerImage.push(imageTag)
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                // Single quotes — shell resolves $DOCKER_IMAGE and $IMAGE_TAG at runtime
                sh '''
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image \
                        $DOCKER_IMAGE:$IMAGE_TAG \
                        --no-progress \
                        --scanners vuln \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table
                '''
            }
        }

        stage('Cleanup') {
            steps {
                sh '''
                    docker rmi $DOCKER_IMAGE:$IMAGE_TAG || true
                    docker rmi $DOCKER_IMAGE:latest    || true
                '''
            }
        }

        stage('Trigger CD Pipeline') {
            steps {
                withCredentials([string(credentialsId: 'JENKINS_API_TOKEN', variable: 'API_TOKEN')]) {
                    sh '''
                        curl -v -k \
                            --user "SROR:${API_TOKEN}" \
                            -X POST \
                            -H "content-type: application/x-www-form-urlencoded" \
                            --data "IMAGE_TAG=${IMAGE_TAG}" \
                            "http://51.20.65.24:8080/job/Git-Ops-register-app/buildWithParameters?token=gitops-token"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "CI pipeline complete — image ${DOCKER_IMAGE}:${env.IMAGE_TAG} deployed"
        }
        failure {
            echo "CI pipeline failed"
        }
    }
}
