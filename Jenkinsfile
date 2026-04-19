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
        NETWORK = "app-network"
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

        stage('Create Network') {
            steps {
                sh 'docker network create $NETWORK || true'
            }
        }

        stage('Deploy Green') {
            steps {
                sh '''
                     docker stop $GREEN || true
                     docker rm $GREEN || true

                     docker stop $NGINX || true
                     docker rm $NGINX || true
                     
                     docker-compose down || true
                     docker-compose up -d
                '''
            }
        }

        stage('Health Check Green') {
            steps {
                script {
                    try {
                        sh '''
                            echo "Waiting for GREEN to be ready..."
                            for i in {1..10}; do
                                curl -f http://localhost:5002 && exit 0
                                sleep 3
                            done
                            exit 1
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

                    echo "Restarting nginx..."
                    docker restart $NGINX

                    sleep 3

                    echo "Validating nginx is running..."
                    if ! docker ps --format '{{.Names}}' | grep -q "^$NGINX$"; then
                        echo "❌ nginx failed to start"
                        docker logs $NGINX
                        exit 1
                    fi
                '''
            }
        }

        stage('Stop Blue') {
            steps {
                sh '''
                    echo "Stopping BLUE..."

                    docker stop $BLUE || true
                    docker rm $BLUE || true
                '''
            }
        }
    }

    post {
        failure {
            echo "🚨 Deployment failed. Rolling back to BLUE..."

            sh '''
                sed -i 's/tic-green/tic-blue/g' nginx/nginx.conf

                docker restart $NGINX || true
            '''
        }
    }
}
