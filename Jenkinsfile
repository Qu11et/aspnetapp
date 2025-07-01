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
        stage('Initialize') {
            agent { label 'agent-builder' }
            steps {
                script {
                    // Debug: In ra tất cả environment variables liên quan đến PR
                    echo "=== DEBUG: Environment Variables ==="
                    echo "BRANCH_NAME: ${env.BRANCH_NAME}"
                    echo "CHANGE_ID: ${env.CHANGE_ID}"
                    echo "CHANGE_TARGET: ${env.CHANGE_TARGET}"
                    echo "CHANGE_BRANCH: ${env.CHANGE_BRANCH}"
                    echo "CHANGE_AUTHOR: ${env.CHANGE_AUTHOR}"
                    echo "CHANGE_TITLE: ${env.CHANGE_TITLE}"
                    echo "BUILD_NUMBER: ${env.BUILD_NUMBER}"
                    echo "JOB_NAME: ${env.JOB_NAME}"
                    
                    // Với "Only branches that are also filed as PRs", tất cả builds đều là PR context
                    def isPullRequest = env.CHANGE_ID != null
                    
                    echo "=== BUILD CONTEXT ==="
                    echo "Is Pull Request: ${isPullRequest}"
                    
                    if (!isPullRequest) {
                        error "This pipeline is configured to run only on Pull Requests. Current build is not a PR context."
                    }
                    
                    // Xác định target branch và deployment environment  
                    env.TARGET_BRANCH = env.CHANGE_TARGET
                    env.SOURCE_BRANCH = env.CHANGE_BRANCH
                    env.BUILD_TYPE = 'PR'
                    
                    echo "Processing Pull Request #${env.CHANGE_ID}"
                    echo "Source: ${env.SOURCE_BRANCH} → Target: ${env.TARGET_BRANCH}"
                    
                    // Validate target branch
                    if (!(env.TARGET_BRANCH in ['develop', 'main'])) {
                        error "Pull requests are only allowed to target 'develop' or 'main' branches. Current target: ${env.TARGET_BRANCH}"
                    }
                    
                    // Set deployment parameters
                    if (env.TARGET_BRANCH == 'develop') {
                        env.DEPLOY_PORT = env.DEV_PORT
                        env.DEPLOYMENT_ENV = 'development'
                        env.SHOULD_DEPLOY = 'true'
                    } else if (env.TARGET_BRANCH == 'main') {
                        env.DEPLOY_PORT = env.PROD_PORT
                        env.DEPLOYMENT_ENV = 'production'
                        env.SHOULD_DEPLOY = 'true'
                    } else {
                        env.SHOULD_DEPLOY = 'false'
                        env.DEPLOYMENT_ENV = 'none'
                    }
                    
                    echo "=== DEPLOYMENT CONFIG ==="
                    echo "Build Type: ${env.BUILD_TYPE}"
                    echo "Target Branch: ${env.TARGET_BRANCH}"
                    echo "Deployment Environment: ${env.DEPLOYMENT_ENV}"
                    echo "Should Deploy: ${env.SHOULD_DEPLOY}"
                    echo "Deploy Port: ${env.DEPLOY_PORT ?: 'N/A'}"
                }
            }
        }

        stage('Checkout') {
            agent { label 'agent-builder' }
            steps {
                script {
                    echo "Checking out source code..."
                    
                    if (env.BUILD_TYPE == 'PR') {
                        // Checkout PR merge commit
                        echo "Checking out PR #${env.CHANGE_ID} merge commit"
                        checkout scm
                    } else {
                        // Checkout branch thông thường
                        echo "Checking out branch: ${env.BRANCH_NAME}"
                        checkout scm
                    }
                }

                sh 'ls -la'
                sh 'pwd'
                sh 'git log --oneline -3'
                sh 'git branch -a'
            }
        }

        stage('Build') {
            agent { label 'agent-builder' }
            steps {
                script {
                    // Tạo unique tag cho build
                    if (env.BUILD_TYPE == 'PR') {
                        env.IMAGE_TAG = "pr-${env.CHANGE_ID}-${env.BUILD_NUMBER}"
                    } else {
                        env.IMAGE_TAG = "${env.BRANCH_NAME.replaceAll('/', '-')}-${env.BUILD_NUMBER}"
                    }
                    
                    echo "Building Docker image: ${IMAGE_NAME}:${env.IMAGE_TAG}"
                    
                    sh """
                    docker build --pull -t ${IMAGE_NAME}:${env.IMAGE_TAG} .
                    """
                    
                    // Tag thêm cho environment
                    if (env.TARGET_BRANCH == 'develop') {
                        sh "docker tag ${IMAGE_NAME}:${env.IMAGE_TAG} ${IMAGE_NAME}:dev-latest"
                    } else if (env.TARGET_BRANCH == 'main') {
                        sh "docker tag ${IMAGE_NAME}:${env.IMAGE_TAG} ${IMAGE_NAME}:prod-latest"
                    }
                }
            }
        }

        stage('Test') {
            agent { label 'agent-builder' }
            steps {
                script {
                    echo "Running tests..."
                    sh """
                    # Kiểm tra Dockerfile.test có tồn tại không
                    if [ -f "Dockerfile.test" ]; then
                        echo "Building test image..."
                        docker build -t aspnetapp-test -f Dockerfile.test .
                        
                        echo "Running tests..."
                        docker run --rm aspnetapp-test
                    else
                        echo "No Dockerfile.test found, skipping test container"
                        echo "Running basic smoke test on main image..."
                        
                        # Chạy container tạm thời để test
                        docker run -d --name temp-test -p 18080:8080 ${IMAGE_NAME}:${env.IMAGE_TAG}
                        sleep 10
                        
                        # Kiểm tra container có chạy không
                        if docker ps | grep temp-test; then
                            echo "Container started successfully"
                            # Có thể thêm health check ở đây
                            # curl -f http://localhost:18080/health || exit 1
                        else
                            echo "Container failed to start"
                            docker logs temp-test
                            exit 1
                        fi
                        
                        # Cleanup
                        docker stop temp-test || true
                        docker rm temp-test || true
                    fi
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            agent { label 'agent-builder' }
            when {
                // Chỉ push khi build thành công và là PR hoặc main branches
                anyOf {
                    expression { env.BUILD_TYPE == 'PR' }
                    expression { env.BUILD_TYPE == 'DIRECT_PUSH' && env.TARGET_BRANCH in ['develop', 'main'] }
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    
                    # Push specific tag
                    docker push ${IMAGE_NAME}:${env.IMAGE_TAG}
                    
                    # Push environment-specific latest tag if applicable
                    if [ "${env.TARGET_BRANCH}" = "develop" ]; then
                        docker push ${IMAGE_NAME}:dev-latest
                        echo "Pushed dev-latest tag"
                    elif [ "${env.TARGET_BRANCH}" = "main" ]; then
                        docker push ${IMAGE_NAME}:prod-latest
                        echo "Pushed prod-latest tag"
                    fi
                    
                    echo "Successfully pushed images to Docker Hub"
                    """
                }
            }
        }

        stage('Deploy to Dev') {
            agent { label 'agent1' }
            when {
                allOf {
                    expression { env.SHOULD_DEPLOY == 'true' }
                    expression { env.TARGET_BRANCH == 'develop' }
                }
            }
            steps {
                withCredentials([file(credentialsId: 'ssh-private-key-file', variable: 'SSH_KEY')]) {
                    script {
                        echo "Deploying to Development environment..."
                        echo "Image: ${IMAGE_NAME}:${env.IMAGE_TAG}"
                        
                        sh """
                        ssh -i \$SSH_KEY -o StrictHostKeyChecking=no -o ConnectTimeout=30 \$SSH_USER@${GCP_VM_DEV} << 'EOF'
set -e
trap 'echo "[ERROR] Dev deployment failed on \$HOSTNAME at \$(date)!" >&2; exit 1' ERR

echo "[INFO] [\$(date)] Starting deployment to DEV environment..."
echo "[INFO] Image: ${IMAGE_NAME}:${env.IMAGE_TAG}"

cd ${DEPLOY_DIR} || mkdir -p ${DEPLOY_DIR} && cd ${DEPLOY_DIR}

echo "[INFO] Pulling Docker image..."
docker pull ${IMAGE_NAME}:${env.IMAGE_TAG}

echo "[INFO] Stopping existing container..."
docker stop aspnetapp-dev 2>/dev/null || echo "No existing container to stop"
docker rm aspnetapp-dev 2>/dev/null || echo "No existing container to remove"

echo "[INFO] Starting new container..."
docker run -d \\
    -p ${env.DEPLOY_PORT}:8080 \\
    --name aspnetapp-dev \\
    --restart unless-stopped \\
    -e ASPNETCORE_ENVIRONMENT=Development \\
    ${IMAGE_NAME}:${env.IMAGE_TAG}

echo "[INFO] Waiting for container to be ready..."
sleep 10

if docker ps -f name=aspnetapp-dev --format "table {{.Names}}\\t{{.Status}}" | grep -q "Up"; then
    echo "[SUCCESS] [\$(date)] Dev deployment complete!"
    echo "[INFO] Application running on http://\$(hostname -I | awk '{print \$1}'):${env.DEPLOY_PORT}"
else
    echo "[ERROR] Container failed to start"
    docker logs aspnetapp-dev
    exit 1
fi
EOF
                        """
                    }
                }
            }
        }

        stage('Deploy to Prod') {
            agent { label 'agent2' }
            when {
                allOf {
                    expression { env.SHOULD_DEPLOY == 'true' }
                    expression { env.TARGET_BRANCH == 'main' }
                }
            }
            steps {
                script {
                    // Manual approval cho production
                    timeout(time: 10, unit: 'MINUTES') {
                        def deployApproved = input(
                            message: "Deploy to Production?",
                            ok: 'Deploy',
                            parameters: [
                                string(name: 'APPROVER', defaultValue: '', description: 'Your name for approval audit'),
                                choice(name: 'DEPLOY_ACTION', choices: ['Deploy', 'Abort'], description: 'Deployment action')
                            ]
                        )
                        
                        if (deployApproved.DEPLOY_ACTION != 'Deploy') {
                            error("Production deployment cancelled by ${deployApproved.APPROVER}")
                        }
                        
                        echo "Production deployment approved by: ${deployApproved.APPROVER}"
                    }
                }
                
                withCredentials([file(credentialsId: 'ssh-private-key-file', variable: 'SSH_KEY')]) {
                    script {
                        echo "Deploying to Production environment..."

                        echo "Image: ${IMAGE_NAME}:${env.IMAGE_TAG}"
                        
                        sh """
                        ssh -i \$SSH_KEY -o StrictHostKeyChecking=no -o ConnectTimeout=30 \$SSH_USER@${GCP_VM_PROD} << 'EOF'
set -e
trap 'echo "[ERROR] Production deployment failed on \$HOSTNAME at \$(date)!" >&2; exit 1' ERR

echo "[INFO] [\$(date)] Starting PRODUCTION deployment..."
echo "[INFO] Image: ${IMAGE_NAME}:${env.IMAGE_TAG}"

cd ${DEPLOY_DIR} || mkdir -p ${DEPLOY_DIR} && cd ${DEPLOY_DIR}

echo "[INFO] Creating backup..."
if docker ps -q -f name=aspnetapp-prod; then
    docker commit aspnetapp-prod ${IMAGE_NAME}:backup-\$(date +%Y%m%d-%H%M%S) || echo "[WARN] Backup failed"
fi

echo "[INFO] Pulling Docker image..."
docker pull ${IMAGE_NAME}:${env.IMAGE_TAG}

echo "[INFO] Stopping existing container..."
docker stop aspnetapp-prod 2>/dev/null || echo "No existing container to stop"
docker rm aspnetapp-prod 2>/dev/null || echo "No existing container to remove"

echo "[INFO] Starting new production container..."
docker run -d \\
    -p ${env.DEPLOY_PORT}:8080 \\
    --name aspnetapp-prod \\
    --restart unless-stopped \\
    -e ASPNETCORE_ENVIRONMENT=Production \\
    ${IMAGE_NAME}:${env.IMAGE_TAG}

echo "[INFO] Waiting for container to be ready..."
sleep 15

if docker ps -f name=aspnetapp-prod --format "table {{.Names}}\\t{{.Status}}" | grep -q "Up"; then
    echo "[SUCCESS] [\$(date)] Production deployment complete!"
    echo "[INFO] Application running on http://\$(hostname -I | awk '{print \$1}'):${env.DEPLOY_PORT}"
else
    echo "[ERROR] Production container failed to start"

    docker logs aspnetapp-prod
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
                echo "=== PIPELINE SUMMARY ==="
                echo "Build Type: ${env.BUILD_TYPE}"
                echo "Source Branch: ${env.SOURCE_BRANCH}"
                echo "Target Branch: ${env.TARGET_BRANCH}"
                echo "Image Tag: ${env.IMAGE_TAG}"
                echo "Deployment: ${env.DEPLOYMENT_ENV}"
                echo "Result: ${currentBuild.result ?: 'SUCCESS'}"
                
                if (env.BUILD_TYPE == 'PR') {
                    echo "Pull Request #${env.CHANGE_ID} build completed"
                }
            }
        }
        
        success {
            script {
                if (env.SHOULD_DEPLOY == 'true') {
                    if (env.TARGET_BRANCH == 'develop') {
                        echo "✅ Development deployment successful!"
                    } else if (env.TARGET_BRANCH == 'main') {
                        echo "✅ Production deployment successful!"
                    }
                } else {
                    echo "✅ Build completed successfully (no deployment)"
                }
            }
        }
        
        failure {
            script {
                def message = "❌ Pipeline failed"
                if (env.BUILD_TYPE == 'PR') {
                    message += " for PR #${env.CHANGE_ID}"
                } else {
                    message += " for branch ${env.BRANCH_NAME}"
                }
                echo message
            }
        }
        
        cleanup {
            script {
                // Cleanup trên build agents
                try {
                    node('agent-builder') {
                        sh """
                        # Remove test containers
                        docker rm -f temp-test aspnetapp-test 2>/dev/null || true
                        
                        # Remove test images
                        docker rmi aspnetapp-test 2>/dev/null || true
                        
                        # Clean up old images (giữ lại 10 images gần nhất)
                        docker images ${IMAGE_NAME} --format "table {{.Repository}}:{{.Tag}}\\t{{.CreatedAt}}" | \\
                        tail -n +11 | awk '{print \$1}' | xargs -r docker rmi 2>/dev/null || true
                        
                        # Prune unused images
                        docker image prune -f || true
                        
                        echo "Cleanup completed"
                        """
                    }
                } catch (Exception e) {
                    echo "Cleanup warning: ${e.getMessage()}"
                }
            }
        }
    }
}