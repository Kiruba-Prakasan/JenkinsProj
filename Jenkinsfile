pipeline {
    agent any

    environment {
        // Define environment variables
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_IMAGE_NAME = 'kirubarp/jenkinsrepo'  // Updated Docker image name
        DOCKER_IMAGE_TAG = "${env.BUILD_NUMBER}"
        KUBERNETES_NAMESPACE = 'default'
        APP_NAME = 'react-frontend'
        HELM_CHART_PATH = './helm-chart'

        // Git repository information
        GIT_REPO_URL = 'https://github.com/Kiruba-Prakasan/JenkinsProj.git'
        GIT_BRANCH = 'main'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: "${GIT_BRANCH}", 
                    credentialsId: 'github_seccred',
                    url: "${GIT_REPO_URL}"
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'npm test -- --watchAll=false --passWithNoTests'
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} ."
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'docker', variable: 'DOCKER_HUB_TOKEN')]) {
                        sh "echo ${DOCKER_HUB_TOKEN} | docker login ${DOCKER_REGISTRY} -u kirubarp --password-stdin"
                    }
                    sh "docker push ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to Kubernetes with Helm') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        sh "sed -i 's|tag:.*|tag: \"${DOCKER_IMAGE_TAG}\"|g' ${HELM_CHART_PATH}/values.yaml"
                        sh "helm upgrade --install ${APP_NAME} ${HELM_CHART_PATH} --namespace ${KUBERNETES_NAMESPACE} --create-namespace"
                        sh "kubectl rollout status deployment/${APP_NAME} -n ${KUBERNETES_NAMESPACE}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'React app deployment successful!'
        }
        failure {
            echo 'React app deployment failed!'
        }
        always {
            sh "docker rmi ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} || true"
        }
    }
}


this is my cript
