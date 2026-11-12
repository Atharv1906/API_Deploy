pipeline {
    agent any

    environment {
        // ---- Edit these three values for your setup ----
        DOCKERHUB_USER   = 'your-dockerhub-username'
        IMAGE_NAME       = 'fastapi-products'
        EC2_HOST         = 'ubuntu@YOUR_EC2_PUBLIC_IP'
        // -------------------------------------------------
        IMAGE_TAG        = "${env.BUILD_NUMBER}"
        FULL_IMAGE       = "${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
        FULL_IMAGE_LATEST = "${DOCKERHUB_USER}/${IMAGE_NAME}:latest"
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 20, unit: 'MINUTES')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building image: ${FULL_IMAGE}"
                    docker build -t ${FULL_IMAGE} -t ${FULL_IMAGE_LATEST} .
                '''
            }
        }

        stage('Smoke Test Image') {
            steps {
                sh '''
                    docker rm -f smoke-test || true
                    docker run -d --name smoke-test -p 8001:8000 ${FULL_IMAGE}
                    sleep 5
                    curl -fsS http://localhost:8001/ || (docker logs smoke-test && exit 1)
                    docker rm -f smoke-test
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_PASS'
                )]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push ${FULL_IMAGE}
                        docker push ${FULL_IMAGE_LATEST}
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ${EC2_HOST} "
                            docker pull ${FULL_IMAGE_LATEST} &&
                            docker rm -f fastapi-products || true &&
                            docker run -d \
                                --name fastapi-products \
                                --restart unless-stopped \
                                -p 80:8000 \
                                ${FULL_IMAGE_LATEST} &&
                            docker image prune -f
                        "
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    PUBLIC_IP=$(echo ${EC2_HOST} | cut -d'@' -f2)
                    sleep 5
                    curl -fsS http://${PUBLIC_IP}/ && echo " -- deploy OK"
                '''
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f || true'
        }
        success {
            echo "Deployed ${FULL_IMAGE} successfully"
        }
        failure {
            echo "Pipeline failed — check logs"
        }
    }
}
