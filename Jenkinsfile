pipeline {
    agent any

    environment {
        tag_version = "${env.BUILD_ID}"
    }

    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    dockerapp = docker.build("manaramarcelo/meuapp-backend:${env.BUILD_ID}", '-f ./src/Dockerfile ./src')
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
                        dockerapp.push('latest')
                        dockerapp.push("${env.BUILD_ID}") 
                    }
                }
            }
        }

        stage('Scan com Trivy') {
            steps {
                script {
                    sh """
                        echo '🔍 Escaneando a imagem com Trivy...'
                        trivy image --exit-code 1 --severity HIGH,CRITICAL manaramarcelo/meuapp-backend:${env.BUILD_ID}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh 'sed -i "s/{{tag}}/$tag_version/g" ./k8s/trabalho-deployment.yaml'
                    sh 'kubectl apply -f ./k8s/trabalho-deployment.yaml'
                    sh 'kubectl apply -f ./k8s/trabalho-service.yaml'
                    sh 'echo "✅ Deployment concluído com a tag: $tag_version"'
                }
            }
        }

        stage('Notificar no Slack') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'slack-credentials', variable: 'SLACK_WEBHOOK')]) {
                        sh '''
                            curl -X POST -H 'Content-type: application/json' \
                            --data '{"text":"✅ Deploy finalizado com sucesso no ambiente Kubernetes."}' \
                            "$SLACK_WEBHOOK"
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                echo '😎 Chuck Norris approved this build.'
                echo '💬 “Chuck Norris doesn’t deploy applications. He roundhouse-kicks them into production.”'
            }
        }
        failure {
            script {
                withCredentials([string(credentialsId: 'slack-credentials', variable: 'SLACK_WEBHOOK')]) {
                    sh '''
                        curl -X POST -H 'Content-type: application/json' \
                        --data '{"text":"❌ Build falhou. Mas Chuck Norris nunca falha."}' \
                        "$SLACK_WEBHOOK"
                    '''
                }
            }
        }
    }

    triggers {
        githubPush()
    }
}
