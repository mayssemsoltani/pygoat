// ============================================================
//  PyGoat - Pipeline DevSecOps
//  v1 : + scan de secrets (Gitleaks) - mode observation
// ============================================================
pipeline {
    agent any

    environment {
        IMAGE     = 'pygoat'
        TAG       = "${env.BUILD_NUMBER}"
        APP_HOSTS = 'pygoat-app,localhost,127.0.0.1,192.168.44.128'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // --- SÉCURITÉ : détection de secrets dans le code et l'historique Git
        stage('Secrets scan (Gitleaks)') {
            steps {
                sh '''
                    docker run --rm --volumes-from jenkins -u $(id -u) \
                        ghcr.io/gitleaks/gitleaks:latest detect \
                        --source "$WORKSPACE" \
                        --report-format json \
                        --report-path "$WORKSPACE/gitleaks-report.json" \
                        --exit-code 0
                '''
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
                        --network infra_default \
                        -e ALLOWED_HOSTS=$APP_HOSTS \
                        -p 8000:8000 $IMAGE:$TAG
                    docker exec pygoat-app python manage.py migrate --noinput
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

    post {
        always {
            archiveArtifacts artifacts: '*-report.json', allowEmptyArchive: true
        }
    }
}
