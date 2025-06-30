pipeline {
    agent none

    environment {
        // Thông tin chung
        SSH_USER = 'TaiKhau'
        DEPLOY_DIR = "/home/TaiKhau/app"

        // IP của GCP VMs
        GCP_VM_DEV = '34.142.138.182'
        GCP_VM_PROD = '34.142.133.202'

        // Docker Hub
        DOCKER_HUB_CREDS = credentials('dockerhub-credentials')
        DOCKER_HUB_USERNAME = "${DOCKER_HUB_CREDS_USR}"
        IMAGE_NAME = "${DOCKER_HUB_USERNAME}/aspnetapp"

        // GitHub Token
        GITHUB_TOKEN = credentials('github-token')
    }

    // Remove explicit triggers - Jenkins will handle this through GitHub Branch Source Plugin
    // The plugin automatically detects pull requests and branch events

    stages {
        stage('Verify PR Context') {
            agent { label 'agent-builder' }
            steps {
                script {
                    // Get the target branch (where PR is going to)
                    env.TARGET_BRANCH = env.CHANGE_TARGET ?: env.BRANCH_NAME
                    
                    // Check if we're in a PR context
                    if (!env.CHANGE_ID) {
                        error "This pipeline should only run on pull requests"
                    }
                    
                    // Check if PR is targeting allowed branches
                    if (!(env.TARGET_BRANCH in ['develop', 'main'])) {
                        error "Pull requests are only allowed to develop or main branches"
                    }
                    
                    echo "Processing pull request #${env.CHANGE_ID} targeting ${env.TARGET_BRANCH}"
                }
            }
        }

        stage('Checkout') {
            agent { label 'agent-builder' }
            steps {
                // Clone with secure token
                git branch: "${env.TARGET_BRANCH}", url: "https://Qu11et:${GITHUB_TOKEN}@github.com/Qu11et/aspnetapp.git"

                sh 'ls -la'
                sh 'pwd'
            }
        }

        stage('Build') {
            agent { label 'agent-builder' }
            steps {
                script {
                    sh """
                    docker build --pull -t ${IMAGE_NAME}:${env.BUILD_NUMBER} .
                    """
                }
            }
        }

        stage('Test') {
            agent { label 'agent-builder' }
            steps {
                sh """
                docker build -t aspnetapp-test -f Dockerfile.test .
                docker images | grep aspnetapp-test
                """
            }
        }

        stage('Push to Docker Hub') {
            agent { label 'agent-builder' }
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
                CONTAINER_PORT = "${DEPLOY_PORT}"
            }
            when {
                allOf {
                    expression { env.CHANGE_ID != null } // Is PR
                    expression { env.TARGET_BRANCH == 'develop' } // PR targeting develop
                }
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
            agent { label 'agent2' }
            environment {
                CONTAINER_PORT = "${DEPLOY_PORT}"
            }
            when {
                allOf {
                    expression { env.CHANGE_ID != null } // Is PR
                    expression { env.TARGET_BRANCH == 'main' } // PR targeting main
                }
            }
            steps {
                input message: "Do you want to proceed with Production deployment?"
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