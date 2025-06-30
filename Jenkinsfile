pipeline {
    agent none

    environment {
        // Thông tin chung test 1
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
                    
                    // Set deployment port based on target branch
                    if (env.TARGET_BRANCH == 'develop') {
                        env.DEPLOY_PORT = env.DEV_PORT
                        env.DEPLOYMENT_ENV = 'development'
                    } else if (env.TARGET_BRANCH == 'main') {
                        env.DEPLOY_PORT = env.PROD_PORT
                        env.DEPLOYMENT_ENV = 'production'
                    }
                    
                    echo "Processing pull request #${env.CHANGE_ID}"
                    echo "Source branch: ${env.CHANGE_BRANCH}"
                    echo "Target branch: ${env.TARGET_BRANCH}"
                    echo "Deployment environment: ${env.DEPLOYMENT_ENV}"
                    echo "Deployment port: ${env.DEPLOY_PORT}"
                }
            }
        }

        stage('Checkout') {
            agent { label 'agent-builder' }
            steps {
                script {
                    // Checkout the PR branch (source branch), not target branch
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "pr/${env.CHANGE_ID}/merge"]],
                        userRemoteConfigs: [[
                            url: "https://github.com/Qu11et/aspnetapp.git",
                            credentialsId: 'github-token'
                        ]],
                        extensions: [
                            [$class: 'CleanBeforeCheckout'],
                            [$class: 'PruneStaleBranch']
                        ]
                    ])
                }

                sh 'ls -la'
                sh 'pwd'
                sh 'git log --oneline -5'
            }
        }

        stage('Build') {
            agent { label 'agent-builder' }
            steps {
                script {
                    // Create unique tag for this build
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.CHANGE_ID}"
                    
                    echo "Building Docker image: ${IMAGE_NAME}:${env.IMAGE_TAG}"
                    
                    sh """
                    docker build --pull -t ${IMAGE_NAME}:${env.IMAGE_TAG} .
                    """
                    
                    // Tag as latest for the environment
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
                    docker build -t aspnetapp-test -f Dockerfile.test .
                    docker images | grep aspnetapp-test
                    
                    # Run tests in container
                    docker run --rm aspnetapp-test || {
                        echo "Tests failed!"
                        exit 1
                    }
                    """
                }
            }
        }

        stage('Security Scan') {
            agent { label 'agent-builder' }
            steps {
                script {
                    echo "Running basic security checks..."
                    sh """
                    # Basic vulnerability scan
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        -v \$(pwd):/tmp/.hadolint hadolint/hadolint:latest \
                        hadolint /tmp/Dockerfile || echo "Dockerfile linting completed with warnings"
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            agent { label 'agent-builder' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    
                    # Push specific tag
                    docker push ${IMAGE_NAME}:${env.IMAGE_TAG}
                    
                    # Push environment-specific latest tag
                    if [ "${env.TARGET_BRANCH}" = "develop" ]; then
                        docker push ${IMAGE_NAME}:dev-latest
                    elif [ "${env.TARGET_BRANCH}" = "main" ]; then
                        docker push ${IMAGE_NAME}:prod-latest
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
                    expression { env.CHANGE_ID != null } // Is PR
                    expression { env.TARGET_BRANCH == 'develop' } // PR targeting develop
                }
            }
            steps {
                withCredentials([file(credentialsId: 'ssh-private-key-file', variable: 'SSH_KEY')]) {
                    script {
                        echo "Deploying to Development environment..."
                        sh """
                        ssh -i \$SSH_KEY -o StrictHostKeyChecking=no -o ConnectTimeout=30 \$SSH_USER@${GCP_VM_DEV} << 'EOF'
set -e
trap 'echo "[ERROR] Deployment failed on \$HOSTNAME at \$(date)!" >&2; exit 1' ERR

echo "[INFO] [\$(date)] Starting deployment to DEV environment..."
echo "[INFO] Switching to deployment directory..."
mkdir -p ${DEPLOY_DIR} && cd ${DEPLOY_DIR}

echo "[INFO] Pulling latest Docker image..."
docker pull ${IMAGE_NAME}:${env.IMAGE_TAG}

echo "[INFO] Stopping existing container..."
if docker ps -q -f name=aspnetapp-dev; then
    docker stop aspnetapp-dev
    echo "[INFO] Stopped existing container"
fi

echo "[INFO] Removing old container..."
if docker ps -aq -f name=aspnetapp-dev; then
    docker rm aspnetapp-dev
    echo "[INFO] Removed old container"
fi

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
    echo "[SUCCESS] [\$(date)] Dev Deployment complete on \$HOSTNAME"
    echo "[INFO] Application is running on port ${env.DEPLOY_PORT}"
else
    echo "[ERROR] Container failed to start properly"
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
                    expression { env.CHANGE_ID != null } // Is PR
                    expression { env.TARGET_BRANCH == 'main' } // PR targeting main
                }
            }
            steps {
                script {
                    // Manual approval for production
                    def deployApproved = input(
                        message: 'Deploy to Production?',
                        parameters: [
                            choice(choices: ['Deploy', 'Abort'], 
                                   description: 'Choose deployment action', 
                                   name: 'DEPLOY_ACTION')
                        ]
                    )
                    
                    if (deployApproved != 'Deploy') {
                        error("Production deployment was cancelled by user")
                    }
                }
                
                withCredentials([file(credentialsId: 'ssh-private-key-file', variable: 'SSH_KEY')]) {
                    script {
                        echo "Deploying to Production environment..."
                        sh """
                        ssh -i \$SSH_KEY -o StrictHostKeyChecking=no -o ConnectTimeout=30 \$SSH_USER@${GCP_VM_PROD} << 'EOF'
set -e
trap 'echo "[ERROR] Production deployment failed on \$HOSTNAME at \$(date)!" >&2; exit 1' ERR

echo "[INFO] [\$(date)] Starting deployment to PRODUCTION environment..."
echo "[INFO] Switching to deployment directory..."
mkdir -p ${DEPLOY_DIR} && cd ${DEPLOY_DIR}

echo "[INFO] Pulling latest Docker image..."
docker pull ${IMAGE_NAME}:${env.IMAGE_TAG}

echo "[INFO] Creating backup of current container..."
if docker ps -q -f name=aspnetapp-prod; then
    docker commit aspnetapp-prod ${IMAGE_NAME}:backup-\$(date +%Y%m%d-%H%M%S) || echo "[WARN] Could not create backup"
fi

echo "[INFO] Stopping existing container..."
if docker ps -q -f name=aspnetapp-prod; then
    docker stop aspnetapp-prod
    echo "[INFO] Stopped existing container"
fi

echo "[INFO] Removing old container..."
if docker ps -aq -f name=aspnetapp-prod; then
    docker rm aspnetapp-prod
    echo "[INFO] Removed old container"
fi

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
    echo "[SUCCESS] [\$(date)] Production Deployment complete on \$HOSTNAME"
    echo "[INFO] Application is running on port ${env.DEPLOY_PORT}"
else
    echo "[ERROR] Production container failed to start properly"
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
                echo "Pipeline completed for PR #${env.CHANGE_ID}"
                echo "Build result: ${currentBuild.result ?: 'SUCCESS'}"
            }
        }
        
        success {
            script {
                if (env.TARGET_BRANCH == 'develop') {
                    echo "✅ Development deployment successful!"
                } else if (env.TARGET_BRANCH == 'main') {
                    echo "✅ Production deployment successful!"
                }
            }
        }
        
        failure {
            script {
                echo "❌ Pipeline failed for PR #${env.CHANGE_ID}"
                // Có thể thêm notification đến Slack/Teams ở đây
            }
        }
        
        cleanup {
            script {
                // Cleanup Docker images on build agents
                node('agent-builder') {
                    sh """
                    # Remove test images
                    docker rmi aspnetapp-test || true
                    
                    # Clean up old images (keep last 5)
                    docker images ${IMAGE_NAME} --format "table {{.Repository}}:{{.Tag}}\\t{{.CreatedAt}}" | \\
                    tail -n +6 | awk '{print \$1}' | xargs -r docker rmi || true
                    
                    # Prune unused images
                    docker image prune -f || true
                    """
                }
            }
        }
    }
}