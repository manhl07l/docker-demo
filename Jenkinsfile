pipeline {
    agent none

    environment {
        IMAGE_NAME   = "eco12345/jenkins-dsl"
        SERVICE_NAME = "jenkins-dsl"
        DEV_SERVER   = "dev@192.168.92.131"
        APP_PORT     = "3000"
        SONAR_HOST   = "http://192.168.92.130:9000"
    }

    stages {

        /* =========================
           INFO / DEBUG
        ========================== */
        stage('Info') {
            agent any
            steps {
                echo "BRANCH_NAME = ${env.BRANCH_NAME}"
                echo "TAG_NAME    = ${env.TAG_NAME}"
            }
        }

        /* =========================
           CHECKOUT
        ========================== */
        stage('Checkout') {
            agent any
            steps {
                checkout scm
            }
        }

        /* =========================
           VERIFY TAG (chỉ cho TAG)
        ========================== */
        stage('Verify tag') {
            agent any
            when {
                buildingTag()
            }
            steps {
                sh '''
                echo "Verifying tag belongs to dev branch..."

                git fetch origin dev

                # HEAD (tag commit) phải thuộc dev
                git branch -r --contains HEAD | grep origin/dev
                '''
            }
        }

        /* =========================
           SONARQUBE (chỉ cho TAG)
        ========================== */
        // stage('SonarQube') {
        //     agent any
        //     when {
        //         buildingTag()
        //     }
        //     environment {
        //         SONAR_SCANNER = tool 'sonar'
        //     }
        //     steps {
        //         withCredentials([
        //             string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')
        //         ]) {
        //             sh """
        //             ${SONAR_SCANNER}/bin/sonar-scanner \
        //               -Dsonar.projectKey=jenkins-dsl \
        //               -Dsonar.sources=. \
        //               -Dsonar.host.url=${SONAR_HOST} \
        //               -Dsonar.login=${SONAR_TOKEN}
        //             """
        //         }
        //     }
        // }

        /* =========================
           BUILD & PUSH IMAGE
        ========================== */
        stage('Build & Push Image') {
            agent any
            when {
                buildingTag()
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                    docker build -t ${IMAGE_NAME}:${TAG_NAME} .
                    docker push ${IMAGE_NAME}:${TAG_NAME}

                    docker logout
                    '''
                }
            }
        }

        /* =========================
           DEPLOY DEV
        ========================== */
        stage('Deploy DEV') {
            agent any
            when {
                buildingTag()
            }
            steps {
                sshagent(['jenkins-agent-01']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ${DEV_SERVER} '
                        docker pull ${IMAGE_NAME}:${TAG_NAME}
                        docker stop ${SERVICE_NAME} || true
                        docker rm ${SERVICE_NAME} || true
                        docker run -d \
                          --name ${SERVICE_NAME} \
                          -p ${APP_PORT}:3000 \
                          ${IMAGE_NAME}:${TAG_NAME}
                    '
                    """
                }
            }
        }
    }
}
