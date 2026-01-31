pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'localhost'  // or your registry if pushing
        IMAGE_NAME = 'food-delivery'
    }

    tools {
        nodejs 'NodeJS-18'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                }
            }
        }

        stage('Install Dependencies') {
            parallel {
                stage('Backend') {
                    steps {
                        dir('backend') {
                            sh 'npm ci'
                        }
                    }
                }
                stage('Frontend') {
                    steps {
                        dir('frontend') {
                            sh 'npm ci'
                        }
                    }
                }
            }
        }

        stage('SAST - SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=food-delivery \
                          -Dsonar.sources=backend/,frontend/src/ \
                          -Dsonar.exclusions=**/node_modules/**,**/*dist/** \
                          -Dsonar.javascript.lcov.reportPaths=backend/coverage/lcov.info,frontend/coverage/lcov.info
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Dependency Check') {
            parallel {
                stage('Backend Audit') {
                    steps {
                        dir('backend') {
                            sh 'npm audit --audit-level=high --json > ../npm-audit-backend.json || true'
                            archiveArtifacts artifacts: 'npm-audit-backend.json'
                        }
                    }
                }
                stage('Frontend Audit') {
                    steps {
                        dir('frontend') {
                            sh 'npm audit --audit-level=high --json > ../npm-audit-frontend.json || true'
                            archiveArtifacts artifacts: 'npm-audit-frontend.json'
                        }
                    }
                }
                stage('OWASP Dependency-Check') {
                    steps {
                        dependencyCheck additionalArguments: '--scan ./backend --scan ./frontend --format HTML --format JSON --prettyPrint'
                        dependencyCheckPublisher pattern: 'dependency-check-report.json'
                    }
                }
            }
        }

        stage('Unit Tests') {
            parallel {
                stage('Backend Tests') {
                    steps {
                        dir('backend') {
                            sh 'npm test -- --coverage --coverageReporters=lcov'
                        }
                    }
                    post {
                        always {
                            publishHTML target: [
                                reportDir: 'backend/coverage/lcov-report',
                                reportFiles: 'index.html',
                                reportName: 'Backend Coverage'
                            ]
                        }
                    }
                }
                stage('Frontend Tests') {
                    steps {
                        dir('frontend') {
                            sh 'npm test -- --coverage --watchAll=false'
                        }
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}-backend:${GIT_COMMIT_SHORT}", "./backend")
                    docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}-frontend:${GIT_COMMIT_SHORT}", "./frontend")
                }
            }
        }

        stage('Container Security Scan') {
            steps {
                sh '''
                    trivy image --severity HIGH,CRITICAL \
                      --format json \
                      --output trivy-backend.json \
                      ${DOCKER_REGISTRY}/${IMAGE_NAME}-backend:${GIT_COMMIT_SHORT}

                    trivy image --severity HIGH,CRITICAL \
                      --format json \
                      --output trivy-frontend.json \
                      ${DOCKER_REGISTRY}/${IMAGE_NAME}-frontend:${GIT_COMMIT_SHORT}
                '''
                archiveArtifacts artifacts: 'trivy-*.json'
            }
        }

        // Add stages for DAST, Deploy, etc. later
    }

    post {
        always {
            cleanWs()
        }
    }
}
