// ============================================================
//  PyGoat - Pipeline DevSecOps
//  v3 : + SAST (Semgrep + SonarQube) - mode observation
// ============================================================
pipeline {
    agent any

    environment {
        IMAGE     = 'pygoat'
        TAG       = "${env.BUILD_NUMBER}"
        APP_HOSTS = 'pygoat-app,localhost,127.0.0.1,192.168.44.128'
    }

    triggers {
        pollSCM('H/2 * * * *')
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

        // --- SÉCURITÉ : secrets (BLOQUANT, exemptions dans .gitleaksignore)
        stage('Secrets scan (Gitleaks)') {
            steps {
                sh '''
                    docker run --rm --volumes-from jenkins -u $(id -u) \
                        ghcr.io/gitleaks/gitleaks:latest detect \
                        --source "$WORKSPACE" \
                        --gitleaks-ignore-path "$WORKSPACE/.gitleaksignore" \
                        --report-format json \
                        --report-path "$WORKSPACE/gitleaks-report.json" \
                        --redact \
                        --exit-code 1
                '''
            }
        }

        // --- SÉCURITÉ : SAST Semgrep (règles Python + Django) - observation
        stage('SAST (Semgrep)') {
            steps {
                sh '''
                    docker run --rm --volumes-from jenkins -u $(id -u) \
                        -e HOME=/tmp -w "$WORKSPACE" \
                        semgrep/semgrep:latest \
                        semgrep scan --config p/python --config p/django \
                        --metrics=off --json --output semgrep-report.json
                '''
            }
        }

        // --- SÉCURITÉ : SAST SonarQube (qualité + sécurité du code)
        // set +x : empêche le token d'apparaître dans les logs Jenkins
        stage('SAST (SonarQube)') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        set +x
                        export SONAR_TOKEN="$SONAR_AUTH_TOKEN"
                        docker run --rm --volumes-from jenkins -u $(id -u) \
                            --network infra_default \
                            -e SONAR_HOST_URL -e SONAR_TOKEN \
                            -e SONAR_USER_HOME="$WORKSPACE/.sonar" \
                            -w "$WORKSPACE" \
                            sonarsource/sonar-scanner-cli:latest \
                            -Dsonar.projectKey=pygoat \
                            -Dsonar.projectName=PyGoat \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=**/migrations/**,**/static/**,**/*.min.js \
                            -Dsonar.python.version=3.11
                    '''
                }
            }
        }

        // --- Quality Gate SonarQube - observation (n'arrête pas encore le build)
        stage('Quality Gate (SonarQube)') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
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
