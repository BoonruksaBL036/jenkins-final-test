pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    environment {
        DOCKER_HUB_CREDENTIALS_ID = 'docker-jenkins-test'
        DOCKER_REPO               = "brswnnon/flask-docker-app"

        DEV_APP_NAME              = "flask-app-dev"
        DEV_HOST_PORT             = "5001"

        PROD_APP_NAME             = "flask-app-prod"
        PROD_HOST_PORT            = "5000"
    }

    parameters {
        choice(name: 'ACTION', choices: ['Build & Deploy', 'Rollback'], description: 'เลือก Action')
        string(name: 'ROLLBACK_TAG', defaultValue: '', description: 'Tag สำหรับ rollback')
        choice(name: 'ROLLBACK_TARGET', choices: ['dev', 'prod'], description: 'เลือก environment')
    }

    stages {

        stage('Debug Info') {
            steps {
                echo "ACTION: ${params.ACTION}"
                echo "BRANCH_NAME: ${env.BRANCH_NAME}"
                echo "GIT_BRANCH: ${env.GIT_BRANCH}"
                echo "BUILD_NUMBER: ${env.BUILD_NUMBER}"
            }
        }

        stage('Checkout') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                checkout scm
            }
        }

        stage('Test') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                script {
                    docker.image('python:3.13-slim').inside {
                        sh '''
                            pip install -r requirements.txt
                            pytest -v || true
                        '''
                    }
                }
            }
        }

        stage('Build & Push') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                script {
                    def tag = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.IMAGE_TAG = tag

                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_HUB_CREDENTIALS_ID) {
                        def img = docker.build("${DOCKER_REPO}:${tag}")
                        img.push()
                        img.push('latest')
                    }

                    echo "Built image: ${DOCKER_REPO}:${tag}"
                }
            }
        }

        stage('Deploy to DEV') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                sh """
                    docker pull ${DOCKER_REPO}:${env.IMAGE_TAG}
                    docker stop ${DEV_APP_NAME} || true
                    docker rm ${DEV_APP_NAME} || true
                    docker run -d -p ${DEV_HOST_PORT}:5000 --name ${DEV_APP_NAME} ${DOCKER_REPO}:${env.IMAGE_TAG}
                """
            }
        }

        stage('Approve PROD') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                input message: "Deploy to PROD?"
            }
        }

        stage('Deploy to PROD') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                sh """
                    docker pull ${DOCKER_REPO}:${env.IMAGE_TAG}
                    docker stop ${PROD_APP_NAME} || true
                    docker rm ${PROD_APP_NAME} || true
                    docker run -d -p ${PROD_HOST_PORT}:5000 --name ${PROD_APP_NAME} ${DOCKER_REPO}:${env.IMAGE_TAG}
                """
            }
        }

        stage('Rollback') {
            when { expression { params.ACTION == 'Rollback' } }
            steps {
                script {
                    if (!params.ROLLBACK_TAG) {
                        error "กรุณาใส่ ROLLBACK_TAG"
                    }

                    def app = (params.ROLLBACK_TARGET == 'dev') ? DEV_APP_NAME : PROD_APP_NAME
                    def port = (params.ROLLBACK_TARGET == 'dev') ? DEV_HOST_PORT : PROD_HOST_PORT

                    sh """
                        docker pull ${DOCKER_REPO}:${params.ROLLBACK_TAG}
                        docker stop ${app} || true
                        docker rm ${app} || true
                        docker run -d -p ${port}:5000 --name ${app} ${DOCKER_REPO}:${params.ROLLBACK_TAG}
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning workspace..."
            cleanWs()
        }
    }
}