pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE_PREFIX = 'placement-tracker'
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
            steps {
                script {
                    // Use simple build number for Docker tags to avoid validation issues
                    def buildNumber = env.BUILD_NUMBER ?: '1'
                    def imageTag = "build${buildNumber}"
                    
                    echo "Building Docker images with tag: ${imageTag}"
                    
                    parallel(
                        'Frontend': {
                            dir('client') {
                                script {
                                    def frontendImage = docker.build("${DOCKER_IMAGE_PREFIX}-frontend:${imageTag}")
                                    echo "Frontend image built: ${DOCKER_IMAGE_PREFIX}-frontend:${imageTag}"
                                }
                            }
                        },
                        'Backend': {
                            dir('server') {
                                script {
                                    def backendImage = docker.build("${DOCKER_IMAGE_PREFIX}-backend:${imageTag}")
                                    echo "Backend image built: ${DOCKER_IMAGE_PREFIX}-backend:${imageTag}"
                                }
                            }
                        }
                    )
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
