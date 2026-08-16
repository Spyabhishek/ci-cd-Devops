pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                sh './mvnw clean test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh './mvnw sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package') {
            steps {
                sh './mvnw package -DskipTests'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t cicd-app:${BUILD_NUMBER} .'
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                    trivy image \
                      --timeout 20m \
                      --scanners vuln \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      cicd-app:${BUILD_NUMBER}
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        docker tag cicd-app:${BUILD_NUMBER} \
                            $DOCKER_USERNAME/cicd-app:${BUILD_NUMBER}

                        docker push \
                            $DOCKER_USERNAME/cicd-app:${BUILD_NUMBER}

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                        set -e

                        NEW_IMAGE="$DOCKER_USERNAME/cicd-app:${BUILD_NUMBER}"

                        echo "New image: $NEW_IMAGE"

                        if docker inspect cicd-app >/dev/null 2>&1; then
                            PREVIOUS_IMAGE=$(docker inspect cicd-app \
                                --format='{{.Config.Image}}')

                            echo "Previous image: $PREVIOUS_IMAGE"
                        else
                            PREVIOUS_IMAGE=""
                            echo "No previous deployment found."
                        fi

                        echo "$PREVIOUS_IMAGE" > previous_image.txt

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        docker pull "$NEW_IMAGE"

                        docker stop cicd-app || true
                        docker rm cicd-app || true

                        docker run -d \
                            --name cicd-app \
                            --restart unless-stopped \
                            -p 8081:8081 \
                            "$NEW_IMAGE"

                        docker logout
                    '''
                }
            }
        }

stage('Health Check') {
    steps {
        script {

            def healthCheck = sh(
                script: '''
                    for i in $(seq 1 12); do

                        echo "Health check attempt $i..."

                        if curl --fail --silent \
                            http://localhost:8081/hello; then

                            echo ""
                            echo "Application is healthy!"
                            exit 0
                        fi

                        echo "Application not ready yet..."
                        sleep 5
                    done

                    echo "Health check failed after 60 seconds."
                    exit 1
                ''',
                returnStatus: true
            )

            if (healthCheck != 0) {

                echo "Health check FAILED. Starting rollback..."

                sh '''
                    PREVIOUS_IMAGE=$(cat previous_image.txt)

                    if [ -n "$PREVIOUS_IMAGE" ]; then

                        echo "Rolling back to: $PREVIOUS_IMAGE"

                        docker stop cicd-app || true
                        docker rm cicd-app || true

                        docker run -d \
                            --name cicd-app \
                            --restart unless-stopped \
                            -p 8081:8081 \
                            "$PREVIOUS_IMAGE"

                        echo "Rollback completed."

                    else

                        echo "No previous image available for rollback."
                        exit 1

                    fi
                '''

                error("Deployment failed. Previous version restored.")
            }
        }
    }
}

stage('Docker Cleanup') {
    steps {
        sh '''
            echo "Cleaning dangling Docker images..."

            docker image prune -f

            echo "Docker cleanup completed."
        '''
    }
}
    }

    post {

        success {
            echo 'CI/CD pipeline completed successfully!'
        }

        failure {
            echo 'CI/CD pipeline failed!'
        }
    }
}