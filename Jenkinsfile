pipeline {
    agent { node { label ''; customWorkspace '/workspace/devsecops-lab' } }
    options { skipDefaultCheckout(true); disableConcurrentBuilds() }
    environment { 
        APP_IMAGE = "classroom-app:${BUILD_NUMBER}" 
        APP_CONTAINER = "classroom-app-${BUILD_NUMBER}"
        LAB_NETWORK = "classroom-net-${BUILD_NUMBER}"    
    }

    triggers {
        pollSCM('H/2 * * * *')
        // TODO: investigar e configurar a alternativa com webhook.
        // githubPush()
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git rev-parse HEAD'
            }
        }
        stage('Verify Docker') {
            steps { sh 'docker version' }
        }
        stage('SCA - Trivy') {
            steps {
                sh '''
                    docker run --rm -v "$WORKSPACE:/src" -w /src \
                    aquasec/trivy:0.58.2 fs --scanners vuln --exit-code 0 . 
                    '''
            }
        }
        stage('SAST - Bandit') {
            steps {
                sh 'docker build -f ci/Dockerfile.bandit -t classroombandit:local .'
                
                sh '''
                    docker run --rm -v "$WORKSPACE:/src" \
                    -w /src \
                    classroombandit:local -r app --exit-zero
                    '''
            }
        }

        stage('Build application image') {
            steps { 
                sh 'docker build -t "$APP_IMAGE" .'
            }
        }

        stage('Image scan - Trivy') {
            steps {
                sh 'docker save "$APP_IMAGE" -o app-image.tar'
                sh '''
                    docker run --rm -v "$WORKSPACE:/src" -w /src \
                    aquasec/trivy:0.58.2 image --input app-image.tar \
                    --scanners vuln --exit-code 0
                    '''
            }

            post {
                always { sh 'rm -f app-image.tar' }
            }

        } 

        stage('Start application') {
            steps {
                    sh '''
                        docker network create "$LAB_NETWORK"
                        docker run -d --name "$APP_CONTAINER" --network "$LAB_NETWORK" "$APP_IMAGE"
                        ready=0
                        for attempt in $(seq 1 30); do
                        if docker exec "$APP_CONTAINER" python -c "import urllib.request;\
                            urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2)"; then
                            ready=1
                            break
                        fi
                        sleep 2
                        done
                        if [ "$ready" -ne 1 ]; then
                        docker logs "$APP_CONTAINER"
                        exit 1
                        fi
                    '''
            }
            post {
                always {
                    sh 'docker logs "$APP_CONTAINER" 2>/dev/null || true'
                    sh 'docker rm -f "$APP_CONTAINER" 2>/dev/null || true'
                    sh 'docker network rm "$LAB_NETWORK" 2>/dev/null || true'
                    sh 'docker image rm "$APP_IMAGE" 2>/dev/null || true'
                }
            }
        }

        
    }
    post {
        always {
            sh 'docker image rm "$APP_IMAGE" 2>/dev/null || true'
            
        }
    }
}
