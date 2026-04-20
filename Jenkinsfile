pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE_PREFIX = 'placement-tracker'
        GIT_COMMIT_HASH = bat(script: 'git rev-parse --short HEAD', returnStdout: true).replaceAll('[^a-zA-Z0-9]', '').trim()
    }
    
    stages {
        stage('Install Dependencies') {
            parallel {
                stage('Frontend') {
                    steps {
                        dir('client') {
                            bat 'npm ci --silent'
                        }
                    }
                }
                stage('Backend') {
                    steps {
                        dir('server') {
                            bat 'npm ci --silent'
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
                            bat 'npm run build'
                        }
                    }
                }
                stage('Backend Test') {
                    steps {
                        dir('server') {
                            bat 'npm ls --depth=0'
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
                            bat 'echo "Validating frontend code..."'
                            bat 'if not exist "dist" (echo "ERROR: Frontend build failed - dist directory not found" && exit /b 1)'
                            bat 'if not exist "package.json" (echo "ERROR: package.json not found" && exit /b 1)'
                            bat 'if not exist "Dockerfile" (echo "ERROR: Dockerfile not found" && exit /b 1)'
                            bat 'echo "Frontend validation passed!"'
                        }
                    }
                }
                stage('Backend Validation') {
                    steps {
                        dir('server') {
                            bat 'echo "Validating backend code..."'
                            bat 'if not exist "server.js" (echo "ERROR: server.js not found" && exit /b 1)'
                            bat 'if not exist "package.json" (echo "ERROR: package.json not found" && exit /b 1)'
                            bat 'if not exist "Dockerfile" (echo "ERROR: Dockerfile not found" && exit /b 1)'
                            bat 'node -c server.js'
                            bat 'echo "Backend validation passed!"'
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
        
    }
    
    post {
        success {
            script {
                echo "Pipeline completed successfully!"
                archiveArtifacts artifacts: 'client/dist/**/*', fingerprint: true
                cleanWs()
            }
        }
        failure {
            script {
                echo "Pipeline failed!"
                cleanWs()
            }
        }
        always {
            script {
                bat 'docker system prune -f || echo "Docker prune failed"'
            }
        }
    }
}
