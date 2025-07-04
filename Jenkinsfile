pipeline {
    agent none

    environment {
        // Thông tin chung 
        SSH_USER = credentials('ssh-user')
        DEPLOY_DIR = credentials('deploy-dir')
        GCP_VM_DEV = credentials('gcp-vm-dev')
        GCP_VM_PROD = credentials('gcp-vm-prod')
        DOCKER_HUB_CREDS = credentials('dockerhub-credentials')
        DOCKER_HUB_USERNAME = "${DOCKER_HUB_CREDS_USR}"
        IMAGE_NAME = "${DOCKER_HUB_USERNAME}/aspnetapp"
        GITHUB_TOKEN = credentials('github-token')
        
        // Tên repo để gửi trạng thái build
        REPO_OWNER = credentials('repo-owner')
        REPO_NAME = credentials('repo-name')

        // Ngày và giờ build (sử dụng định dạng ISO)
        BUILD_DATE = sh(script: 'date -u "+%Y-%m-%dT%H:%M:%SZ"', returnStdout: true).trim()
    }

    stages {
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

        stage('Checkout') {
            agent { label 'agent-builder' }
            steps {
                // Lấy nhánh hiện tại
                script {
                    // Xác định nếu đây là một Pull Request
                    env.IS_PR = env.CHANGE_ID ? true : false
                    
                    // Tạo biến cho Git version
                    checkout scm
                    env.GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.GIT_BRANCH_NAME = env.IS_PR ? env.CHANGE_BRANCH : env.BRANCH_NAME
                    
                    // Xác định version cho tag
                    // Đọc version từ file version.txt nếu có, hoặc mặc định 1.0.0
                    env.APP_VERSION = "1.0.0"
                    if (fileExists('version.txt')) {
                        env.APP_VERSION = readFile('version.txt').trim()
                    }
                    
                    // Tạo các biến tag cho image
                    env.IMAGE_TAG_LATEST = "latest"
                    env.IMAGE_TAG_VERSION = "${env.APP_VERSION}"
                    env.IMAGE_TAG_COMMIT = "${env.GIT_COMMIT_SHORT}"
                    env.IMAGE_TAG_BRANCH = "${env.GIT_BRANCH_NAME}".replaceAll('/', '-')
                    
                    // Tạo tag đặc biệt cho staging và production
                    if (env.GIT_BRANCH_NAME == 'main') {
                        env.IMAGE_TAG_ENV = "production"
                    } else if (env.GIT_BRANCH_NAME == 'dev') {
                        env.IMAGE_TAG_ENV = "staging"
                    } else {
                        env.IMAGE_TAG_ENV = "dev"
                    }
                    
                    // Tag đầy đủ với version, commit và thời gian build
                    env.IMAGE_TAG_FULL = "${env.APP_VERSION}-${env.GIT_COMMIT_SHORT}-${BUILD_NUMBER}"
                    
                    echo "Các tag được sử dụng cho Docker image:"
                    echo "- ${IMAGE_NAME}:${IMAGE_TAG_LATEST}"
                    echo "- ${IMAGE_NAME}:${IMAGE_TAG_VERSION}"
                    echo "- ${IMAGE_NAME}:${IMAGE_TAG_COMMIT}"
                    echo "- ${IMAGE_NAME}:${IMAGE_TAG_BRANCH}"
                    echo "- ${IMAGE_NAME}:${IMAGE_TAG_ENV}"
                    echo "- ${IMAGE_NAME}:${IMAGE_TAG_FULL}"
                    
                    if (env.IS_PR) {
                        // Pull Request
                        env.CURRENT_BRANCH = env.CHANGE_BRANCH
                        echo "Processing Pull Request #${env.CHANGE_ID} from branch: ${env.CURRENT_BRANCH} to ${env.CHANGE_TARGET}"
                        
                        // Đánh dấu trạng thái bắt đầu build
                        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_ACCESS_TOKEN')]) {
                            sh """
                            curl -H "Authorization: token ${GITHUB_ACCESS_TOKEN}" \
                                 -X POST \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${env.GIT_COMMIT} \
                                 -d '{
                                   "state": "pending",
                                   "context": "continuous-integration/jenkins/pr-merge",
                                   "description": "Build is running",
                                   "target_url": "${env.BUILD_URL}"
                                 }'
                            """
                            
                            // Update pr-head context
                            sh """
                            curl -H "Authorization: token ${GITHUB_ACCESS_TOKEN}" \
                                 -X POST \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${env.GIT_COMMIT} \
                                 -d '{
                                   "state": "pending",
                                   "context": "continuous-integration/jenkins/pr-head",
                                   "description": "Build is running",
                                   "target_url": "${env.BUILD_URL}"
                                 }'
                            """
                        }
                    } else {
                        // Branch thông thường
                        env.CURRENT_BRANCH = env.BRANCH_NAME
                        echo "Processing branch: ${env.CURRENT_BRANCH}"
                        
                        // Update branch context cho non-PR builds
                        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_ACCESS_TOKEN')]) {
                            sh """
                            curl -H "Authorization: token ${GITHUB_ACCESS_TOKEN}" \
                                 -X POST \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${env.GIT_COMMIT} \
                                 -d '{
                                   "state": "pending",
                                   "context": "continuous-integration/jenkins/branch",
                                   "description": "Build is running",
                                   "target_url": "${env.BUILD_URL}"
                                 }'
                            """
                        }
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
                    // Build Docker image với multiple tags
                    sh """
                    docker build --pull -t ${IMAGE_NAME}:${IMAGE_TAG_LATEST} \\
                                      -t ${IMAGE_NAME}:${IMAGE_TAG_VERSION} \\
                                      -t ${IMAGE_NAME}:${IMAGE_TAG_COMMIT} \\
                                      -t ${IMAGE_NAME}:${IMAGE_TAG_BRANCH} \\
                                      -t ${IMAGE_NAME}:${IMAGE_TAG_ENV} \\
                                      -t ${IMAGE_NAME}:${IMAGE_TAG_FULL} \\
                                      --build-arg VERSION=${APP_VERSION} \\
                                      --build-arg BUILD_DATE=${BUILD_DATE} \\
                                      --build-arg VCS_REF=${GIT_COMMIT_SHORT} \\
                                      --label org.opencontainers.image.created=${BUILD_DATE} \\
                                      --label org.opencontainers.image.version=${APP_VERSION} \\
                                      --label org.opencontainers.image.revision=${GIT_COMMIT_SHORT} \\
                                      --label org.label-schema.build-date=${BUILD_DATE} \\
                                      --label org.label-schema.vcs-ref=${GIT_COMMIT_SHORT} \\
                                      --label org.label-schema.version=${APP_VERSION} \\
                                      .
                    """
                }
            }
        }

        stage('Test') {
            agent { label 'agent-builder' }
            steps {
                sh """
                docker build -t ${IMAGE_NAME}:test-${GIT_COMMIT_SHORT} -f Dockerfile.test .
                docker images | grep ${IMAGE_NAME}
                """
            }
        }

        stage('Push to Docker Hub') {
            agent { label 'agent-builder' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    
                    # Push tất cả các tag của image
                    docker push ${IMAGE_NAME}:${IMAGE_TAG_LATEST}
                    docker push ${IMAGE_NAME}:${IMAGE_TAG_VERSION}
                    docker push ${IMAGE_NAME}:${IMAGE_TAG_COMMIT}
                    docker push ${IMAGE_NAME}:${IMAGE_TAG_BRANCH}
                    docker push ${IMAGE_NAME}:${IMAGE_TAG_ENV}
                    docker push ${IMAGE_NAME}:${IMAGE_TAG_FULL}
                    
                    # Lưu thông tin tag cho các bước triển khai tiếp theo
                    echo "${IMAGE_TAG_FULL}" > deploy_image_tag.txt
                    """
                    
                    // Lưu tag để sử dụng cho các stage tiếp theo
                    script {
                        env.DEPLOY_IMAGE_TAG = env.IMAGE_TAG_FULL
                    }
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
                // Thêm bước xác nhận trước khi deploy
                // input message: "Bạn có muốn triển khai phiên bản ${DEPLOY_IMAGE_TAG} lên môi trường Dev không?", 
                //       ok: "Tiếp tục triển khai",
                //       submitter: "admin,TaiKhau"
                      
                echo "Triển khai phiên bản ${DEPLOY_IMAGE_TAG} lên Dev được xác nhận, tiếp tục..."
                
                withCredentials([file(credentialsId: 'ssh-private-key-file', variable: 'SSH_KEY')]) {
                    script {
                        sh """
                        ssh -i $SSH_KEY -o StrictHostKeyChecking=no $SSH_USER@${GCP_VM_DEV} << 'EOF'
set -e
trap 'echo "[ERROR] Deployment failed on \$(hostname)!" >&2; exit 1' ERR

echo "[INFO] Switching to deployment directory..."
mkdir -p $DEPLOY_DIR && cd $DEPLOY_DIR

echo "[INFO] Pulling Docker image ${IMAGE_NAME}:${DEPLOY_IMAGE_TAG}..."
docker pull ${IMAGE_NAME}:${DEPLOY_IMAGE_TAG}

echo "[INFO] Creating or updating environment variables file..."
cat > .env << 'ENVFILE'
APP_VERSION=${APP_VERSION}
BUILD_NUMBER=${BUILD_NUMBER}
GIT_COMMIT='\${GIT_COMMIT_SHORT}'
DEPLOY_DATE=$(date -u "+%Y-%m-%dT%H:%M:%SZ")
ENVFILE

echo "[INFO] Restarting container..."
docker stop aspnetapp || true
docker rm aspnetapp || true
docker run -d --env-file .env -p ${CONTAINER_PORT}:8080 --name aspnetapp ${IMAGE_NAME}:${DEPLOY_IMAGE_TAG}

echo "[INFO] Tagging current deployment..."
echo "${DEPLOY_IMAGE_TAG}" > current_deployment.txt

echo "[SUCCESS] Dev Deployment complete on \$(hostname)"
EOF
                        """
                    }
                }
            }
            post {
                success {
                    echo "Đã triển khai thành công lên môi trường Dev"
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
                // input message: "Bạn có chắc muốn deploy phiên bản ${DEPLOY_IMAGE_TAG} lên môi trường Production?",
                //       ok: "Xác nhận triển khai",
                //       submitter: "admin,TaiKhau"

                echo "Triển khai phiên bản ${DEPLOY_IMAGE_TAG} lên Prod được xác nhận, tiếp tục..."
                
                withCredentials([file(credentialsId: 'ssh-private-key-file', variable: 'SSH_KEY')]) {
                    script {
                        sh """
                        ssh -i $SSH_KEY -o StrictHostKeyChecking=no $SSH_USER@${GCP_VM_PROD} << 'EOF'
set -e
trap 'echo "[ERROR] Deployment failed on \$(hostname)!" >&2; exit 1' ERR

echo "[INFO] Switching to deployment directory..."
mkdir -p $DEPLOY_DIR && cd $DEPLOY_DIR

echo "[INFO] Pulling Docker image ${IMAGE_NAME}:${DEPLOY_IMAGE_TAG}..."
docker pull ${IMAGE_NAME}:${DEPLOY_IMAGE_TAG}

echo "[INFO] Creating or updating environment variables file..."
cat > .env << 'ENVFILE'
APP_VERSION=${APP_VERSION}
BUILD_NUMBER=${BUILD_NUMBER}
GIT_COMMIT='\${GIT_COMMIT_SHORT}'
ENVIRONMENT=production
DEPLOY_DATE=$(date -u "+%Y-%m-%dT%H:%M:%SZ")
ENVFILE

echo "[INFO] Restarting container..."
docker stop aspnetapp || true
docker rm aspnetapp || true
docker run -d --env-file .env -p ${CONTAINER_PORT}:8080 --name aspnetapp ${IMAGE_NAME}:${DEPLOY_IMAGE_TAG}

echo "[INFO] Tagging current deployment..."
echo "${DEPLOY_IMAGE_TAG}" > current_deployment.txt

echo "[SUCCESS] Prod Deployment complete on \$(hostname)"
EOF
                        """
                    }
                }
            }
            post {
                success {
                    echo "Đã triển khai thành công lên môi trường Production"
                }
            }
        }
    }
    
    post {
        success {
            node('agent-builder') {
                script {
                    if (env.IS_PR == 'true' && env.GIT_COMMIT) {
                        // Cập nhật tất cả các context với trạng thái success
                        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_ACCESS_TOKEN')]) {
                            // Update pr-merge context
                            sh """
                            curl -H "Authorization: token ${GITHUB_ACCESS_TOKEN}" \
                                 -X POST \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${env.GIT_COMMIT} \
                                 -d '{
                                   "state": "success",
                                   "context": "continuous-integration/jenkins/pr-merge",
                                   "description": "Build succeeded",
                                   "target_url": "${env.BUILD_URL}"
                                 }'
                            """
                            
                            // Update pr-head context
                            sh """
                            curl -H "Authorization: token ${GITHUB_ACCESS_TOKEN}" \
                                 -X POST \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${env.GIT_COMMIT} \
                                 -d '{
                                   "state": "success",
                                   "context": "continuous-integration/jenkins/pr-head",
                                   "description": "Build succeeded",
                                   "target_url": "${env.BUILD_URL}"
                                 }'
                            """
                        }
                    } else if (env.GIT_COMMIT) {
                        // Update branch context cho non-PR builds
                        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_ACCESS_TOKEN')]) {
                            sh """
                            curl -H "Authorization: token ${GITHUB_ACCESS_TOKEN}" \
                                 -X POST \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${env.GIT_COMMIT} \
                                 -d '{
                                   "state": "success",
                                   "context": "continuous-integration/jenkins/branch",
                                   "description": "Build succeeded",
                                   "target_url": "${env.BUILD_URL}"
                                 }'
                            """
                        }
                    }
                    
                    // Clean up old images
                    sh '''
                    docker system prune -af --volumes
                    '''
                }
            }
        }
        failure {
            node('agent-builder') {
                script {
                    if (env.IS_PR == 'true' && env.GIT_COMMIT) {
                        // Cập nhật tất cả các context với trạng thái failure
                        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_ACCESS_TOKEN')]) {
                            // Update pr-merge context
                            sh """
                            curl -H "Authorization: token ${GITHUB_ACCESS_TOKEN}" \
                                 -X POST \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${env.GIT_COMMIT} \
                                 -d '{
                                   "state": "failure",
                                   "context": "continuous-integration/jenkins/pr-merge",
                                   "description": "Build failed",
                                   "target_url": "${env.BUILD_URL}"
                                 }'
                            """
                            
                            // Update pr-head context
                            sh """
                            curl -H "Authorization: token ${GITHUB_ACCESS_TOKEN}" \
                                 -X POST \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${env.GIT_COMMIT} \
                                 -d '{
                                   "state": "failure",
                                   "context": "continuous-integration/jenkins/pr-head",
                                   "description": "Build failed",
                                   "target_url": "${env.BUILD_URL}"
                                 }'
                            """
                        }
                    } else if (env.GIT_COMMIT) {
                        // Update branch context cho non-PR builds
                        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_ACCESS_TOKEN')]) {
                            sh """
                            curl -H "Authorization: token ${GITHUB_ACCESS_TOKEN}" \
                                 -X POST \
                                 -H "Accept: application/vnd.github.v3+json" \
                                 https://api.github.com/repos/${REPO_OWNER}/${REPO_NAME}/statuses/${env.GIT_COMMIT} \
                                 -d '{
                                   "state": "failure",
                                   "context": "continuous-integration/jenkins/branch",
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