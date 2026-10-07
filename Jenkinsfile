pipeline {

    agent any

    environment {
        IMAGE_NAME = "cibss/jenkins-demo"
        IMAGE_TAG = "latest"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Building application...'
                sh 'ls -la'
            }
        }

        stage('Test') {
            steps {
                echo 'Running automated test...'
                sh 'test -f index.html'
                sh 'grep "Jenkins CI/CD Pipeline Successful" index.html'
            }
        }

        stage('Package') {
            steps {
                echo 'Packaging application...'
                sh 'tar -czf package.tar.gz index.html Dockerfile Jenkinsfile'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG .'
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                    sh 'docker push $IMAGE_NAME:$IMAGE_TAG'
                }
            }
        }
    }

    post {
        success {
            echo 'PIPELINE SUCCESSFUL!'
        }

        failure {
            echo 'PIPELINE FAILED!'
        }
    }
}
