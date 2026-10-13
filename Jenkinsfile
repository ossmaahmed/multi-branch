pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        APP_NAME = 'new-app'
    }

    stages {
        stage('Prepare') {
            steps {
                script {
                    // اسم الـ branch بعد تنظيفه (feature/x -> feature-x) عشان يصلح كاسم docker
                    env.BRANCH_SAFE    = env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9_.-]', '-').toLowerCase()
                    env.CONTAINER_NAME = "${APP_NAME}-${env.BRANCH_SAFE}"
                    env.IMAGE_TAG      = "${env.BRANCH_SAFE}-${env.BUILD_NUMBER}"

                    echo "Branch=${env.BRANCH_NAME} Container=${env.CONTAINER_NAME} Tag=${env.IMAGE_TAG}"
                }
            }
        }

        stage('Getting Repo files') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${APP_NAME}:${IMAGE_TAG} .'
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
                        docker tag ${APP_NAME}:${IMAGE_TAG} $DOCKER_USERNAME/${APP_NAME}:${IMAGE_TAG}
                        docker push $DOCKER_USERNAME/${APP_NAME}:${IMAGE_TAG}
                        docker logout
                    '''
                }
            }
        }

        stage('Run Docker Container') {
            steps {
                sh '''
                    # لو في كونتينر قديم لنفس الـ branch، خُد بورته عشان يفضل ثابت بين الـ builds
                    OLD_PORT=$(docker port ${CONTAINER_NAME} 5000 2>/dev/null | head -n1 | awk -F: '{print $NF}')
                    docker rm -f ${CONTAINER_NAME} || true

                    # أول مرة: Docker بيختار بورت فاضي تلقائياً (-p 5000)
                    # بعد كده: بيستخدم نفس البورت القديم، ولو اتشغل عليه حاجة تانية بيرجع لبورت عشوائي
                    if [ -n "$OLD_PORT" ]; then PORT_MAP="${OLD_PORT}:5000"; else PORT_MAP="5000"; fi
                    docker run -d --name ${CONTAINER_NAME} -p ${PORT_MAP} ${APP_NAME}:${IMAGE_TAG} || {
                        docker rm -f ${CONTAINER_NAME} || true
                        docker run -d --name ${CONTAINER_NAME} -p 5000 ${APP_NAME}:${IMAGE_TAG}
                    }

                    HOST_PORT=$(docker port ${CONTAINER_NAME} 5000 | head -n1 | awk -F: '{print $NF}')
                    echo "=============================================="
                    echo "Branch ${BRANCH_NAME} is running on port: ${HOST_PORT}"
                    echo "=============================================="
                    docker ps --filter "name=${APP_NAME}-"
                '''
            }
        }

        stage('Cleanup old images') {
            steps {
                sh '''
                    # امسح الـ images القديمة بتاعت نفس الـ branch وسيب الأخيرة بس
                    docker images ${APP_NAME} --format '{{.Tag}}' \
                      | grep -E "^${BRANCH_SAFE}-[0-9]+$" \
                      | grep -v "^${IMAGE_TAG}$" \
                      | xargs -r -I{} docker rmi ${APP_NAME}:{} || true
                    docker image prune -f || true
                '''
            }
        }
    }

    post {
        failure {
            sh 'docker rm -f ${CONTAINER_NAME} || true'
        }
    }
}
