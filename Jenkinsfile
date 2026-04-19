pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        IMAGE = "tic-app:latest"
        GREEN = "tic-green"
        BLUE  = "tic-blue"
        PORT_BLUE = "5001"
        PORT_GREEN = "5002"
        NGINX_CONTAINER = "nginx-proxy"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/hritikj/Blue-Green.git'
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                docker build -t $IMAGE .
                '''
            }
        }

        stage('Deploy Green') {
            steps {
                sh '''
                docker stop $GREEN || true
                docker rm $GREEN || true

                docker run -d \
                  --name $GREEN \
                  -p $PORT_GREEN:5000 \
                  $IMAGE
                '''
            }
        }

        stage('Health Check Green') {
            steps {
                script {
                    sh 'sleep 10'

                    def status = sh(
                        script: "curl -f http://localhost:${PORT_GREEN}",
                        returnStatus: true
                    )

                    if (status != 0) {
                        error("❌ Green deployment failed health check")
                    }
                }
            }
        }

        stage('Switch Traffic to Green') {
            steps {
                sh '''
                sed -i 's/tic-blue/tic-green/g' nginx/nginx.conf

                # Reload nginx safely (fallback included)
                docker exec $NGINX_CONTAINER nginx -s reload || \
                docker compose exec nginx nginx -s reload
                '''
            }
        }

        stage('Deploy Blue (cleanup old version)') {
            steps {
                sh '''
                docker stop $BLUE || true
                docker rm $BLUE || true

                # Optional: promote green → blue for next cycle
                docker rename $GREEN $BLUE || true
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful - traffic switched to GREEN"
        }

        failure {
            echo "❌ Deployment failed - keeping BLUE running"

            sh '''
            docker stop $GREEN || true
            docker rm $GREEN || true
            '''
        }
    }
}
