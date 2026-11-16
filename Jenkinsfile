```groovy
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        BACKEND_IMAGE  = 'taskmanager-backend'
        FRONTEND_IMAGE = 'taskmanager-frontend'
        IMAGE_TAG      = "${BUILD_NUMBER}"

        AWS_DEFAULT_REGION = 'eu-north-1'
        ECR_REGISTRY = '726701726154.dkr.ecr.eu-north-1.amazonaws.com'
        ECR_REPO = 'taskmanager'

        EKS_CLUSTER = 'taskmanager-dev-eks'
        K8S_NAMESPACE = 'taskmanager-dev'
        HELM_RELEASE = 'taskmanager-dev'
        HELM_CHART = 'taskmanager-helm'
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

                    echo Both images tagged for ECR.
                '''
            }
        }

        stage('Push Backend Image to ECR') {
            steps {
                retry(3) {
                    timeout(time: 20, unit: 'MINUTES') {
                        bat '''
                            docker push %ECR_REGISTRY%/%ECR_REPO%:backend-build-%IMAGE_TAG%
                            if errorlevel 1 exit /b 1
                        '''
                    }
                }
            }
        }

        stage('Push Frontend Image to ECR') {
            steps {
                retry(3) {
                    timeout(time: 15, unit: 'MINUTES') {
                        bat '''
                            docker push %ECR_REGISTRY%/%ECR_REPO%:frontend-build-%IMAGE_TAG%
                            if errorlevel 1 exit /b 1
                        '''
                    }
                }
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

        stage('Deploy to EKS') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'jenkins_ecr_access_key',
                        variable: 'AWS_ACCESS_KEY_ID'
                    ),
                    string(
                        credentialsId: 'jenkins_ecr_secret_key',
                        variable: 'AWS_SECRET_ACCESS_KEY'
                    ),
                    usernamePassword(
                        credentialsId: 'taskmanager-db-credentials',
                        usernameVariable: 'DB_USERNAME',
                        passwordVariable: 'DB_PASSWORD'
                    ),
                    string(
                        credentialsId: 'taskmanager-jwt-secret',
                        variable: 'JWT_SECRET'
                    )
                ]) {
                    bat '''
                        @echo off

                        echo Checking AWS identity...
                        aws sts get-caller-identity
                        if errorlevel 1 exit /b 1

                        echo Configuring EKS access...
                        aws eks update-kubeconfig --name %EKS_CLUSTER% --region %AWS_DEFAULT_REGION%
                        if errorlevel 1 exit /b 1

                        echo Checking Kubernetes access...
                        kubectl get nodes
                        if errorlevel 1 exit /b 1

                        echo Ensuring namespace exists...
                        kubectl create namespace %K8S_NAMESPACE% --dry-run=client -o yaml | kubectl apply -f -
                        if errorlevel 1 exit /b 1

                        echo Creating/updating application secrets...

                        powershell.exe -NoProfile -NonInteractive -ExecutionPolicy Bypass -Command ^
                          "$ErrorActionPreference = 'Stop';" ^
                          "$db = @{apiVersion='v1';kind='Secret';metadata=@{name='taskmanager-db-secret';namespace=$env:K8S_NAMESPACE};type='Opaque';data=@{POSTGRES_USER=[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($env:DB_USERNAME));POSTGRES_PASSWORD=[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($env:DB_PASSWORD))}};" ^
                          "$dbJson = $db | ConvertTo-Json -Depth 5 -Compress;" ^
                          "$dbJson | kubectl apply -f -;" ^
                          "if ($LASTEXITCODE -ne 0) { exit 1 };" ^
                          "$jwt = @{apiVersion='v1';kind='Secret';metadata=@{name='taskmanager-backend-secret';namespace=$env:K8S_NAMESPACE};type='Opaque';data=@{JWT_SECRET=[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($env:JWT_SECRET))}};" ^
                          "$jwtJson = $jwt | ConvertTo-Json -Depth 5 -Compress;" ^
                          "$jwtJson | kubectl apply -f -;" ^
                          "if ($LASTEXITCODE -ne 0) { exit 1 }"

                        if errorlevel 1 exit /b 1

                        echo Deploying current build through Helm...

                        helm upgrade --install %HELM_RELEASE% %HELM_CHART% ^
                          --namespace %K8S_NAMESPACE% ^
                          --set-string backend.image.tag=backend-build-%IMAGE_TAG% ^
                          --set-string frontend.image.tag=frontend-build-%IMAGE_TAG%

                        if errorlevel 1 exit /b 1

                        echo Checking backend rollout...
                        kubectl rollout status deployment/taskmanager-backend -n %K8S_NAMESPACE% --timeout=300s
                        if errorlevel 1 exit /b 1

                        echo Checking frontend rollout...
                        kubectl rollout status deployment/taskmanager-frontend -n %K8S_NAMESPACE% --timeout=300s
                        if errorlevel 1 exit /b 1

                        echo Checking PostgreSQL rollout...
                        kubectl rollout status deployment/taskmanager-postgres -n %K8S_NAMESPACE% --timeout=300s
                        if errorlevel 1 exit /b 1

                        echo Deployment resources:
                        kubectl get deployments,services,pods -n %K8S_NAMESPACE%

                        echo EKS DEPLOYMENT COMPLETED SUCCESSFULLY.
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
            echo 'CI/CD PIPELINE COMPLETED SUCCESSFULLY'
            echo "Backend: ${env.ECR_REGISTRY}/${env.ECR_REPO}:backend-build-${env.IMAGE_TAG}"
            echo "Frontend: ${env.ECR_REGISTRY}/${env.ECR_REPO}:frontend-build-${env.IMAGE_TAG}"
            echo "Deployed to EKS namespace: ${env.K8S_NAMESPACE}"
        }

        failure {
            echo 'CI/CD PIPELINE FAILED'
        }
    }
}