
pipeline {
    agent any

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
