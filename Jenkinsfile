pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        IMAGE = "tic-app:latest"
        GREEN = "tic-green"
        BLUE = "tic-blue"
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
                //docker run -d -p 5002:5000 --name $GREEN $IMAGE
                '''
            }
        }

        stage('Health Check Green') {
            steps {
                script {
                    try {
                        sh 'sleep 10'
                        sh 'curl -f http://localhost:5002'
                    } catch (err) {
                        error("Green deployment failed")
                    }
                }
            }
        }

        stage('Switch Traffic to Green') {
            steps {
                sh '''
                sed -i 's/tic-blue/tic-green/g' nginx/nginx.conf
                docker exec nginx-proxy nginx -s reload
                '''
            }
        }

        stage('Stop Blue') {
            steps {
                sh '''
                docker stop $BLUE || true
                docker rm $BLUE || true
                '''
            }
        }
    }

    post {
        failure {
            echo "Green failed → Keeping Blue running"
        }
    }
}
