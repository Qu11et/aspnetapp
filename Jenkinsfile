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
        
        // Tên repo để gửi trạng thái build
        REPO_OWNER = 'Qu11et'  // Thay bằng owner repository của bạn
        REPO_NAME = 'aspnetapp' // Thay bằng tên repository của bạn
    }
    
    stages {
        stage('Checkout') {
            agent { label 'agent-builder' }
            steps {
                // Lấy nhánh hiện tại
                script {
                    // Xác định nếu đây là một Pull Request
                    env.IS_PR = env.CHANGE_ID ? true : false
                    
                    if (env.IS_PR) {
                        // Pull Request
                        env.CURRENT_BRANCH = env.CHANGE_BRANCH
                        echo "Processing Pull Request #${env.CHANGE_ID} from branch: ${env.CURRENT_BRANCH} to ${env.CHANGE_TARGET}"
                        
                        // Đánh dấu trạng thái bắt đầu build
                        step([
                            $class: 'GitHubCommitStatusSetter',
                            reposSource: [$class: 'ManuallyEnteredRepositorySource', url: "https://github.com/${REPO_OWNER}/${REPO_NAME}"],
                            contextSource: [$class: 'ManuallyEnteredCommitContextSource', context: 'Jenkins Pipeline'],
                            statusResultSource: [$class: 'ConditionalStatusResultSource', results: [
                                [$class: 'AnyBuildResult', message: 'Building', state: 'PENDING']
                            ]]
                        ])
                    } else {
                        // Branch thông thường
                        env.CURRENT_BRANCH = env.BRANCH_NAME
                        echo "Processing branch: ${env.CURRENT_BRANCH}"
                    }
                }

                checkout scm
                
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
                anyOf {
                    branch 'dev'
                    expression { return env.CHANGE_TARGET == 'dev' }
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
                anyOf {
                    branch 'main'
                    expression { return env.CHANGE_TARGET == 'main' }
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
    
    post {
        success {
            node('agent-builder') { 
                script {
                    if (env.CHANGE_ID) {
                        // Cập nhật trạng thái GitHub - Thành công
                        step([
                            $class: 'GitHubCommitStatusSetter',
                            reposSource: [$class: 'ManuallyEnteredRepositorySource', url: "https://github.com/${REPO_OWNER}/${REPO_NAME}"],
                            contextSource: [$class: 'ManuallyEnteredCommitContextSource', context: 'Jenkins Pipeline'],
                            statusResultSource: [$class: 'ConditionalStatusResultSource', results: [
                                [$class: 'AnyBuildResult', message: 'Build succeeded', state: 'SUCCESS']
                            ]]
                        ])
                    }
                }
            }
        }
        failure {
            node('agent-builder') { 
                script {
                    if (env.CHANGE_ID) {
                        // Cập nhật trạng thái GitHub - Thất bại
                        step([
                            $class: 'GitHubCommitStatusSetter',
                            reposSource: [$class: 'ManuallyEnteredRepositorySource', url: "https://github.com/${REPO_OWNER}/${REPO_NAME}"],
                            contextSource: [$class: 'ManuallyEnteredCommitContextSource', context: 'Jenkins Pipeline'],
                            statusResultSource: [$class: 'ConditionalStatusResultSource', results: [
                                [$class: 'AnyBuildResult', message: 'Build failed', state: 'FAILURE']
                            ]]
                        ])
                    }
                }
            }
        }
    }
}