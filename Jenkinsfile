pipeline {
    agent any

    environment {
        // Environment variables
        ECR_REPO = '905418319927.dkr.ecr.us-east-1.amazonaws.com/gitlab-app' // Your AWS ECR repository URL
        IMAGE_NAME = 'latest'
        SONAR_HOST_URL = 'http://34.224.95.219:9000/'
        SONAR_LOGIN = 'sqa_9eeff9fd6fe6cc1ba506f3053f98af90e1236886'
        AWS_REGION = 'us-east-1'
        EC2_IP = '34.224.95.219'
        SSH_KEY_PATH = 'ssh_key'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/Bhargava895/python-app.git'
            }
        }

        stage('Unit Testing') {
            steps {
                sh 'python3 -m unittest discover'
            }
        }

        stage('Code Coverage') {
            steps {
                sh 'coverage run -m unittest discover'
                sh 'coverage report'
                sh 'coverage xml'
            }
        }

        stage('Static Code Analysis with SonarQube') {
            environment {
                SONAR_SCANNER_HOME = tool 'SonarQubeScanner'
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=your_project_key \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dsonar.login=${SONAR_LOGIN}'''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "${IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh "docker build -t ${ECR_REPO}:${imageTag} ."
                }
            }
        }

        stage('Scan Docker Image with Trivy') {
            steps {
                script {
                    def imageTag = "${IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh "trivy image ${ECR_REPO}:${imageTag}"
                }
            }
        }

        stage('Push to AWS ECR') {
            steps {
                script {
                    def imageTag = "${IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}"
                    sh "docker push ${ECR_REPO}:${imageTag}"
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    def imageTag = "${IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh "ssh -i ${SSH_KEY_PATH} ec2-user@${EC2_IP} 'docker pull ${ECR_REPO}:${imageTag} && docker run -d --name ${IMAGE_NAME} ${ECR_REPO}:${imageTag}'"
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
