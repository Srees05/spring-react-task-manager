pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify Project') {
            steps {
                bat '''
                    echo Checking project structure...
                    if not exist backend\\pom.xml exit /b 1
                    if not exist frontend\\package.json exit /b 1
                    echo Project structure verified.
                '''
            }
        }
    }
}