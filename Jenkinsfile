
pipeline {
    agent any

    environment {
        BACKEND_IMAGE  = 'taskmanager-backend'
        FRONTEND_IMAGE = 'taskmanager-frontend'
        IMAGE_TAG      = "${BUILD_NUMBER}"

        AWS_DEFAULT_REGION = 'eu-north-1'
        ECR_REGISTRY = '726701726154.dkr.ecr.eu-north-1.amazonaws.com'
        ECR_REPO = 'taskmanager'
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

        stage('Trivy Security Scan - Backend') {
            steps {
                catchError(
                    buildResult: 'SUCCESS',
                    stageResult: 'FAILURE'
                ) {
                    bat '''
                        docker run --rm ^
                          -v //var/run/docker.sock:/var/run/docker.sock ^
                          -v trivy-cache:/root/.cache/trivy ^
                          aquasec/trivy:0.75.0 image ^
                          --scanners vuln ^
                          --pkg-types os ^
                          --timeout 10m ^
                          --severity HIGH,CRITICAL ^
                          --exit-code 1 ^
                          %BACKEND_IMAGE%:%IMAGE_TAG%

                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('AWS ECR Login') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'jenkins_ecr_access_key',
                        variable: 'AWS_ACCESS_KEY_ID'
                    ),
                    string(
                        credentialsId: 'jenkins_ecr_secret_key',
                        variable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    bat '''
                        @echo off
                        aws ecr get-login-password --region %AWS_DEFAULT_REGION% | docker login --username AWS --password-stdin %ECR_REGISTRY%
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('Tag Docker Images for ECR') {
            steps {
                bat '''
                    docker tag %BACKEND_IMAGE%:%IMAGE_TAG% %ECR_REGISTRY%/%ECR_REPO%:backend-build-%IMAGE_TAG%
                    if errorlevel 1 exit /b 1

                    docker tag %FRONTEND_IMAGE%:%IMAGE_TAG% %ECR_REGISTRY%/%ECR_REPO%:frontend-build-%IMAGE_TAG%
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Push Backend Image to ECR') {
            steps {
                bat '''
                    docker push %ECR_REGISTRY%/%ECR_REPO%:backend-build-%IMAGE_TAG%
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Push Frontend Image to ECR') {
            steps {
                bat '''
                    docker push %ECR_REGISTRY%/%ECR_REPO%:frontend-build-%IMAGE_TAG%
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Verify ECR Images') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'jenkins_ecr_access_key',
                        variable: 'AWS_ACCESS_KEY_ID'
                    ),
                    string(
                        credentialsId: 'jenkins_ecr_secret_key',
                        variable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    bat '''
                        @echo off
                        aws ecr describe-images ^
                          --repository-name %ECR_REPO% ^
                          --region %AWS_DEFAULT_REGION% ^
                          --image-ids imageTag=backend-build-%IMAGE_TAG% ^
                          --query "imageDetails[0].imageDigest" ^
                          --output text
                        if errorlevel 1 exit /b 1

                        aws ecr describe-images ^
                          --repository-name %ECR_REPO% ^
                          --region %AWS_DEFAULT_REGION% ^
                          --image-ids imageTag=frontend-build-%IMAGE_TAG% ^
                          --query "imageDetails[0].imageDigest" ^
                          --output text
                        if errorlevel 1 exit /b 1

                        echo Both Docker images verified in AWS ECR.
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
            echo 'CI PIPELINE COMPLETED SUCCESSFULLY'
            echo "Backend: ${env.ECR_REGISTRY}/${env.ECR_REPO}:backend-build-${env.IMAGE_TAG}"
            echo "Frontend: ${env.ECR_REGISTRY}/${env.ECR_REPO}:frontend-build-${env.IMAGE_TAG}"
        }

        failure {
            echo 'CI PIPELINE FAILED'
        }
    }
}
