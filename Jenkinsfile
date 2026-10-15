
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        GIT_REPO       = 'https://github.com/bbhavanasree2212-code/university-lms-devops.git'
        IMAGE_NAME     = 'bbhavanasree/university-lms'
        DEPLOYMENT     = 'university-lms'
        CONTAINER_NAME = 'university-lms'
        SERVICE_NAME   = 'university-lms-service'
        APP_URL        = 'http://localhost:30084'
    }

    stages {
        stage('1. Checkout Source') {
            steps {
                deleteDir()
                git branch: 'main', url: "${GIT_REPO}"

                bat 'if not exist index.html exit /b 1'
                bat 'if not exist Dockerfile exit /b 1'
                bat 'if not exist deployment.yaml exit /b 1'
                bat 'if not exist k8s\\service.yaml exit /b 1'

                echo 'Source checkout successful.'
            }
        }

        stage('2. Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "build-${env.BUILD_NUMBER}"
                }

                bat 'docker version'
                bat 'docker build -t %IMAGE_NAME%:%IMAGE_TAG% .'
                bat 'docker tag %IMAGE_NAME%:%IMAGE_TAG% %IMAGE_NAME%:latest'

                echo "Built ${env.IMAGE_NAME}:${env.IMAGE_TAG}"
            }
        }

        stage('3. Login and Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    script {
                        // Write the token as plain ASCII bytes.
                        // This avoids CMD interpreting special characters
                        // in the token. The temporary file is deleted below.
                        powershell '''
                            $tokenPath = Join-Path $env:WORKSPACE '.docker-token.tmp'

                            if ([string]::IsNullOrWhiteSpace($env:DOCKER_TOKEN)) {
                                throw 'Docker Hub token is empty.'
                            }

                            [System.IO.File]::WriteAllText(
                                $tokenPath,
                                $env:DOCKER_TOKEN,
                                [System.Text.Encoding]::ASCII
                            )
                        '''

                        try {
                            bat '''
                                @echo off

                                docker login --username "%DOCKER_USER%" --password-stdin < "%WORKSPACE%\\.docker-token.tmp"
                                if errorlevel 1 exit /b 1

                                docker push %IMAGE_NAME%:%IMAGE_TAG%
                                if errorlevel 1 (
                                    docker logout
                                    exit /b 1
                                )

                                docker push %IMAGE_NAME%:latest
                                if errorlevel 1 (
                                    docker logout
                                    exit /b 1
                                )

                                docker logout
                            '''

                            echo 'Docker Hub authentication and image push succeeded.'
                        } finally {
                            powershell '''
                                Remove-Item `
                                    -LiteralPath (Join-Path $env:WORKSPACE '.docker-token.tmp') `
                                    -Force `
                                    -ErrorAction SilentlyContinue
                            '''
                        }
                    }
                }
            }
        }

        stage('4. Validate Kubernetes') {
            steps {
                bat 'kubectl config current-context'
                bat 'kubectl get nodes'
                bat 'kubectl get deployment %DEPLOYMENT%'
                bat 'kubectl get service %SERVICE_NAME%'
            }
        }

        stage('5. Deploy to Kubernetes') {
            steps {
                script {
                    try {
                        bat '''
                            kubectl set image deployment/%DEPLOYMENT% %CONTAINER_NAME%=%IMAGE_NAME%:%IMAGE_TAG%
                            if errorlevel 1 exit /b 1

                            kubectl rollout status deployment/%DEPLOYMENT% --timeout=180s
                            if errorlevel 1 exit /b 1
                        '''

                        echo "Deployment ${env.IMAGE_TAG} completed successfully."
                    } catch (err) {
                        echo 'Deployment failed. Attempting to restore the previous revision.'

                        bat '''
                            kubectl rollout undo deployment/%DEPLOYMENT%
                            if errorlevel 1 exit /b 1

                            kubectl rollout status deployment/%DEPLOYMENT% --timeout=180s
                            if errorlevel 1 exit /b 1
                        '''

                        error('Deployment failed. Rollback was attempted.')
                    }
                }
            }
        }

        stage('6. Verify Deployment') {
            steps {
                bat 'kubectl get deployment %DEPLOYMENT% -o wide'
                bat 'kubectl get pods -l app=university-lms -o wide'
                bat 'kubectl get service %SERVICE_NAME%'
                bat 'kubectl rollout history deployment/%DEPLOYMENT%'

                echo "Open the application at ${env.APP_URL}"
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: University LMS CI/CD pipeline completed.'
            echo 'Image built, pushed, and Kubernetes rollout verified.'
            echo 'Application: http://localhost:30084'
        }

        failure {
            echo 'FAILURE: Inspect Console Output for the first failing stage.'
        }

        always {
            echo 'University LMS pipeline finished.'
        }
    }
}
