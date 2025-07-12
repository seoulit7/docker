pipeline {
    agent any

    environment {
        TIME_ZONE = 'Asia/Seoul'
        PROFILE = 'local'
        AWS_CREDENTIAL_NAME = 'aws-key'
        DEPLOY_CREDENTIAL_NAME = 'deploy-ssh-key'
        REGION = "ap-northeast-2"
        ECR_PATH = '670246014570.dkr.ecr.ap-northeast-2.amazonaws.com'
        IMAGE_NAME = '670246014570.dkr.ecr.ap-northeast-2.amazonaws.com/board'
        DEPLOY_HOST = "15.164.229.6"
    }

    stages {
        stage('Pull Codes from Github') {
            steps {
                checkout scm
            }
        }

        stage('Build Codes by Gradle') {
            steps {
                sh './gradlew clean build'
            }
        }

        stage('Dockerizing Project by Dockerfile') {
            steps {
                sh """
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                    docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest
                """
            }
            post {
                success {
                    echo '성공: Docker 이미지 빌드 완료'
                }
                failure {
                    error '실패: Dockerizing 과정에서 오류 발생'
                }
            }
        }

        stage('Upload to AWS ECR') {
            steps {
                script {
                    docker.withRegistry("https://${ECR_PATH}", "ecr:${REGION}:${AWS_CREDENTIAL_NAME}") {
                        docker.image("${IMAGE_NAME}:${BUILD_NUMBER}").push()
                        docker.image("${IMAGE_NAME}:latest").push()
                    }
                }
            }
            post {
                success {
                    echo '성공: ECR 이미지 업로드 완료'
                }
                failure {
                    error '실패: ECR 업로드 중 오류 발생'
                }
            }
        }

        stage('Deploy to AWS EC2 VM') {
            steps {
                sshagent(credentials: [DEPLOY_CREDENTIAL_NAME]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${DEPLOY_HOST} '
                            aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_PATH};
                            docker pull ${IMAGE_NAME}:${BUILD_NUMBER};
                            docker stop board || true;
                            docker rm board || true;
                            docker run -d -p 80:8080 --name board ${IMAGE_NAME}:${BUILD_NUMBER};
                        '
                    """
                }
            }
        }
    }
}