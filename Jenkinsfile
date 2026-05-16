pipeline {

    agent any

    environment {

        DOCKER_IMAGE = "sagarsalve49/myapp"
        DOCKER_TAG = "${BUILD_NUMBER}"

    }

    triggers {
        githubPush()
    }

    stages {

        stage('Clone Code') {

            steps {

                git branch: 'main',
                url: 'https://github.com/Sagar-salve-49/FinacPlus-Task.git'
            }
        }

        stage('Build Docker Image') {

            steps {

                sh 'docker build -t $DOCKER_IMAGE:$DOCKER_TAG .'

                sh 'docker tag $DOCKER_IMAGE:$DOCKER_TAG $DOCKER_IMAGE:latest'
            }
        }

        stage('Push Docker Image') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

                    docker push $DOCKER_IMAGE:$DOCKER_TAG

                    docker push $DOCKER_IMAGE:latest
                    '''
                }
            }
        }

        stage('Deploy to EKS') {

            steps {

                sh '''
                aws eks update-kubeconfig \
                --region eu-north-1 \
                --name devops-cluster

                kubectl apply -f deployment.yaml

                kubectl apply -f service.yaml
                '''
            }
        }

        stage('Verify Deployment') {

            steps {

                sh '''
                kubectl get pods

                kubectl get svc
                '''
            }
        }
    }

    post {

        success {

            echo 'Pipeline Executed Successfully'
        }

        failure {

            echo 'Pipeline Failed'
        }
    }
}
