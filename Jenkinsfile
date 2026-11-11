
pipeline {
    agent any

    tools {
        // Uses the scanner configured in Jenkins Tools
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test Backend') {
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
                            -Dsonar.exclusions=**/node_modules/**,**/dist/**
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
    }

    post {
        always {
            junit allowEmptyResults: true,
                  testResults: 'backend/target/surefire-reports/*.xml'
        }

        success {
            echo 'CI Pipeline: SUCCESS'
        }

        failure {
            echo 'CI Pipeline: FAILED'
        }
    }
}
