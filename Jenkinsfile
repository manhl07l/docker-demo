pipeline {
    agent any

    environment {
        IMAGE_NAME = "eco12345/jenkins-dsl"
        SERVICE_NAME = "jenkins-dsl"
        DEV_SERVER = "dev@192.168.92.131"
        APP_PORT   = "3000"
        SONAR_HOST = "http://192.168.92.130:9000"
    }

    stages {

        stage('Checkout') {
            agent any
            steps {
                checkout scm
                sh 'git fetch --tags'
                sh 'git describe --tags --exact-match || true'
                sh 'echo "GIT_BRANCH=${GIT_BRANCH}"'
            }
        }

            /* =========================
            VERIFY TAG
            Chỉ chạy khi có TAG
            ========================== */
        stage('Verify tag') {
            agent any
            when {
                expression {
                env.GIT_BRANCH?.startsWith("refs/tags/")
                }
            }
            steps {
                sh '''
                echo "Verifying tag belongs to dev..."
                git fetch origin dev
                git branch --contains HEAD | grep dev
                '''
            }
        }

            /* =========================
            BUILD
            ========================== */
        stage('Build') {
            agent any
            when {
                anyOf {
                branch 'dev'
                branch 'stg'
                branch 'master'
                expression { env.GIT_BRANCH?.startsWith("refs/tags/") }
                }
            }
            steps {
                sh './gradlew clean build -x test'
            }
        }

            /* =========================
            TEST
            ========================== */
        stage('Test') {
            agent any
            steps {
                sh './gradlew test'
            }
        }

            /* =========================
            SONARQUBE
            ========================== */
        stage('SonarQube') {
            agent any
            environment {
                SONAR_SCANNER = tool 'sonar'
            }
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                sh """
                    ${SONAR_SCANNER}/bin/sonar-scanner \
                    -Dsonar.projectKey=myapp \
                    -Dsonar.sources=src \
                    -Dsonar.host.url=${SONAR_HOST} \
                    -Dsonar.login=${SONAR_TOKEN}
                """
                }
            }
        }

        stage('Build & Push Image') {
            agent any
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
