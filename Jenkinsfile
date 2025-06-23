pipeline {
    agent none

    environment {
        // Thông tin chung
        SSH_USER = 'TaiKhau'
        DEPLOY_DIR = "/home/TaiKhau/app"

        // IP của GCP VMs
        GCP_VM_DEV = '34.126.143.226'
        GCP_VM_PROD = '34.143.235.29'

        // Docker Hub
        DOCKER_HUB_CREDS = credentials('dockerhub-credentials')
        DOCKER_HUB_USERNAME = "${DOCKER_HUB_CREDS_USR}"
        IMAGE_NAME = "${DOCKER_HUB_USERNAME}/aspnetapp"

        // GitHub Token
        GITHUB_TOKEN = credentials('github-token')
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout') {
            agent { label 'agent1' }
            steps {
                // Lấy nhánh hiện tại
                script {
                    env.CURRENT_BRANCH = env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'dev'
                }

                // Clone với token an toàn
                git branch: "${env.CURRENT_BRANCH}", url: "https://Qu11et:${GITHUB_TOKEN}@github.com/Qu11et/aspnetapp.git"

                sh 'ls -la'
                sh 'pwd'
            }
        }

        stage('Build') {
            agent { label 'agent1' }
            steps {
                script {
                    //def arch = ''
                    sh """
                    docker buildx build \
                        --platform linux/amd64 \
                        --build-arg TARGETARCH=linux-x64 \
                        -t ${IMAGE_NAME}:${env.BUILD_NUMBER} \
                        -f Dockerfile .
                    """
                }
            }
        }

        stage('Test') {
            agent { label 'agent1' }
            steps {
                sh """
                docker buildx build \
                    --platform linux/amd64 \
                    --build-arg TARGETARCH=linux-x64 \
                    -f Dockerfile.test .
                """
            }
        }

        stage('Push to Docker Hub') {
            agent { label 'agent1' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker push ${IMAGE_NAME}:${env.BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Deploy to Dev') {
            agent { label 'agent1' }
            environment {
                CONTAINER_PORT = '8081'
            }
            when {
                branch 'dev'
            }
            steps {
                withCredentials([file(credentialsId: 'ssh-private-key-file', variable: 'SSH_KEY')]) {
                    script {
                        sh """
                        ssh -i $SSH_KEY -o StrictHostKeyChecking=no $SSH_USER@${GCP_VM_DEV} << EOF
set -e
trap 'echo "[ERROR] Deployment failed on \$HOSTNAME!" >&2; exit 1' ERR

echo "[INFO] Switching to deployment directory..."
mkdir -p $DEPLOY_DIR && cd $DEPLOY_DIR

echo "[INFO] Pulling latest Docker image..."
docker pull ${IMAGE_NAME}:${BUILD_NUMBER}

echo "[INFO] Restarting container..."
docker stop aspnetapp || true
docker rm aspnetapp || true
docker run -d -p ${CONTAINER_PORT}:8080 --name aspnetapp ${IMAGE_NAME}:${BUILD_NUMBER}

echo "[SUCCESS] Dev Deployment complete on \$HOSTNAME"
EOF
                        """
                    }
                }
            }
        }

        stage('Deploy to Prod') {
            agent { label 'agent1' }
            environment {
                CONTAINER_PORT = '8080'
            }
            when {
                branch 'main'
            }
            steps {
                input message: "Bạn có chắc muốn deploy lên môi trường Production?"
                withCredentials([file(credentialsId: 'ssh-private-key-file', variable: 'SSH_KEY')]) {
                    script {
                        sh """
                        ssh -i $SSH_KEY -o StrictHostKeyChecking=no $SSH_USER@${GCP_VM_PROD} << EOF
set -e
trap 'echo "[ERROR] Deployment failed on \$HOSTNAME!" >&2; exit 1' ERR

echo "[INFO] Switching to deployment directory..."
mkdir -p $DEPLOY_DIR && cd $DEPLOY_DIR

echo "[INFO] Pulling latest Docker image..."
docker pull ${IMAGE_NAME}:${BUILD_NUMBER}

echo "[INFO] Restarting container..."
docker stop aspnetapp || true
docker rm aspnetapp || true
docker run -d -p ${CONTAINER_PORT}:8080 --name aspnetapp ${IMAGE_NAME}:${BUILD_NUMBER}

echo "[SUCCESS] Prod Deployment complete on \$HOSTNAME"
EOF
                        """
                    }
                }
            }
        }
    }
}
