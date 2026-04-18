pipeline {
    agent any

    environment {
        DOCKERHUB_CREDS = credentials('dockerhub-creds')
        DEV_REPO        = 'antben1204/devops-build-dev'
        PROD_REPO       = 'antben1204/devops-build-prod'
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
        APP_HOST        = '3.7.66.199'
        APP_USER        = 'ubuntu'
        CONTAINER_NAME  = 'devops-build-app'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.SHORT_COMMIT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                }
                sh 'echo Branch: ${BRANCH_NAME}'
                sh 'echo Commit: ${GIT_COMMIT}'
            }
        }

        stage('Build Image') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'dev') {
                        env.TARGET_REPO = env.DEV_REPO
                    } else if (env.BRANCH_NAME == 'prod') {
                        env.TARGET_REPO = env.PROD_REPO
                    } else {
                        currentBuild.result = 'NOT_BUILT'
                        error("Branch ${env.BRANCH_NAME} is not configured for image push")
                    }
                }

                sh '''
                    docker build -t ${TARGET_REPO}:${IMAGE_TAG} .
                    docker tag ${TARGET_REPO}:${IMAGE_TAG} ${TARGET_REPO}:latest
                    docker tag ${TARGET_REPO}:${IMAGE_TAG} ${TARGET_REPO}:${SHORT_COMMIT}
                '''
            }
        }

        stage('Docker Login') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'prod'
                }
            }
            steps {
                sh '''
                    echo "$DOCKERHUB_CREDS_PSW" | docker login -u "$DOCKERHUB_CREDS_USR" --password-stdin
                '''
            }
        }

        stage('Push Image') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'prod'
                }
            }
            steps {
                sh '''
                    docker push ${TARGET_REPO}:${IMAGE_TAG}
                    docker push ${TARGET_REPO}:latest
                    docker push ${TARGET_REPO}:${SHORT_COMMIT}
                '''
            }
        }

        stage('Deploy to App EC2') {
            when {
                branch 'prod'
            }
            steps {
                sshagent(credentials: ['app-server-ssh']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ${APP_USER}@${APP_HOST} "
                            docker login -u ${DOCKERHUB_CREDS_USR} -p ${DOCKERHUB_CREDS_PSW} &&
                            docker pull ${PROD_REPO}:latest &&
                            docker stop ${CONTAINER_NAME} || true &&
                            docker rm ${CONTAINER_NAME} || true &&
                            docker run -d --name ${CONTAINER_NAME} -p 80:80 ${PROD_REPO}:latest
                        "
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            sh 'docker image prune -f || true'
        }
        success {
            echo "Build, push, and optional deploy completed for branch ${BRANCH_NAME}"
        }
        failure {
            echo "Pipeline failed on branch ${BRANCH_NAME}"
        }
    }
}
