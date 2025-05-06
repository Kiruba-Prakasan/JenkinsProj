pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_IMAGE_NAME = 'kirubarp/jenkinsrepo'
        DOCKER_IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
        KUBERNETES_NAMESPACE = 'default'
        APP_NAME = 'react-frontend'
        HELM_CHART_PATH = './helm-chart'
        GIT_REPO_URL = 'https://github.com/Kiruba-Prakasan/JenkinsProj.git'
        GIT_BRANCH = 'main'
    }

    stages {
        stage('Setup Environment') {
            steps {
                script {
                    sh 'nvm install 16'
                    sh 'node --version'
                    sh 'npm install -g npm@latest'
                }
            }
        }

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
                sh 'npm audit --audit-level=critical || true' // Warn but don't fail
            }
        }

        stage('Run Tests') {
            steps {
                sh 'npm test -- --watchAll=false --passWithNoTests --coverage'
                sh 'cat coverage/lcov.info | npx coveralls || true' // Optional coverage reporting
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Build Docker Image') {
            environment {
                DOCKER_BUILDKIT = "1"
            }
            steps {
                script {
                    sh "docker build --progress=plain -t ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} ."
                    sh "docker tag ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} ${DOCKER_IMAGE_NAME}:latest"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh "echo ${DOCKER_PASS} | docker login ${DOCKER_REGISTRY} -u ${DOCKER_USER} --password-stdin"
                        sh "docker push ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                        sh "docker push ${DOCKER_IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes with Helm') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        sh """
                        sed -i 's|repository:.*|repository: ${DOCKER_IMAGE_NAME}|g' ${HELM_CHART_PATH}/values.yaml
                        sed -i 's|tag:.*|tag: \"${DOCKER_IMAGE_TAG}\"|g' ${HELM_CHART_PATH}/values.yaml
                        helm upgrade --install ${APP_NAME} ${HELM_CHART_PATH} \
                            --namespace ${KUBERNETES_NAMESPACE} \
                            --create-namespace \
                            --wait \
                            --timeout 5m
                        kubectl rollout status deployment/${APP_NAME} -n ${KUBERNETES_NAMESPACE} --timeout=5m
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'React app deployment successful!'
            slackSend(color: 'good', message: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
        }
        failure {
            echo 'React app deployment failed!'
            slackSend(color: 'danger', message: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
            script {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh "helm rollback ${APP_NAME} -n ${KUBERNETES_NAMESPACE} || true"
                }
            }
        }
        always {
            sh "docker rmi ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} || true"
            cleanWs()
        }
    }
}
