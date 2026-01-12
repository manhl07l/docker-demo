pipeline {
    agent none

    environment {
        IMAGE_NAME = "eco12345/jenkins-dsl"
        SERVICE_NAME = "jenkins-dsl"
        DEV_SERVER = "dev@192.168.92.131"
        APP_PORT   = "3000"
        SONAR_HOST = "http://192.168.92.130:9000"
    }

    stages {

        stage('Check branch & tag') {
            agent any
            when {
                allOf {
                    branch 'dev'
                    expression { env.GIT_TAG_NAME }
                }
            }
            steps {
                echo "Triggered by tag: ${GIT_TAG_NAME}"
            }
        }

        stage('Checkout') {
            agent any
            steps {
                checkout scm
            }
        }

        stage('Build & Test & Sonar') {
            agent { label 'docker' }
            steps {
                withCredentials([
                    string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')
                ]) {
                    sh '''
                    ./gradlew clean test build

                    sonar-scanner \
                      -Dsonar.projectKey=gs-gradle \
                      -Dsonar.projectVersion=${GIT_TAG_NAME} \
                      -Dsonar.host.url=${SONAR_HOST} \
                      -Dsonar.login=${SONAR_TOKEN}
                    '''
                }
            }
        }

        stage('Build & Push Image') {
            agent { label 'docker' }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                    docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}
                    docker build -t ${IMAGE_NAME}:${GIT_TAG_NAME} .
                    docker push ${IMAGE_NAME}:${GIT_TAG_NAME}
                    '''
                }
            }
        }

        stage('Deploy DEV') {
            agent any
            steps {
                sshagent(['jenkins-agent-01']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ${DEV_SERVER} '
                        docker pull ${IMAGE_NAME}:${GIT_TAG_NAME}
                        docker stop ${SERVICE_NAME} || true
                        docker rm ${SERVICE_NAME} || true
                        docker run -d --name ${SERVICE_NAME} -p ${APP_PORT}:3000 \
                          ${IMAGE_NAME}:${GIT_TAG_NAME}
                    '
                    """
                }
            }
        }
    }
}
