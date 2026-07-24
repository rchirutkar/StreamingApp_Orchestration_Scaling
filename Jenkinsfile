pipeline {
    agent { label 'Ranjeet_node' }

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_CREDENTIALS = 'aws-prod'
    }

    options {
        timestamps()
        //timeout(time: 20, unit: 'MINUTES') // Fails the build if it takes longer than  20 mins
        //wrap([$class: 'AnsiColorBuildWrapper', colorMapName: 'Xterm'])
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        stage('Clean Workspace') {
            steps {
                echo 'Cleaning Workspace...'
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Verify Environment') {
            steps {
                sh '''
                    echo "===== Environment ====="
                    hostname
                    pwd
                    whoami
                    docker --version
                    docker compose version
                    aws --version
                '''
            }
        }

        stage('Build Images') {
            steps {
                sh '''
                    docker compose build
                '''
            }
        }

        stage('Login to Amazon ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-prod']]) {
                    sh '''
                        ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

                        aws ecr get-login-password --region $AWS_REGION | \
                        docker login \
                        --username AWS \
                        --password-stdin \
                        ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                        echo "ECR Login Successful"
                    '''
                }
            }
        }

        stage('Tag Images') {
            steps {
                sh '''
                    ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

                    for IMAGE in auth admin chat streaming frontend
                    do
                        docker tag streamingapp-$IMAGE:latest \
                        ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/streamingapp-$IMAGE:${BUILD_NUMBER}
                    done
                '''
            }
        }

        stage('Push Images') {
            steps {
                sh '''
                    ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

                    for IMAGE in auth admin chat streaming frontend
                    do
                        docker push \
                        ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/streamingapp-$IMAGE:${BUILD_NUMBER}
                    done
                '''
            }
        }
    }

    post {

        success {
            echo "Images pushed successfully."
        }

        failure {
            echo "Pipeline failed."
        }

        always {
            sh 'docker image prune -f || true'
        }
    }
}
