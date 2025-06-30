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
        // Standard triggers that work with GitHub Branch Source Plugin
        // Automatically triggered by GitHub webhooks via the plugin
        // No explicit pullRequestBuildTrigger needed - handled by the plugin
        // This replaces pullRequestBuildTrigger with standard Jenkins pipeline triggers
    }

    stages {
        stage('Environment Setup') {
            agent { label 'agent-builder' }
            steps {
                script {
                    // Use built-in Jenkins environment variables for pull request detection
                    // CHANGE_ID is set by GitHub Branch Source Plugin for PR builds
                    // This replaces custom pullRequestBuildTrigger logic
                    if (env.CHANGE_ID) {
                        echo "This is a Pull Request build. PR ID: ${env.CHANGE_ID}"
                        echo "PR Target Branch: ${env.CHANGE_TARGET}"
                        echo "PR Source Branch: ${env.CHANGE_BRANCH}"
                        env.IS_PR = 'true'
                        env.CURRENT_BRANCH = env.CHANGE_BRANCH
                    } else {
                        echo "This is a regular branch build"
                        env.IS_PR = 'false'
                        env.CURRENT_BRANCH = env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'dev'
                    }
                    echo "Current branch: ${env.CURRENT_BRANCH}"
                    echo "Is PR: ${env.IS_PR}"
                }

                // GitHub Branch Source Plugin handles repository checkout automatically
                // Using 'checkout scm' for compatibility with the plugin
                checkout scm

                sh 'ls -la'
                sh 'pwd'
            }
        }

        stage('Build') {
            agent { label 'agent-builder' }
            steps {
                script {
                    //def arch = ''
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

        stage('PR Status Update') {
            agent { label 'agent-builder' }
            when {
                environment name: 'IS_PR', value: 'true'
            }
            steps {
                script {
                    echo "Pull Request build completed successfully!"
                    echo "PR ${env.CHANGE_ID}: ${env.CHANGE_TITLE}"
                    echo "Source: ${env.CHANGE_BRANCH} -> Target: ${env.CHANGE_TARGET}"
                    echo "Build and tests passed. Ready for review."
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
                    branch 'dev'
                    // Only deploy on actual branch builds, not PR builds
                    // This maintains the same deployment logic but excludes PRs
                    not { 
                        environment name: 'IS_PR', value: 'true' 
                    }
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
                    branch 'main'
                    // Only deploy on actual branch builds, not PR builds
                    // This maintains the same deployment logic but excludes PRs
                    not { 
                        environment name: 'IS_PR', value: 'true' 
                    }
                }
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
