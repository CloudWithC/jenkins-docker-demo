
pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'SIMULATE_POST_SWITCH_FAILURE',
            defaultValue: false,
            description: 'Set true only to test automatic rollback'
        )
    }

    environment {
        IMAGE_NAME = 'cibss/jenkins-demo'
        BLUE = 'app-blue'
        GREEN = 'app-green'
        PROXY = 'deploy-proxy'
        NETWORK = 'deploy-net'
        ACTIVE_FILE = '/home/cibss/davine-week1/jenkinsdockerdemo/proxy/default.conf'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                sh '''
                    set -eu

                    test -f index.html
                    grep -q 'Jenkins CI/CD Pipeline Successful' index.html

                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .

                    docker run -d --name ci-test-${BUILD_NUMBER} ${IMAGE_NAME}:${BUILD_NUMBER}

                    sleep 2

                    docker exec ci-test-${BUILD_NUMBER} \
                      wget -qO- http://localhost/ \
                      | grep -q 'Jenkins CI/CD Pipeline Successful'

                    docker rm -f ci-test-${BUILD_NUMBER}
                '''
            }
        }

        stage('Docker Hub Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        set +x

                        echo "$DOCKER_PASS" | docker login \
                          -u "$DOCKER_USER" --password-stdin

                        docker push ${IMAGE_NAME}:${BUILD_NUMBER}

                        docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest
                        docker push ${IMAGE_NAME}:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Prepare Candidate') {
            steps {
                sh '''
                    set -eu

                    docker network inspect ${NETWORK} >/dev/null
                    docker inspect ${PROXY} >/dev/null
                    docker inspect ${BLUE} >/dev/null

                    docker rm -f ${GREEN} 2>/dev/null || true

                    docker run -d \
                      --name ${GREEN} \
                      --network ${NETWORK} \
                      ${IMAGE_NAME}:${BUILD_NUMBER}

                    sleep 2

                    docker exec ${PROXY} wget -qO- http://${GREEN}/ \
                      | grep -q 'Jenkins CI/CD Pipeline Successful'
                '''
            }
        }

        stage('Switch Traffic and Verify') {
            steps {
                sh '''
                    set -eu

                    # Save the currently active route before switching.
                    cp ${ACTIVE_FILE} ${WORKSPACE}/proxy-before-deploy.conf

                    # Route traffic to the candidate version.
                    cat > ${ACTIVE_FILE} <<'CONF'
server {
    listen 80;
    location / {
        proxy_pass http://app-green:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
CONF

                    # Validate and reload the proxy.
                    if docker exec ${PROXY} nginx -t && \
                       docker exec ${PROXY} nginx -s reload; then

                        sleep 2

                        # Optionally simulate a post-switch health-check failure.
                        if [ "${SIMULATE_POST_SWITCH_FAILURE}" = "true" ]; then
                            echo 'Simulating post-deployment health-check failure.'
                            health_ok=false
                        elif curl --fail --silent http://localhost:8087/ \
                          | grep -q 'Jenkins CI/CD Pipeline Successful'; then
                            health_ok=true
                        else
                            health_ok=false
                        fi

                        if [ "$health_ok" = true ]; then
                            echo 'New version deployed and health check passed.'
                        else
                            echo 'Health check failed; restoring previous proxy config.'

                            cp ${WORKSPACE}/proxy-before-deploy.conf ${ACTIVE_FILE}

                            docker exec ${PROXY} nginx -t
                            docker exec ${PROXY} nginx -s reload

                            echo 'Rollback completed. Previous route restored.'
                            exit 1
                        fi
                    else
                        echo 'Proxy validation or reload failed; restoring previous route.'

                        cp ${WORKSPACE}/proxy-before-deploy.conf ${ACTIVE_FILE}

                        docker exec ${PROXY} nginx -t
                        docker exec ${PROXY} nginx -s reload

                        exit 1
                    fi
                '''
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed. Check the stage logs and verify the active proxy version.'
        }

        success {
            echo 'Build, test, image push, deployment, and HTTP verification completed.'
        }
    }
}
