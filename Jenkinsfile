
pipeline {
    agent any

    environment {
        BACKEND_IMAGE  = 'taskmanager-backend'
        FRONTEND_IMAGE = 'taskmanager-frontend'
        IMAGE_TAG      = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Backend Build & Unit Tests') {
            steps {
                dir('backend') {
                    bat 'mvnw.cmd -B clean verify'
                }
            }
        }

        stage('Frontend Build & Lint') {
            steps {
                dir('frontend') {
                    bat '''
                        call npm ci
                        if errorlevel 1 exit /b 1

                        call npm run lint
                        if errorlevel 1 exit /b 1

                        call npm run build
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        bat """
                            "${scannerHome}\\bin\\sonar-scanner.bat" ^
                            -Dsonar.projectKey=taskmanager ^
                            -Dsonar.projectName=TaskManager ^
                            -Dsonar.sources=backend/src/main,frontend/src ^
                            -Dsonar.tests=backend/src/test ^
                            -Dsonar.java.binaries=backend/target/classes ^
                            -Dsonar.java.test.binaries=backend/target/test-classes ^
                            -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/target/**
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build - Backend') {
            steps {
                dir('backend') {
                    bat '''
                        docker build -t %BACKEND_IMAGE%:%IMAGE_TAG% .
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('Docker Build - Frontend') {
            steps {
                dir('frontend') {
                    bat '''
                        docker build -t %FRONTEND_IMAGE%:%IMAGE_TAG% .
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('Verify Docker Images') {
            steps {
                bat '''
                    docker image inspect %BACKEND_IMAGE%:%IMAGE_TAG% >NUL
                    if errorlevel 1 exit /b 1

                    docker image inspect %FRONTEND_IMAGE%:%IMAGE_TAG% >NUL
                    if errorlevel 1 exit /b 1

                    echo Both Docker images built successfully.
                '''
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true,
                  testResults: 'backend/target/surefire-reports/*.xml'
        }

        success {
            echo 'CI Pipeline: SUCCESS'
            echo "Docker images created with tag: ${env.IMAGE_TAG}"
        }

        failure {
            echo 'CI Pipeline: FAILED'
        }
    }
}
