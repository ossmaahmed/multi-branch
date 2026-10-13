pipeline {
    agent any

    environment {
        APP_NAME = 'app-hello-nti'
        REPO_URL = 'https://github.com/ossmaahmed/python-jenkins'
    }

    stages {
        stage('Getting Repo files') {
            steps {
                git branch: 'main', credentialsId: 'jenkins-token', url: "${REPO_URL}"
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${APP_NAME}:${BUILD_NUMBER} .'
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                        credentialsId: 'docker',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                        docker tag ${APP_NAME}:${BUILD_NUMBER} $DOCKER_USERNAME/${APP_NAME}:${BUILD_NUMBER}
                        docker push $DOCKER_USERNAME/${APP_NAME}:${BUILD_NUMBER}
                        docker logout
                    '''
                }
            }
        }

        stage('Run Docker Container') {
            steps {
                sh '''
                    docker rm -f ${APP_NAME} || true
                    docker run -d -p 8082:8082 --name ${APP_NAME} ${APP_NAME}:${BUILD_NUMBER}
                    docker ps
                '''
            }
        }
    }
}
