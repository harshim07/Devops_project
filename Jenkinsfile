pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE_PREFIX = 'placement-tracker'
        GIT_COMMIT_HASH = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
                echo "Checked out commit: ${env.GIT_COMMIT_HASH}"
            }
        }
        
        stage('Install Dependencies') {
            parallel {
                stage('Frontend') {
                    steps {
                        dir('client') {
                            sh 'npm ci --silent'
                        }
                    }
                }
                stage('Backend') {
                    steps {
                        dir('server') {
                            sh 'npm ci --silent'
                        }
                    }
                }
            }
        }
        
        stage('Build & Test') {
            parallel {
                stage('Frontend Build') {
                    steps {
                        dir('client') {
                            sh 'npm run build'
                        }
                    }
                }
                stage('Backend Test') {
                    steps {
                        dir('server') {
                            sh 'npm ls --depth=0'
                        }
                    }
                }
            }
        }
        
        stage('Validate Before Build') {
            parallel {
                stage('Frontend Validation') {
                    steps {
                        dir('client') {
                            sh 'echo "Validating frontend code..."'
                            sh 'if [ ! -d "dist" ]; then echo "ERROR: Frontend build failed - dist directory not found"; exit 1; fi'
                            sh 'if [ ! -f "package.json" ]; then echo "ERROR: package.json not found"; exit 1; fi'
                            sh 'if [ ! -f "Dockerfile" ]; then echo "ERROR: Dockerfile not found"; exit 1; fi'
                            sh 'echo "Frontend validation passed!"'
                        }
                    }
                }
                stage('Backend Validation') {
                    steps {
                        dir('server') {
                            sh 'echo "Validating backend code..."'
                            sh 'if [ ! -f "server.js" ]; then echo "ERROR: server.js not found"; exit 1; fi'
                            sh 'if [ ! -f "package.json" ]; then echo "ERROR: package.json not found"; exit 1; fi'
                            sh 'if [ ! -f "Dockerfile" ]; then echo "ERROR: Dockerfile not found"; exit 1; fi'
                            sh 'node -c server.js'
                            sh 'echo "Backend validation passed!"'
                        }
                    }
                }
            }
        }
        
        stage('Build Docker Images') {
            parallel {
                stage('Frontend') {
                    steps {
                        script {
                            def frontendImage = docker.build("${DOCKER_IMAGE_PREFIX}-frontend:${env.GIT_COMMIT_HASH}", "./client")
                            frontendImage.tag("${DOCKER_IMAGE_PREFIX}-frontend:latest")
                        }
                    }
                }
                stage('Backend') {
                    steps {
                        script {
                            def backendImage = docker.build("${DOCKER_IMAGE_PREFIX}-backend:${env.GIT_COMMIT_HASH}", "./server")
                            backendImage.tag("${DOCKER_IMAGE_PREFIX}-backend:latest")
                        }
                    }
                }
            }
        }
        
        stage('Test with Docker Compose') {
            steps {
                script {
                    try {
                        sh 'docker-compose -f docker-compose.yml up -d'
                        
                        sh '''
                            echo "Waiting for services to be ready..."
                            timeout 300 {
                                until curl -f http://localhost:5000/; do
                                    echo "Backend not ready, waiting..."
                                    sleep 5
                                done
                            }
                            until curl -f http://localhost:3000/; do
                                echo "Frontend not ready, waiting..."
                                sleep 5
                            done
                            echo "Application is running!"
                        '''
                        
                    } catch (Exception e) {
                        sh 'docker-compose -f docker-compose.yml logs'
                        error "Application test failed"
                    } finally {
                        sh 'docker-compose -f docker-compose.yml down -v'
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo "Pipeline completed successfully!"
            archiveArtifacts artifacts: 'client/dist/**/*', fingerprint: true
            cleanWs()
        }
        failure {
            echo "Pipeline failed!"
            sh 'docker-compose -f docker-compose.yml logs || true'
            cleanWs()
        }
        always {
            sh 'docker-compose -f docker-compose.yml down -v || true'
            sh 'docker system prune -f || true'
        }
    }
}
