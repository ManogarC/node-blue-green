pipeline {
    agent any

    environment {
        IMAGE_NAME = "manogar27/blue-green-node-app"
        NETWORK = "app-network"
        NGINX_CONTAINER = "nginx-proxy"
        BLUE_CONTAINER = "app-blue"
        GREEN_CONTAINER = "app-green"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat """
                    docker build -t %IMAGE_NAME%:%BUILD_NUMBER% .
                    docker tag %IMAGE_NAME%:%BUILD_NUMBER% %IMAGE_NAME%:latest
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    bat """
                        docker login -u %DOCKER_USER% -p %DOCKER_PASSWORD%
                        docker push %IMAGE_NAME%:%BUILD_NUMBER%
                        docker push %IMAGE_NAME%:latest
                    """
                }
            }
        }

        stage('Determine Environment') {
            steps {
                script {

                    def blueRunning = bat(
                        script: "docker ps --filter name=%BLUE_CONTAINER% --format \"{{.Names}}\"",
                        returnStdout: true
                    ).trim()

                    if (blueRunning == BLUE_CONTAINER) {
                        env.ACTIVE = "blue"
                        env.TARGET = "green"
                    } else {
                        env.ACTIVE = "green"
                        env.TARGET = "blue"
                    }

                    echo "Active environment: ${env.ACTIVE}"
                    echo "Target environment: ${env.TARGET}"
                }
            }
        }

        stage('Deploy New Version') {
            steps {
                script {

                    def targetContainer =
                        env.TARGET == "blue" ? BLUE_CONTAINER : GREEN_CONTAINER

                    bat """
                        docker rm -f ${targetContainer} || exit 0

                        docker run -d ^
                          --name ${targetContainer} ^
                          --network %NETWORK% ^
                          -e APP_ENV=${env.TARGET.capitalize()} ^
                          -e APP_VERSION=%BUILD_NUMBER% ^
                          %IMAGE_NAME%:%BUILD_NUMBER%
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                script {

                    def targetContainer =
                        env.TARGET == "blue" ? BLUE_CONTAINER : GREEN_CONTAINER

                    bat """
                        timeout /t 5 /nobreak

                        docker run --rm ^
                          --network %NETWORK% ^
                          curlimages/curl ^
                          http://${targetContainer}:3000/health
                    """
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                script {

                    def targetContainer =
                        env.TARGET == "blue" ? BLUE_CONTAINER : GREEN_CONTAINER

                    bat """
                        powershell -Command "(Get-Content nginx/nginx.conf) -replace 'server app-(blue|green):3000;', 'server ${targetContainer}:3000;' | Set-Content nginx/nginx.conf"

                        docker exec %NGINX_CONTAINER% nginx -t

                        docker exec %NGINX_CONTAINER% nginx -s reload
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                bat """
                    timeout /t 3 /nobreak

                    curl http://localhost:8090/
                """
            }
        }
    }

    post {
        success {
            echo 'Blue-Green deployment completed successfully.'
        }

        failure {
            echo 'Deployment failed.'
        }
    }
}