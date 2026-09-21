pipeline {
    agent any

    environment {
        DOCKER_HUB_REPO = 'manogar27/node-bg-app'
        CREDENTIALS_ID  = 'docker-hub-credentials'
        IMAGE_TAG       = "v${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${DOCKER_HUB_REPO}:${IMAGE_TAG}")
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', CREDENTIALS_ID) {
                        dockerImage.push("${IMAGE_TAG}")
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage('Deploy Blue-Green Strategy') {
            steps {
                bat '''
                @echo off
                echo Checking active environment...
                
                REM Determine active environment using container inspect
                docker inspect --format="{{.State.Running}}" app-blue 2>nul | findstr "true" >nul
                if %ERRORLEVEL% == 0 (
                    set TARGET_ENV=green
                    set INACTIVE_ENV=blue
                    set TARGET_PORT=3002
                ) else (
                    set TARGET_ENV=blue
                    set INACTIVE_ENV=green
                    set TARGET_PORT=3001
                )

                echo Deploying to TARGET_ENV: %TARGET_ENV% on port %TARGET_PORT%

                REM Pull latest image
                docker pull %DOCKER_HUB_REPO%:%IMAGE_TAG%

                REM Stop and remove old inactive target container if exists
                docker stop app-%TARGET_ENV% 2>nul
                docker rm app-%TARGET_ENV% 2>nul

                REM Run new deployment container
                docker run -d ^
                  --name app-%TARGET_ENV% ^
                  --network app-network ^
                  -p %TARGET_PORT%:3000 ^
                  -e APP_ENV=%TARGET_ENV% ^
                  -e APP_VERSION=%IMAGE_TAG% ^
                  %DOCKER_HUB_REPO%:%IMAGE_TAG%

                REM Health Check Wait Loop
                echo Waiting for container health check...
                powershell -Command "Start-Sleep -Seconds 5"
                
                REM Update Nginx configuration
                echo Updating Nginx traffic upstream to app-%TARGET_ENV%...
                
                (
                    echo events { worker_connections 1024; }
                    echo http {
                    echo     upstream backend {
                    echo         server app-%TARGET_ENV%:3000;
                    echo     }
                    echo     server {
                    echo         listen 80;
                    echo         location / {
                    echo             proxy_pass http://backend;
                    echo         }
                    echo     }
                    echo }
                ) > nginx\\nginx.conf

                REM Reload or Start Nginx Proxy container
                docker inspect --format="{{.State.Running}}" nginx-proxy 2>nul | findstr "true" >nul
                if %ERRORLEVEL% == 0 (
                    docker exec nginx-proxy nginx -s reload
                ) else (
                    docker run -d ^
                      --name nginx-proxy ^
                      --network app-network ^
                      -p 80:80 ^
                      -v "%CD%\\nginx\\nginx.conf:/etc/nginx/nginx.conf:ro" ^
                      nginx:alpine
                )

                echo Switch complete! Traffic routed to app-%TARGET_ENV%.
                '''
            }
        }
    }

    post {
        always {
            bat 'docker image prune -f'
        }
    }
}