pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        IMAGE = "tic-app:latest"
        GREEN = "tic-green"
        BLUE  = "tic-blue"
        GREEN_PORT = "5002"
        BLUE_PORT  = "5001"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/hritikj/Blue-Green.git'
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build -t $IMAGE .'
            }
        }

        stage('Deploy Green') {
            steps {
                sh '''
                docker stop $GREEN || true
                docker rm $GREEN || true

                docker-compose up -d
                '''
            }
        }

        stage('Health Check Green') {
            steps {
                script {
                    def success = false

                    for (int i = 0; i < 10; i++) {
                        sleep 3

                        def status = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${GREEN_PORT} || true",
                            returnStdout: true
                        ).trim()

                        if (status == "200") {
                            success = true
                            break
                        }

                        echo "Waiting for green... attempt ${i+1}"
                    }

                    if (!success) {
                        error("❌ Green deployment failed health check")
                    }
                }
            }
        }

        stage('Switch Traffic to Green') {
            steps {
                sh """
                sed -i 's/tic-blue/tic-green/g' nginx/nginx.conf

                NGINX_CONTAINER=$(docker ps -q --filter "name=nginx")

                if [ -n "$NGINX_CONTAINER" ]; then
                    docker exec $NGINX_CONTAINER nginx -s reload
                else
                    echo "❌ NGINX container not found"
                    exit 1
                fi
                """
            }
        }

        stage('Stop Blue') {
            steps {
                sh """
                    docker stop \$BLUE || true
                    docker rm \$BLUE || true
                """
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful - Green is live"
        }

        failure {
            echo "❌ Deployment failed - Blue remains active"

            sh '''
            docker stop $GREEN || true
            docker rm $GREEN || true
            '''
        }
    }
}
