pipeline {
    agent any

    environment {
        DOCKER_REPO = 'nicolasgreskiv/pucpr-gh-pages'
        FRONTEND_DIR = 'projeto-web'
        BACKEND_DIR = 'projeto-spring'
        K8S_DIR = 'k8s'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs() // Limpa o workspace antes do build para evitar arquivos antigos
            }
        }

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Docker Hub Backup') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"

                        // Renomear imagens atuais como backup e excluir backups antigos se necessário
                        sh """
                            echo "Preparando backup das imagens Docker Hub..."

                            # Renomear a imagem frontend-latest para frontend-backup se existir
                            docker pull ${DOCKER_REPO}:frontend-latest || true
                            docker tag ${DOCKER_REPO}:frontend-latest ${DOCKER_REPO}:frontend-backup || true

                            # Renomear a imagem backend-latest para backend-backup se existir
                            docker pull ${DOCKER_REPO}:backend-latest || true
                            docker tag ${DOCKER_REPO}:backend-latest ${DOCKER_REPO}:backend-backup || true

                            # Manter apenas um backup para cada imagem
                            docker images ${DOCKER_REPO} --format '{{.Repository}}:{{.Tag}}' | grep 'backup' | tail -n +2 | xargs -r docker rmi || true
                        """
                    }
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir(FRONTEND_DIR) {
                    script {
                        def frontendTag = "frontend-latest"
                        sh "docker build --no-cache -t ${DOCKER_REPO}:${frontendTag} -f Dockerfile ."
                    }
                }
            }
        }

        stage('Build Backend') {
            steps {
                dir(BACKEND_DIR) {
                    script {
                        def backendTag = "backend-latest"
                        sh "docker build --no-cache -t ${DOCKER_REPO}:${backendTag} -f Dockerfile ."
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                        // Enviar as novas imagens para o Docker Hub
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
            cleanWs() // Limpa o workspace após o build
        }
    }
}
