pipeline {
    agent none

    environment {
        // Thông tin chung
        SSH_USER = 'TaiKhau'
        DEPLOY_DIR = "/home/TaiKhau/app"
        
        // Ports cho deployment
        DEV_PORT = '8081'
        PROD_PORT = '8080'

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

    stages {
        stage('Initialize & Validate') {
            agent { label 'agent-builder' }
            steps {
                script {
                    // Debug: In ra tất cả environment variables
                    echo "=== DEBUG: Environment Variables ==="
                    echo "BRANCH_NAME: ${env.BRANCH_NAME}"
                    echo "CHANGE_ID: ${env.CHANGE_ID}"
                    echo "CHANGE_TARGET: ${env.CHANGE_TARGET}"
                    echo "CHANGE_BRANCH: ${env.CHANGE_BRANCH}"
                    echo "CHANGE_AUTHOR: ${env.CHANGE_AUTHOR}"
                    echo "CHANGE_TITLE: ${env.CHANGE_TITLE}"
                    echo "BUILD_NUMBER: ${env.BUILD_NUMBER}"
                    echo "JOB_NAME: ${env.JOB_NAME}"
                    
                    // Xác định build context
                    def isPullRequest = env.CHANGE_ID != null
                    def isDirectPush = !isPullRequest
                    
                    echo "=== BUILD CONTEXT ANALYSIS ==="
                    echo "Is Pull Request: ${isPullRequest}"
                    echo "Is Direct Push: ${isDirectPush}"
                    
                    if (isPullRequest) {
                        // ========== PULL REQUEST CONTEXT ==========
                        env.SOURCE_BRANCH = env.CHANGE_BRANCH
                        env.TARGET_BRANCH = env.CHANGE_TARGET
                        env.BUILD_TYPE = 'PULL_REQUEST'
                        
                        echo "Pull Request Details:"
                        echo "  PR #${env.CHANGE_ID}"
                        echo "  Source: ${env.SOURCE_BRANCH}"
                        echo "  Target: ${env.TARGET_BRANCH}"
                        echo "  Author: ${env.CHANGE_AUTHOR}"
                        echo "  Title: ${env.CHANGE_TITLE}"
                        
                        // Validate PR target branch
                        if (env.TARGET_BRANCH == 'develop') {
                            env.DEPLOYMENT_ENV = 'development'
                            env.DEPLOY_PORT = env.DEV_PORT
                            env.SHOULD_DEPLOY = 'true'
                            env.VM_IP = env.GCP_VM_DEV
                            env.CONTAINER_NAME = 'aspnetapp-dev'
                            env.AGENT_LABEL = 'agent1'
                        } else if (env.TARGET_BRANCH == 'main') {
                            env.DEPLOYMENT_ENV = 'production'
                            env.DEPLOY_PORT = env.PROD_PORT
                            env.SHOULD_DEPLOY = 'true'
                            env.VM_IP = env.GCP_VM_PROD
                            env.CONTAINER_NAME = 'aspnetapp-prod'
                            env.AGENT_LABEL = 'agent2'
                        } else {
                            error "Pull Requests are only allowed to target 'develop' or 'main' branches. Current target: ${env.TARGET_BRANCH}"
                        }
                        
                    } else {
                        // ========== DIRECT PUSH CONTEXT ==========
                        env.SOURCE_BRANCH = env.BRANCH_NAME
                        env.TARGET_BRANCH = env.BRANCH_NAME
                        
                        echo "Direct Push Details:"
                        echo "  Branch: ${env.BRANCH_NAME}"
                        
                        if (env.BRANCH_NAME == 'develop') {
                            env.BUILD_TYPE = 'DIRECT_PUSH_DEV'
                            env.DEPLOYMENT_ENV = 'development'
                            env.DEPLOY_PORT = env.DEV_PORT
                            env.SHOULD_DEPLOY = 'true'
                            env.VM_IP = env.GCP_VM_DEV
                            env.CONTAINER_NAME = 'aspnetapp-dev'
                            env.AGENT_LABEL = 'agent1'
                        } else if (env.BRANCH_NAME == 'main') {
                            // Chặn direct push vào main
                            error "Direct push to 'main' branch is not allowed. Please create a Pull Request."
                        } else if (env.BRANCH_NAME?.startsWith('feature/') || env.BRANCH_NAME?.startsWith('hotfix/') || env.BRANCH_NAME?.startsWith('bugfix/')) {
                            // Feature branches - Skip pipeline
                            echo "Feature/Hotfix/Bugfix branch detected: ${env.BRANCH_NAME}"
                            echo "Skipping pipeline execution as per configuration."
                            currentBuild.result = 'ABORTED'
                            error "Pipeline skipped for feature branch: ${env.BRANCH_NAME}"
                        } else {
                            // Other branches - không deploy
                            env.BUILD_TYPE = 'BRANCH_BUILD'
                            env.DEPLOYMENT_ENV = 'none'
                            env.SHOULD_DEPLOY = 'false'
                            echo "Branch '${env.BRANCH_NAME}' will be built but not deployed."
                        }
                    }
                    
                    // Tạo image tag
                    if (env.BUILD_TYPE == 'PULL_REQUEST') {
                        env.IMAGE_TAG = "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}"
                    } else {
                        env.IMAGE_TAG = "${env.BRANCH_NAME.replaceAll('/', '-')}-${env.BUILD_NUMBER}"
                    }
                    
                    // Summary
                    echo "=== PIPELINE CONFIGURATION ==="
                    echo "Build Type: ${env.BUILD_TYPE}"
                    echo "Source Branch: ${env.SOURCE_BRANCH}"
                    echo "Target Branch: ${env.TARGET_BRANCH}"
                    echo "Deployment Environment: ${env.DEPLOYMENT_ENV}"
                    echo "Should Deploy: ${env.SHOULD_DEPLOY}"
                    echo "Deploy Port: ${env.DEPLOY_PORT ?: 'N/A'}"
                    echo "VM IP: ${env.VM_IP ?: 'N/A'}"
                    echo "Container Name: ${env.CONTAINER_NAME ?: 'N/A'}"
                    echo "Image Tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Checkout') {
            agent { label 'agent-builder' }
            steps {
                script {
                    echo "Checking out source code..."
                    
                    if (env.BUILD_TYPE == 'PULL_REQUEST') {
                        echo "Checking out PR #${env.CHANGE_ID} merge result"
                    } else {
                        echo "Checking out branch: ${env.BRANCH_NAME}"
                    }
                    
                    // Jenkins multibranch tự động checkout đúng code
                    checkout scm
                }

                sh 'ls -la'
                sh 'pwd'
                sh 'git log --oneline -3'
                sh 'git status'
            }
        }

        stage('Build Docker Image') {
            agent { label 'agent-builder' }
            steps {
                script {
                    echo "Building Docker image: ${IMAGE_NAME}:${env.IMAGE_TAG}"
                    
                    sh """
                    docker build --pull -t ${IMAGE_NAME}:${env.IMAGE_TAG} .
                    """
                    
                    // Tag environment-specific latest images
                    if (env.DEPLOYMENT_ENV == 'development') {
                        sh "docker tag ${IMAGE_NAME}:${env.IMAGE_TAG} ${IMAGE_NAME}:dev-latest"
                        echo "Tagged as dev-latest"
                    } else if (env.DEPLOYMENT_ENV == 'production') {
                        sh "docker tag ${IMAGE_NAME}:${env.IMAGE_TAG} ${IMAGE_NAME}:prod-latest"
                        echo "Tagged as prod-latest"
                    }
                    
                    // List built images
                    sh "docker images | grep ${IMAGE_NAME}"
                }
            }
        }

        stage('Run Tests') {
            agent { label 'agent-builder' }
            steps {
                script {
                    echo "Running application tests..."
                    
                    sh """
                    # Kiểm tra có Dockerfile.test không
                    if [ -f "Dockerfile.test" ]; then
                        echo "Found Dockerfile.test - building test image..."
                        docker build -t aspnetapp-test -f Dockerfile.test .
                        
                        echo "Running tests in container..."
                        docker run --rm aspnetapp-test
                        
                        echo "Tests completed successfully"
                    else
                        echo "No Dockerfile.test found - running smoke test..."
                        
                        # Smoke test: khởi động container tạm thời
                        echo "Starting container for smoke test..."
                        docker run -d --name smoke-test-${env.BUILD_NUMBER} -p 18080:8080 ${IMAGE_NAME}:${env.IMAGE_TAG}
                        
                        # Đợi container khởi động
                        sleep 15
                        
                        # Kiểm tra container có chạy không
                        if docker ps | grep smoke-test-${env.BUILD_NUMBER}; then
                            echo "✅ Smoke test passed - Container started successfully"
                            
                            # Có thể thêm health check ở đây
                            # docker exec smoke-test-${env.BUILD_NUMBER} curl -f http://localhost:8080/health || exit 1
                        else
                            echo "❌ Smoke test failed - Container did not start properly"
                            docker logs smoke-test-${env.BUILD_NUMBER}
                            exit 1
                        fi
                        
                        # Cleanup
                        docker stop smoke-test-${env.BUILD_NUMBER} || true
                        docker rm smoke-test-${env.BUILD_NUMBER} || true
                    fi
                    """
                }
            }
        }

        stage('Push to Docker Registry') {
            agent { label 'agent-builder' }
            when {
                // Chỉ push khi cần deploy hoặc là main branches
                anyOf {
                    expression { env.SHOULD_DEPLOY == 'true' }
                    expression { env.BRANCH_NAME in ['develop', 'main'] }
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    echo "Logging in to Docker Hub..."
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    
                    echo "Pushing image: ${IMAGE_NAME}:${env.IMAGE_TAG}"
                    docker push ${IMAGE_NAME}:${env.IMAGE_TAG}
                    
                    # Push environment-specific latest tags
                    if [ "${env.DEPLOYMENT_ENV}" = "development" ]; then
                        echo "Pushing dev-latest tag..."
                        docker push ${IMAGE_NAME}:dev-latest
                    elif [ "${env.DEPLOYMENT_ENV}" = "production" ]; then
                        echo "Pushing prod-latest tag..."
                        docker push ${IMAGE_NAME}:prod-latest
                    fi
                    
                    echo "✅ Docker images pushed successfully"
                    """
                }
            }
        }

        stage('Deploy to Development') {
            agent { 
                label "${env.AGENT_LABEL ?: 'agent1'}" 
            }
            when {
                allOf {
                    expression { env.SHOULD_DEPLOY == 'true' }
                    expression { env.DEPLOYMENT_ENV == 'development' }
                }
            }
            steps {
                withCredentials([file(credentialsId: 'ssh-private-key-file', variable: 'SSH_KEY')]) {
                    script {
                        echo "🚀 Deploying to Development Environment"
                        echo "Image: ${IMAGE_NAME}:${env.IMAGE_TAG}"
                        echo "Target VM: ${env.VM_IP}"
                        echo "Port: ${env.DEPLOY_PORT}"
                        
                        sh """
                        ssh -i \$SSH_KEY -o StrictHostKeyChecking=no -o ConnectTimeout=30 ${SSH_USER}@${env.VM_IP} << 'EOF'
set -e
trap 'echo "[ERROR] Dev deployment failed on \$HOSTNAME at \$(date)!" >&2; exit 1' ERR

echo "=================================================="
echo "🚀 DEVELOPMENT DEPLOYMENT STARTED"
echo "Time: \$(date)"
echo "Image: ${IMAGE_NAME}:${env.IMAGE_TAG}"
echo "Container: ${env.CONTAINER_NAME}"
echo "Port: ${env.DEPLOY_PORT}"
echo "=================================================="

# Tạo deployment directory
echo "[INFO] Preparing deployment directory..."
mkdir -p ${DEPLOY_DIR} && cd ${DEPLOY_DIR}

# Pull image
echo "[INFO] Pulling Docker image..."
docker pull ${IMAGE_NAME}:${env.IMAGE_TAG}

# Stop và remove container cũ
echo "[INFO] Stopping existing container..."
if docker ps -q -f name=${env.CONTAINER_NAME}; then
    docker stop ${env.CONTAINER_NAME}
    echo "[INFO] Stopped existing container: ${env.CONTAINER_NAME}"
fi

echo "[INFO] Removing old container..."
if docker ps -aq -f name=${env.CONTAINER_NAME}; then
    docker rm ${env.CONTAINER_NAME}
    echo "[INFO] Removed old container: ${env.CONTAINER_NAME}"
fi

# Start container mới
echo "[INFO] Starting new development container..."
docker run -d \\
    -p ${env.DEPLOY_PORT}:8080 \\
    --name ${env.CONTAINER_NAME} \\
    --restart unless-stopped \\
    -e ASPNETCORE_ENVIRONMENT=Development \\
    -e ASPNETCORE_URLS=http://+:8080 \\
    ${IMAGE_NAME}:${env.IMAGE_TAG}

# Health check
echo "[INFO] Performing health check..."
sleep 10

# Kiểm tra container status
if docker ps -f name=${env.CONTAINER_NAME} --format "table {{.Names}}\\t{{.Status}}" | grep -q "Up"; then
    CONTAINER_IP=\$(hostname -I | awk '{print \$1}')
    echo ""
    echo "=================================================="
    echo "✅ DEVELOPMENT DEPLOYMENT SUCCESSFUL!"
    echo "Time: \$(date)"
    echo "Application URL: http://\$CONTAINER_IP:${env.DEPLOY_PORT}"
    echo "Container Status: \$(docker ps -f name=${env.CONTAINER_NAME} --format '{{.Status}}')"
    echo "=================================================="
else
    echo ""
    echo "=================================================="
    echo "❌ DEPLOYMENT FAILED - Container not running"
    echo "Container logs:"
    docker logs ${env.CONTAINER_NAME}
    echo "=================================================="
    exit 1
fi
EOF
                        """
                    }
                }
            }
        }

        stage('Deploy to Production') {
            agent { 
                label "${env.AGENT_LABEL ?: 'agent2'}" 
            }
            when {
                allOf {
                    expression { env.SHOULD_DEPLOY == 'true' }
                    expression { env.DEPLOYMENT_ENV == 'production' }
                }
            }
            steps {
                script {
                    // Production deployment approval
                    echo "⚠️ Production deployment requested"
                    echo "PR: #${env.CHANGE_ID} - ${env.CHANGE_TITLE}"
                    echo "Author: ${env.CHANGE_AUTHOR}"
                    echo "Image: ${IMAGE_NAME}:${env.IMAGE_TAG}"
                    
                    timeout(time: 15, unit: 'MINUTES') {
                        def approval = input(
                            message: "🚀 Deploy to Production?",
                            ok: 'Deploy to Production',
                            parameters: [
                                string(
                                    name: 'APPROVER_NAME', 
                                    defaultValue: '', 
                                    description: 'Your full name for audit trail'
                                ),
                                choice(
                                    name: 'APPROVAL_ACTION', 
                                    choices: ['DEPLOY', 'CANCEL'], 
                                    description: 'Deployment decision'
                                ),
                                text(
                                    name: 'DEPLOYMENT_NOTES',
                                    defaultValue: '',
                                    description: 'Optional deployment notes'
                                )
                            ]
                        )
                        
                        if (approval.APPROVAL_ACTION != 'DEPLOY') {
                            error("❌ Production deployment cancelled by ${approval.APPROVER_NAME}")
                        }
                        
                        echo "✅ Production deployment approved by: ${approval.APPROVER_NAME}"
                        if (approval.DEPLOYMENT_NOTES) {
                            echo "📝 Deployment notes: ${approval.DEPLOYMENT_NOTES}"
                        }
                        
                        env.APPROVER_NAME = approval.APPROVER_NAME
                    }
                }
                
                withCredentials([file(credentialsId: 'ssh-private-key-file', variable: 'SSH_KEY')]) {
                    script {
                        echo "🚀 Deploying to Production Environment"
                        echo "Approved by: ${env.APPROVER_NAME}"
                        
                        sh """
                        ssh -i \$SSH_KEY -o StrictHostKeyChecking=no -o ConnectTimeout=30 ${SSH_USER}@${env.VM_IP} << 'EOF'
set -e
trap 'echo "[ERROR] Production deployment failed on \$HOSTNAME at \$(date)!" >&2; exit 1' ERR

echo "=================================================="
echo "🚀 PRODUCTION DEPLOYMENT STARTED"
echo "Time: \$(date)"
echo "Image: ${IMAGE_NAME}:${env.IMAGE_TAG}"
echo "Container: ${env.CONTAINER_NAME}"
echo "Port: ${env.DEPLOY_PORT}"
echo "Approved by: ${env.APPROVER_NAME}"
echo "=================================================="

# Tạo deployment directory
echo "[INFO] Preparing deployment directory..."
mkdir -p ${DEPLOY_DIR} && cd ${DEPLOY_DIR}

# Backup current container
echo "[INFO] Creating backup of current production container..."
if docker ps -q -f name=${env.CONTAINER_NAME}; then
    BACKUP_TAG="backup-\$(date +%Y%m%d-%H%M%S)"
    docker commit ${env.CONTAINER_NAME} ${IMAGE_NAME}:\$BACKUP_TAG || echo "[WARN] Could not create backup"
    echo "[INFO] Backup created: ${IMAGE_NAME}:\$BACKUP_TAG"
fi

# Pull new image
echo "[INFO] Pulling new Docker image..."
docker pull ${IMAGE_NAME}:${env.IMAGE_TAG}

# Stop current container
echo "[INFO] Stopping current production container..."
if docker ps -q -f name=${env.CONTAINER_NAME}; then
    docker stop ${env.CONTAINER_NAME}
    echo "[INFO] Stopped production container: ${env.CONTAINER_NAME}"
fi

echo "[INFO] Removing old container..."
if docker ps -aq -f name=${env.CONTAINER_NAME}; then
    docker rm ${env.CONTAINER_NAME}
    echo "[INFO] Removed old container: ${env.CONTAINER_NAME}"
fi

# Start new production container  
echo "[INFO] Starting new production container..."
docker run -d \\
    -p ${env.DEPLOY_PORT}:8080 \\
    --name ${env.CONTAINER_NAME} \\
    --restart unless-stopped \\
    -e ASPNETCORE_ENVIRONMENT=Production \\
    -e ASPNETCORE_URLS=http://+:8080 \\
    ${IMAGE_NAME}:${env.IMAGE_TAG}

# Extended health check for production
echo "[INFO] Performing production health check..."
sleep 20

# Kiểm tra container status
if docker ps -f name=${env.CONTAINER_NAME} --format "table {{.Names}}\\t{{.Status}}" | grep -q "Up"; then
    CONTAINER_IP=\$(hostname -I | awk '{print \$1}')
    echo ""
    echo "=================================================="
    echo "✅ PRODUCTION DEPLOYMENT SUCCESSFUL!"
    echo "Time: \$(date)"
    echo "Application URL: http://\$CONTAINER_IP:${env.DEPLOY_PORT}"
    echo "Container Status: \$(docker ps -f name=${env.CONTAINER_NAME} --format '{{.Status}}')"
    echo "Approved by: ${env.APPROVER_NAME}"
    echo "=================================================="
    
    # Log deployment event
    echo "\$(date): Production deployment completed - Image: ${IMAGE_NAME}:${env.IMAGE_TAG} - Approved by: ${env.APPROVER_NAME}" >> ${DEPLOY_DIR}/deployment.log
else
    echo ""
    echo "=================================================="
    echo "❌ PRODUCTION DEPLOYMENT FAILED"
    echo "Container logs:"
    docker logs ${env.CONTAINER_NAME}
    echo "=================================================="
    exit 1
fi
EOF
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                echo "=================================================="
                echo "📋 PIPELINE EXECUTION SUMMARY"
                echo "=================================================="
                echo "Build Type: ${env.BUILD_TYPE}"
                echo "Source Branch: ${env.SOURCE_BRANCH}"
                echo "Target Branch: ${env.TARGET_BRANCH}"
                echo "Image Tag: ${env.IMAGE_TAG}"
                echo "Deployment Environment: ${env.DEPLOYMENT_ENV}"
                echo "Build Result: ${currentBuild.result ?: 'SUCCESS'}"
                echo "Build Duration: ${currentBuild.durationString}"
                echo "Build URL: ${env.BUILD_URL}"
                
                if (env.BUILD_TYPE == 'PULL_REQUEST') {
                    echo "Pull Request: #${env.CHANGE_ID} by ${env.CHANGE_AUTHOR}"
                    echo "PR Title: ${env.CHANGE_TITLE}"
                }
                
                if (env.APPROVER_NAME) {
                    echo "Production Approved by: ${env.APPROVER_NAME}"
                }
                echo "=================================================="
            }
        }
        
        success {
            script {
                def message = "✅ Pipeline completed successfully!"
                
                if (env.SHOULD_DEPLOY == 'true') {
                    if (env.DEPLOYMENT_ENV == 'development') {
                        message = "✅ Development deployment completed successfully!"
                    } else if (env.DEPLOYMENT_ENV == 'production') {
                        message = "✅ Production deployment completed successfully!"
                    }
                }
                
                echo message
                
                // Có thể thêm Slack/Teams notification ở đây
                // slackSend(message: message)
            }
        }
        
        failure {
            script {
                def message = "❌ Pipeline failed"
                
                if (env.BUILD_TYPE == 'PULL_REQUEST') {
                    message += " for PR #${env.CHANGE_ID}"
                } else {
                    message += " for branch ${env.BRANCH_NAME}"
                }
                
                if (env.DEPLOYMENT_ENV && env.DEPLOYMENT_ENV != 'none') {
                    message += " during ${env.DEPLOYMENT_ENV} deployment"
                }
                
                echo message
                
                // Có thể thêm notification ở đây
            }
        }
        
        aborted {
            script {
                if (currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')) {
                    echo "⏹️ Pipeline was manually aborted by user"
                } else {
                    echo "⏹️ Pipeline was aborted (likely feature branch skip)"
                }
            }
        }
        
        cleanup {
            script {
                // Cleanup trên build nodes
                try {
                    node('agent-builder') {
                        sh """
                        # Remove test containers
                        docker rm -f smoke-test-${env.BUILD_NUMBER} aspnetapp-test 2>/dev/null || true
                        
                        # Remove test images
                        docker rmi aspnetapp-test 2>/dev/null || true
                        
                        # Clean up old images (giữ lại 10 images gần nhất)
                        docker images ${IMAGE_NAME} --format "table {{.Repository}}:{{.Tag}}\\t{{.CreatedAt}}" | \\
                        tail -n +11 | awk '{print \$1}' | xargs -r docker rmi 2>/dev/null || true
                        
                        # Prune unused images và containers
                        docker container prune -f || true
                        docker image prune -f || true
                        
                        echo "✅ Cleanup completed"
                        """
                    }
                } catch (Exception e) {
                    echo "⚠️ Cleanup warning: ${e.getMessage()}"
                }
            }
        }
    }
}