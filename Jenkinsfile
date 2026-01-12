pipeline {
    agent none

    environment {
        IMAGE_NAME   = "eco12345/jenkins-dsl"
        SERVICE_NAME = "jenkins-dsl"
        DEV_SERVER   = "dev@192.168.92.131"
        APP_PORT     = "3000"
    }

    stages {

        stage('Checkout') {
            agent any
            steps {
                checkout scm
                sh 'git fetch --all --tags'
                echo "BRANCH_NAME=${env.BRANCH_NAME}"
            }
        }

        stage('Verify tag from dev') {
            agent any
            when {
                buildingTag()
            }
            steps {
                sh '''
                git fetch origin dev
                git merge-base --is-ancestor HEAD origin/dev \
                  || (echo "❌ Tag not created from dev" && exit 1)
                '''
            }
        }

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
                    docker login -u $DOCKER_USER -p $DOCKER_PASS
                    docker build -t $IMAGE_NAME:$BRANCH_NAME .
                    docker push $IMAGE_NAME:$BRANCH_NAME
                    '''
                }
            }
        }

        stage('Deploy DEV') {
            agent any
            when {
                buildingTag()
            }
            steps {
                sshagent(['jenkins-agent-01']) {
                    sh """
                    ssh ${DEV_SERVER} '
                      docker pull $IMAGE_NAME:$BRANCH_NAME
                      docker stop $SERVICE_NAME || true
                      docker rm $SERVICE_NAME || true
                      docker run -d --name $SERVICE_NAME \
                        -p $APP_PORT:3000 \
                        $IMAGE_NAME:$BRANCH_NAME
                    '
                    """
                }
            }
        }
    }
}
