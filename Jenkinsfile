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

                docker run -d \
                  --name $GREEN \
                  -p $GREEN_PORT:5000 \
                  $IMAGE
                '''
            }
        }

        stage('Health Check Green') {
            steps {
                script {
                    sh 'sleep 10'

                    def status = sh(
                        script: "curl -f http://localhost:${GREEN_PORT}",
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
                # Update nginx config safely
                sed -i 's/tic-blue/tic-green/g' nginx/nginx.conf

                # Find nginx container dynamically (NO hardcoding)
                NGINX_CONTAINER=$(docker ps -q --filter "name=nginx")

                if [ -n "$NGINX_CONTAINER" ]; then
                    docker exec $NGINX_CONTAINER nginx -s reload
                else
                    echo "⚠️ NGINX container not found - skipping reload"
                    exit 1
                fi
                '''
            }
        }

        stage('Promote Green to Blue') {
            steps {
                sh '''
                docker stop $BLUE || true
                docker rm $BLUE || true

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
