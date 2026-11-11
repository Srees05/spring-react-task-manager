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

    }

    post {
        always {
            junit allowEmptyResults: true,
                  testResults: 'backend/target/surefire-reports/*.xml'
        }

        success {
            echo 'Backend Build & Tests: SUCCESS'
        }

        failure {
            echo 'Backend Build & Tests: FAILED'
        }
    }
}