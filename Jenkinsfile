// ============================================================
//  PyGoat - Pipeline "AS-IS" (DevOps sans contrôles de sécurité)
//  Sert de référence avant l'ajout des contrôles DevSecOps
// ============================================================
pipeline {
    agent any

    environment {
        IMAGE = 'pygoat'
        TAG   = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build -t $IMAGE:$TAG .'
            }
        }

        stage('Deploy (staging)') {
            steps {
                sh '''
                    docker rm -f pygoat-app || true
                    docker run -d --name pygoat-app \
                        --network infra_default -e ALLOWED_HOSTS=pygoat-app,localhost,127.0.0.1,192.168.44.128 \
                        -p 8000:8000 $IMAGE:$TAG
                '''
            }
        }

        stage('Smoke test') {
            steps {
                sh '''
                    for i in $(seq 1 30); do
                        curl -fs -o /dev/null http://pygoat-app:8000 && exit 0
                        sleep 2
                    done
                    echo "Application injoignable"
                    exit 1
                '''
            }
        }
    }
}
