pipeline {
    agent any

    environment {
        DOCKER_HUB_CREDS = credentials('docker-credentials')

        IMAGE_NAME   = "${DOCKER_HUB_CREDS_USR}/task-manager-app"
        IMAGE_TAG    = "${IMAGE_NAME}:${BUILD_NUMBER}"
        IMAGE_LATEST = "${IMAGE_NAME}:latest"

        K8S_DEPLOYMENT = "taskmanager-app"
        K8S_CONTAINER  = "taskmanager-app"

        DEV_NS   = "taskmanager-dev"
        STAGE_NS = "taskmanager-staging"
        PROD_NS  = "taskmanager-production"
    }

    stages {

//CHECKOUT STAGE 
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

// INSTALL STAGE 
        stage('Install') {
            steps {
                sh 'npm install'
            }
        }

// BUILD IMAGE STAGE
        stage('Build Docker Image') {
            steps {
                sh '''
                chmod 666 /var/run/docker.sock || true

                docker build -t ${IMAGE_TAG} -t ${IMAGE_LATEST} .
                '''
            }
        }

//PUSH IMAGE STAGE
        stage('Push Image') {
            steps {
                sh '''
                echo ${DOCKER_HUB_CREDS_PSW} | docker login -u ${DOCKER_HUB_CREDS_USR} --password-stdin

                docker push ${IMAGE_TAG}
                docker push ${IMAGE_LATEST}

                docker logout
                '''
            }
        }

//DEV STAGE
        stage('Deploy DEV') {
            steps {
                sh '''
                export KUBECONFIG=/var/jenkins_home/.kube/config

                kubectl apply -f k8s/dev/

                kubectl set image deployment/${K8S_DEPLOYMENT} \
                ${K8S_CONTAINER}=${IMAGE_TAG} -n ${DEV_NS}

                kubectl rollout status deployment/${K8S_DEPLOYMENT} -n ${DEV_NS}

                kubectl get pods -n ${DEV_NS}
                '''
            }
        }

//STAGING STAGE
        stage('Deploy STAGING') {
            steps {
                sh '''
                export KUBECONFIG=/var/jenkins_home/.kube/config

                kubectl apply -f k8s/staging/

                kubectl set image deployment/${K8S_DEPLOYMENT} \
                ${K8S_CONTAINER}=${IMAGE_TAG} -n ${STAGE_NS}

                kubectl rollout status deployment/${K8S_DEPLOYMENT} -n ${STAGE_NS}
                '''
            }
        }

//APPROVAL STAGE
        stage('Approval') {
            steps {
                input message: 'Deploy to PRODUCTION?'
            }
        }

//PRODUCTION STAGE
        stage('Deploy PROD') {
            steps {

                withCredentials([
                    file(credentialsId: 'kubeconfig-ec2', variable: 'KUBECONFIG')
                ]) {

                    sh '''
                    kubectl apply --validate=false -f k8s/production/
                    '''
                }
            }
        }
    }

//POST STAGE
    post {
        always {
            echo 'Pipeline Finished'
        }

        success {
            echo 'Deployment Successful'
        }

        failure {
            echo 'Deployment Failed'
        }
    }
}
