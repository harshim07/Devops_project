pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE_PREFIX = 'placement-tracker'
        GIT_COMMIT_HASH = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    }
    
    stages {
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
        
    }
    
    post {
        success {
            echo "Pipeline completed successfully!"
            archiveArtifacts artifacts: 'client/dist/**/*', fingerprint: true
            cleanWs()
        }
        failure {
            echo "Pipeline failed!"
            cleanWs()
        }
        always {
            sh 'docker system prune -f || true'
        }
    }
}
