pipeline {
    agent none

    environment {
        // Thông tin chung (giữ nguyên)
        SSH_USER = 'TaiKhau'
        DEPLOY_DIR = "/home/TaiKhau/app"
        GCP_VM_DEV = '34.142.138.182'
        GCP_VM_PROD = '34.142.133.202'
        DOCKER_HUB_CREDS = credentials('dockerhub-credentials')
        DOCKER_HUB_USERNAME = "${DOCKER_HUB_CREDS_USR}"
        IMAGE_NAME = "${DOCKER_HUB_USERNAME}/aspnetapp"
        GITHUB_TOKEN = credentials('github-token')
        
        // Tên repo để gửi trạng thái build
        REPO_OWNER = 'Qu11et'
        REPO_NAME = 'aspnetapp'
    }

    stage('Validate Branch Name Convention') {
        when {
            expression { env.CHANGE_ID != null } // Chỉ kiểm tra khi là Pull Request
        }
        steps {
            script {
            def branchName = env.CHANGE_BRANCH
            echo "🔍 Checking branch name: ${branchName}"

            // Quy tắc:
            // - Bắt đầu với: feature/, bugfix/, hotfix/, refactor/
            // - Sau đó là chuỗi lowercase chữ số và dấu -
            // - Không có -- hoặc kết thúc bằng -
            def regex = ~/^(feature|bugfix|hotfix|refactor)\/(issue-\d+-)?[a-z0-9]+(-[a-z0-9]+)*$/

            if (!(branchName ==~ regex) && !(branchName in ['main', 'dev'])) {
                echo "👉 Quy tắc: 'feature/issue-123-new-login' (lowercase, số, dấu -, không kết thúc bằng -)"
                error "❌ Branch name '${branchName}' không hợp lệ theo convention!"
                // Không dừng pipeline, chỉ cảnh báo
            } else {
                echo "✅ Branch name is valid!"
            }
            }
        }
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
                        
                        // Lưu commit SHA vào biến môi trường
                        checkout scm
                        env.GIT_COMMIT = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                        echo "Commit SHA: ${env.GIT_COMMIT}"
                        
                        // Đánh dấu trạng thái bắt đầu build bằng GitHub Status API trực tiếp
                        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_ACCESS_TOKEN')]) {
                            sh """
                            curl -H "Authorization: token ${GITHUB_ACCESS_TOKEN}" \
                                 -X POST \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${env.GIT_COMMIT} \
                                 -d '{
                                   "state": "pending",
                                   "context": "Jenkins Pipeline",
                                   "description": "Build is running",
                                   "target_url": "${env.BUILD_URL}"
                                 }'
                            """
                        }
                    } else {
                        // Branch thông thường
                        env.CURRENT_BRANCH = env.BRANCH_NAME
                        echo "Processing branch: ${env.CURRENT_BRANCH}"
                        checkout scm
                        env.GIT_COMMIT = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    }
                }
                
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
                // Giữ nguyên các bước triển khai Dev
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
                // Giữ nguyên các bước triển khai Prod
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
                    if (env.IS_PR == 'true' && env.GIT_COMMIT) {
                        // Cập nhật trạng thái thành công trực tiếp qua GitHub Status API
                        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_ACCESS_TOKEN')]) {
                            sh """
                            curl -H "Authorization: token ${GITHUB_ACCESS_TOKEN}" \
                                 -X POST \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${env.GIT_COMMIT} \
                                 -d '{
                                   "state": "success",
                                   "context": "Jenkins Pipeline",
                                   "description": "Build succeeded",
                                   "target_url": "${env.BUILD_URL}"
                                 }'
                            """
                        }
                    }
                }
            }
        }
        failure {
            node('agent-builder') {
                script {
                    if (env.IS_PR == 'true' && env.GIT_COMMIT) {
                        // Cập nhật trạng thái thất bại trực tiếp qua GitHub Status API
                        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_ACCESS_TOKEN')]) {
                            sh """
                            curl -H "Authorization: token ${GITHUB_ACCESS_TOKEN}" \
                                 -X POST \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${env.GIT_COMMIT} \
                                 -d '{
                                   "state": "failure",
                                   "context": "Jenkins Pipeline",
                                   "description": "Build failed",
                                   "target_url": "${env.BUILD_URL}"
                                 }'
                            """
                        }
                    }
                }
            }
        }
    }
}