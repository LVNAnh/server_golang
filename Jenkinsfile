pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'lvnanh/server_golang'
        DOCKER_TAG = 'latest'
        PROD_SERVER = 'ec2-13-213-69-113.ap-southeast-1.compute.amazonaws.com'
        PROD_USER = 'ubuntu'
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'master', url: 'https://github.com/LVNAnh/server_golang.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image for linux/amd64 platform...'
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}", "--platform linux/amd64 .")
                }
            }
        }

        stage('Run Tests') {
            steps {
                echo 'Running tests...'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-credentials') {
                        docker.image("${DOCKER_IMAGE}:${DOCKER_TAG}").push()
                    }
                }
            }
        }

        stage('Deploy Golang to DEV') {
            steps {
                script {
                    echo 'Clearing server_golang-related images and containers...'
                    sh '''
                        docker container stop server-golang || echo "No container named server-golang to stop"
                        docker container rm server-golang || echo "No container named server-golang to remove"
                        docker image rm ${DOCKER_IMAGE}:${DOCKER_TAG} || echo "No image ${DOCKER_IMAGE}:${DOCKER_TAG} to remove"
                    '''

                    echo 'Deploying to DEV environment...'
                    sh '''
                        docker image pull ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker network create dev || echo "Network already exists"
                        docker container run -d --rm --name server-golang -p 3000:3000 --network dev ${DOCKER_IMAGE}:${DOCKER_TAG}
                    '''
                }
            }
        }

        stage('Deploy to Production on AWS') {
            steps {
                script {
                    echo 'Deploying to Production...'
                    sshagent(['aws-ssh-key']) {
                        sh '''
                            ssh -o StrictHostKeyChecking=no ${PROD_USER}@${PROD_SERVER} <<-EOF
                                docker container stop server-golang || echo "No container to stop"
                                docker container rm server-golang || echo "No container to remove"
                                docker image rm ${DOCKER_IMAGE}:${DOCKER_TAG} || echo "No image to remove"
                                docker image pull ${DOCKER_IMAGE}:${DOCKER_TAG} || { echo "Failed to pull image"; exit 1; }
                                docker container run -d --rm --name server-golang -p 3000:3000 ${DOCKER_IMAGE}:${DOCKER_TAG} || { echo "Failed to run container"; exit 1; }
                            EOF
                        '''
                    }
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
