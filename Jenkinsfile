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

        stage('Checkout') {
            agent any
            steps {
                checkout scm
                sh 'git fetch --tags'
                script {
                    if (env.GIT_BRANCH?.startsWith("refs/tags/")) {
                        env.TAG_NAME = env.GIT_BRANCH.replace("refs/tags/", "")
                        echo "TAG_NAME=${env.TAG_NAME}"
                    } else {
                        error "This build is not triggered by a tag"
                    }
                }
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
                echo "Verifying tag belongs to dev...v1"
                git fetch origin dev
                git branch --contains HEAD | grep dev
                '''
            }
        }
            /* =========================
            SONARQUBE
            ========================== */
        // stage('SonarQube') {
        //     agent any
        //     environment {
        //         SONAR_SCANNER = tool 'sonar'
        //     }
        //     steps {
        //         withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
        //         sh """
        //             ${SONAR_SCANNER}/bin/sonar-scanner \
        //             -Dsonar.projectKey=gs-gradle \
        //             -Dsonar.sources=src \
        //             -Dsonar.host.url=${SONAR_HOST} \
        //             -Dsonar.login=${SONAR_TOKEN}
        //         """
        //         }
        //     }
        // }

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
                    docker build -t ${IMAGE_NAME}:$TAG_NAME .
                    docker push ${IMAGE_NAME}:$TAG_NAME
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
                        docker pull ${IMAGE_NAME}:$TAG_NAME
                        docker stop ${SERVICE_NAME} || true
                        docker rm ${SERVICE_NAME} || true
                        docker run -d --name ${SERVICE_NAME} -p ${APP_PORT}:3000 \
                          ${IMAGE_NAME}:$TAG_NAME
                    '
                    """
                }
            }
        }
    }
}
