pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        IMAGE = "tic-app:latest"
        GREEN = "tic-green"
        BLUE = "tic-blue"
        NGINX = "nginx-proxy"
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

                    docker run -d -p 5002:5000 --name $GREEN $IMAGE
                '''
            }
        }

        stage('Health Check Green') {
            steps {
                script {
                    try {
                        sh '''
                            sleep 10
                            curl -f http://localhost:5002
                        '''
                    } catch (err) {
                        error("❌ Green deployment failed")
                    }
                }
            }
        }

        stage('Switch Traffic to Green') {
            steps {
                sh '''
                    echo "Switching traffic to GREEN..."

                    sed -i 's/tic-blue/tic-green/g' nginx/nginx.conf

                    # Ensure nginx is running
                    if ! docker ps --format '{{.Names}}' | grep -q "^$NGINX$"; then
                        echo "⚠️ nginx not running. Starting..."
                        docker start $NGINX || exit 1
                    fi

                    # Validate nginx config before reload
                    docker exec $NGINX nginx -t

                    # Reload nginx
                    docker exec $NGINX nginx -s reload
                '''
            }
        }

        stage('Stop Blue') {
            steps {
                sh '''
                    echo "Stopping BLUE container..."

                    docker stop $BLUE || true
                    docker rm $BLUE || true
                '''
            }
        }
    }

    post {
        failure {
            echo "🚨 Green failed → Rolling back to BLUE"

            sh '''
                sed -i 's/tic-green/tic-blue/g' nginx/nginx.conf

                if docker ps --format '{{.Names}}' | grep -q "^$NGINX$"; then
                    docker exec $NGINX nginx -s reload
                fi
            '''
        }
    }
}
