// ============================================================
//  PyGoat - Pipeline DevSecOps (version finale)
//  Politique de blocage :
//   - Gitleaks       : BLOQUANT (tout nouveau secret)
//   - SonarQube QG   : BLOQUANT (nouveau code uniquement)
//   - Semgrep        : rapport seul
//   - Checkov (IaC)  : rapport seul (Dockerfile)
//   - Trivy SCA      : BLOQUANT (CRITICAL avec correctif dispo)
//   - Trivy image    : BLOQUANT (CRITICAL avec correctif dispo)
//   - Syft (SBOM)    : inventaire CycloneDX de l'image
//   - ZAP baseline   : rapport seul (staging)
//   - Slack          : notification en cas de blocage / retour au vert
// ============================================================
pipeline {
    agent any

    environment {
        IMAGE           = 'pygoat'
        TAG             = "${env.BUILD_NUMBER}"
        APP_HOSTS       = 'pygoat-app,localhost,127.0.0.1,192.168.44.128'
        TRIVY_CACHE_DIR = '/var/jenkins_home/.trivy-cache'
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

        // ---------- 1. SECRETS (bloquant) ----------
        stage('Secrets scan (Gitleaks)') {
            steps {
                sh '''
                    docker run --rm --volumes-from jenkins -u $(id -u) \
                        ghcr.io/gitleaks/gitleaks:latest detect \
                        --source "$WORKSPACE" \
                        --gitleaks-ignore-path "$WORKSPACE/.gitleaksignore" \
                        --report-format json \
                        --report-path "$WORKSPACE/gitleaks-report.json" \
                        --redact --exit-code 1
                '''
            }
        }

        // ---------- 2. SAST ----------
        stage('SAST (Semgrep)') {
            steps {
                sh '''
                    docker run --rm -u $(id -u) -e HOME=/tmp \
                        --mount type=volume,src=infra_jenkins_home,dst=/src,volume-subpath=workspace/$JOB_NAME \
                        -w /src semgrep/semgrep:latest \
                        semgrep scan --config p/python --config p/django \
                        --metrics=off --json --output /src/semgrep-report.json
                '''
            }
        }

        // set +x : le token SonarQube n'apparait jamais dans les logs
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
                            -Dsonar.exclusions=**/migrations/**,**/static/**,**/*.min.js,*-report.*,.sonar/**,.scannerwork/** \
                            -Dsonar.python.version=3.11 \
                            -Dsonar.working.directory="$WORKSPACE/.scannerwork"
                    '''
                }
            }
        }

        stage('Quality Gate (SonarQube)') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ---------- 3. IaC (rapport seul) ----------
        stage('IaC scan (Checkov)') {
            steps {
                sh '''
                    docker run --rm --volumes-from jenkins -u $(id -u) -e HOME=/tmp \
                        bridgecrew/checkov:latest \
                        -f "$WORKSPACE/Dockerfile" --framework dockerfile \
                        -o json --soft-fail > "$WORKSPACE/checkov-report.json"
                '''
            }
        }

        // ---------- 4. SCA - dependances (bloquant sur CRITICAL corrigeable) ----------
        stage('SCA (Trivy dependencies)') {
            steps {
                sh '''
                    docker run --rm --volumes-from jenkins -u $(id -u) \
                        -e TRIVY_CACHE_DIR aquasec/trivy:latest fs \
                        --scanners vuln --severity HIGH,CRITICAL \
                        --format json -o "$WORKSPACE/trivy-fs-report.json" "$WORKSPACE"
                    docker run --rm --volumes-from jenkins -u $(id -u) \
                        -e TRIVY_CACHE_DIR aquasec/trivy:latest fs \
                        --scanners vuln --severity HIGH,CRITICAL \
                        --format template --template "@/contrib/html.tpl" \
                        -o "$WORKSPACE/trivy-fs-report.html" "$WORKSPACE"
                    docker run --rm --volumes-from jenkins -u $(id -u) \
                        -e TRIVY_CACHE_DIR aquasec/trivy:latest fs \
                        --scanners vuln --severity CRITICAL --ignore-unfixed \
                        --ignorefile "$WORKSPACE/.trivyignore" \
                        --skip-dirs "$WORKSPACE/dockerized_labs" --exit-code 1 "$WORKSPACE"
                '''
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build -t $IMAGE:$TAG .'
            }
        }

        // ---------- 5. SCAN D'IMAGE (bloquant sur CRITICAL corrigeable) ----------
        stage('Image scan (Trivy)') {
            steps {
                sh '''
                    SOCK_GID=$(stat -c %g /var/run/docker.sock)
                    docker run --rm --volumes-from jenkins -u $(id -u) --group-add $SOCK_GID \
                        -e TRIVY_CACHE_DIR aquasec/trivy:latest image \
                        --scanners vuln --severity HIGH,CRITICAL \
                        --format template --template "@/contrib/html.tpl" \
                        -o "$WORKSPACE/trivy-image-report.html" $IMAGE:$TAG
                    docker run --rm --volumes-from jenkins -u $(id -u) --group-add $SOCK_GID \
                        -e TRIVY_CACHE_DIR aquasec/trivy:latest image \
                        --scanners vuln --severity CRITICAL --ignore-unfixed \
                        --ignorefile "$WORKSPACE/.trivyignore" \
                        --exit-code 1 $IMAGE:$TAG
                '''
            }
        }

        // ---------- 6. SBOM (inventaire CycloneDX) ----------
        stage('SBOM (Syft)') {
            steps {
                sh '''
                    SOCK_GID=$(stat -c %g /var/run/docker.sock)
                    docker run --rm --volumes-from jenkins -u $(id -u) --group-add $SOCK_GID \
                        anchore/syft:latest $IMAGE:$TAG \
                        -o cyclonedx-json="$WORKSPACE/sbom-report.json"
                '''
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

        // ---------- 7. DAST (rapport seul, en staging) ----------
        stage('DAST (OWASP ZAP baseline)') {
            steps {
                sh '''
                    docker run --rm --network infra_default \
                        -v infra_jenkins_home:/zap/wrk:rw \
                        ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
                        -t http://pygoat-app:8000 \
                        -r "workspace/$JOB_NAME/zap-report.html" \
                        -J "workspace/$JOB_NAME/zap-report.json" \
                        -I
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '*-report.json, *-report.html', allowEmptyArchive: true
            publishHTML(target: [allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true,
                reportDir: '.', reportFiles: 'trivy-fs-report.html', reportName: 'Trivy - Dependances'])
            publishHTML(target: [allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true,
                reportDir: '.', reportFiles: 'trivy-image-report.html', reportName: 'Trivy - Image'])
            publishHTML(target: [allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true,
                reportDir: '.', reportFiles: 'zap-report.html', reportName: 'OWASP ZAP'])
        }
        // ---------- ALERTING : Slack (webhook stocke en credential Jenkins) ----------
        failure {
            withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_URL')]) {
                sh '''
                    set +x
                    curl -s -X POST -H 'Content-type: application/json' \
                        --data '{"text":":x: Pipeline BLOQUE : '"$JOB_NAME"' #'"$BUILD_NUMBER"' - '"$BUILD_URL"'"}' \
                        "$SLACK_URL"
                '''
            }
        }
        fixed {
            withCredentials([string(credentialsId: 'slack-webhook', variable: 'SLACK_URL')]) {
                sh '''
                    set +x
                    curl -s -X POST -H 'Content-type: application/json' \
                        --data '{"text":":white_check_mark: Pipeline de nouveau au vert : '"$JOB_NAME"' #'"$BUILD_NUMBER"'"}' \
                        "$SLACK_URL"
                '''
            }
        }
    }
}
