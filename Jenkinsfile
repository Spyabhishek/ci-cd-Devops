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
                        set -e

                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        docker tag cicd-app:${BUILD_NUMBER} \
                            "$DOCKER_USERNAME/cicd-app:${BUILD_NUMBER}"

                        docker push \
                            "$DOCKER_USERNAME/cicd-app:${BUILD_NUMBER}"

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

                    sshagent(['oracle-vm-ssh']) {

                        sh '''
                            set -e

                            NEW_IMAGE="$DOCKER_USERNAME/cicd-app:${BUILD_NUMBER}"

                            echo "========================================"
                            echo "Deploying to Oracle VM"
                            echo "Host: $ORACLE_HOST"
                            echo "Image: $NEW_IMAGE"
                            echo "========================================"

                            ssh -o StrictHostKeyChecking=no \
                                ubuntu@$ORACLE_HOST \
                                "docker pull $NEW_IMAGE"

                            ssh -o StrictHostKeyChecking=no \
                                ubuntu@$ORACLE_HOST << EOF

                                set -e

                                echo "Checking current deployment..."

                                if docker inspect cicd-app >/dev/null 2>&1; then

                                    PREVIOUS_IMAGE=\$(docker inspect cicd-app \
                                        --format='{{.Config.Image}}')

                                    echo "\$PREVIOUS_IMAGE" > /tmp/cicd-previous-image.txt

                                    echo "Previous image: \$PREVIOUS_IMAGE"

                                else

                                    echo "" > /tmp/cicd-previous-image.txt

                                    echo "No previous deployment found."

                                fi

                                echo "Stopping old container..."

                                docker stop cicd-app || true
                                docker rm cicd-app || true

                                echo "Starting new container..."

                                docker run -d \
                                    --name cicd-app \
                                    --restart unless-stopped \
                                    -p 8081:8081 \
                                    "$NEW_IMAGE"

                                echo "New container started."

                                docker ps --filter "name=cicd-app"

                                EOF

                            echo "Deployment completed successfully."
                        '''
                    }
                }
            }
        }

        stage('Health Check') {
            steps {
                sshagent(['oracle-vm-ssh']) {

                    script {

                        def healthCheck = sh(
                            script: '''
                                for i in $(seq 1 12); do

                                    echo "Health check attempt $i..."

                                    if ssh -o StrictHostKeyChecking=no \
                                        ubuntu@$ORACLE_HOST \
                                        "curl --fail --silent http://localhost:8081/hello"; then

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

                            echo "========================================"
                            echo "HEALTH CHECK FAILED"
                            echo "Starting rollback..."
                            echo "========================================"

                            sh '''
                                ssh -o StrictHostKeyChecking=no \
                                    ubuntu@$ORACLE_HOST << 'EOF'

                                    set -e

                                    PREVIOUS_IMAGE=$(cat /tmp/cicd-previous-image.txt 2>/dev/null || true)

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

                                    EOF
                            '''

                            echo "Checking rolled-back application..."

                            def rollbackHealthCheck = sh(
                                script: '''
                                    for i in $(seq 1 12); do

                                        echo "Rollback health check attempt $i..."

                                        if ssh -o StrictHostKeyChecking=no \
                                            ubuntu@$ORACLE_HOST \
                                            "curl --fail --silent http://localhost:8081/hello"; then

                                            echo ""
                                            echo "Rolled-back application is healthy!"
                                            exit 0

                                        fi

                                        echo "Rolled-back application not ready yet..."
                                        sleep 5

                                    done

                                    echo "Rollback health check failed."
                                    exit 1
                                ''',
                                returnStatus: true
                            )

                            if (rollbackHealthCheck != 0) {

                                error(
                                    "CRITICAL: Deployment failed AND rollback health check failed!"
                                )

                            }

                            error(
                                "Deployment failed. Previous version restored successfully."
                            )
                        }
                    }
                }
            }
        }

        stage('Docker Cleanup') {
            steps {
                sh '''
                    echo "Cleaning dangling Docker images on Jenkins..."

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