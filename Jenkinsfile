pipeline {
    agent none

    environment {
        IMAGE_NAME   = "eco12345/jenkins-dsl"
        SERVICE_NAME = "jenkins-dsl"
        DEV_SERVER   = "agent@192.168.92.131"
        APP_PORT     = "3000"
    }

    stages {

        /* =========================
           INFO
        ========================== */
        stage('Info') {
            agent any
            steps {
                echo "BRANCH_NAME = ${env.BRANCH_NAME}"
                echo "TAG_NAME    = ${env.TAG_NAME}"
            }
        }

        stage('Guard – tag only') {
            agent any
            when {
                not { buildingTag() }
            }
            steps {
                echo "ℹ️ Branch build detected (${env.BRANCH_NAME})"
                echo "ℹ️ This pipeline is TAG-based CD only → skip"

                script {
                    currentBuild.result = 'NOT_BUILT'
                }
            }
        }

        /* =========================
           CHECKOUT
        ========================== */
        stage('Checkout') {
            agent any
            when {
                buildingTag()
            }
            steps {
                checkout scm
            }
        }

        /* =========================
           VERIFY TAG SOURCE
           (chỉ chạy cho TAG)
        ========================== */
        stage('Verify tag from dev') {
            agent any
            when {
                buildingTag()
            }
            steps {
                sh '''
                echo "Verify: tag must be created from dev"

                git fetch origin dev

                git merge-base --is-ancestor HEAD origin/dev \
                  || (echo "❌ Tag NOT created from dev" && exit 1)
                '''
            }
        }

        /* =========================
           BUILD & PUSH DOCKER IMAGE
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
           DEPLOY
        ========================== */
        stage('Deploy DEV') {
            agent any
            when {
                buildingTag()
            }
            steps {
                sshagent(['ssh-key-for-deploy']) {
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

    post {
        success {
            echo "✅ Deploy ${TAG_NAME} SUCCESS"
        }
        failure {
            echo "❌ Pipeline FAILED"
        }
    }
}
