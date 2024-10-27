pipeline {
    agent any

    environment {
        DOCKER_REPO = 'nicolasgreskiv/pucpr-gh-pages'
        FRONTEND_DIR = 'projeto-web'
        BACKEND_DIR = 'projeto-spring'
        K8S_DIR = 'k8s'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                // Checkout do repositório para acessar os diretórios do front-end e back-end
                checkout scm
            }
        }

        stage('Prepare Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                        
                        // Renomear imagens antigas como backup e manter apenas o último backup
                        sh """
                            docker pull ${DOCKER_REPO}:frontend-latest || true
                            docker tag ${DOCKER_REPO}:frontend-latest ${DOCKER_REPO}:frontend-backup || true
                            docker rmi ${DOCKER_REPO}:frontend-latest || true

                            docker pull ${DOCKER_REPO}:backend-latest || true
                            docker tag ${DOCKER_REPO}:backend-latest ${DOCKER_REPO}:backend-backup || true
                            docker rmi ${DOCKER_REPO}:backend-latest || true

                            # Excluir backups antigos, mantendo apenas o mais recente
                            docker images ${DOCKER_REPO} --format '{{.Repository}}:{{.Tag}}' | grep '-backup' | sort | head -n -1 | xargs -r docker rmi || true
                        """
                    }
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir(FRONTEND_DIR) {
                    script {
                        def frontendTag = "frontend-${new Date().format("yyyyMMdd-HHmmss")}"
                        docker.build("${DOCKER_REPO}:${frontendTag}", "-f Dockerfile .")
                        docker.tag("${DOCKER_REPO}:${frontendTag}", "${DOCKER_REPO}:frontend-latest")
                    }
                }
            }
        }

        stage('Build Backend') {
            steps {
                dir(BACKEND_DIR) {
                    script {
                        def backendTag = "backend-${new Date().format("yyyyMMdd-HHmmss")}"
                        docker.build("${DOCKER_REPO}:${backendTag}", "-f Dockerfile .")
                        docker.tag("${DOCKER_REPO}:${backendTag}", "${DOCKER_REPO}:backend-latest")
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        sh "docker push ${DOCKER_REPO}:frontend-latest"
                        sh "docker push ${DOCKER_REPO}:backend-latest"
                    }
                }
            }
        }

        stage('Test kubectl') {
            steps {
                script {
                    sh 'microk8s kubectl version --client'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                        // Aplicar YAMLs do diretório `k8s` no Kubernetes
                        sh """
                            microk8s kubectl apply -f ${K8S_DIR}/create-base-configmap.yaml
                            microk8s kubectl apply -f ${K8S_DIR}/database-pv.yaml
                            microk8s kubectl apply -f ${K8S_DIR}/database-service.yaml
                            microk8s kubectl apply -f ${K8S_DIR}/database-statefulset.yaml
                            microk8s kubectl apply -f ${K8S_DIR}/backend-deployment.yaml
                            microk8s kubectl apply -f ${K8S_DIR}/backend-service.yaml
                            microk8s kubectl apply -f ${K8S_DIR}/frontend-deployment.yaml
                            microk8s kubectl apply -f ${K8S_DIR}/frontend-service.yaml
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
    }
}
