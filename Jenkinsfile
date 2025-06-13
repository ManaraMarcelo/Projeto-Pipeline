pipeline {
    agent any

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
        stage('Deploy to Kubernetes') {
            environment {
                tag_version = "${env.BUILD_ID}"
            }
            steps {
                script {
                    sh 'sed -i "s/{{tag}}/$tag_version/g" ./k8s/trabalho-deployment.yaml'
                    sh 'kubectl apply -f ./k8s/trabalho-deployment.yaml'
                    sh 'kubectl apply -f ./k8s/trabalho-service.yaml'
                    sh 'echo "Deployment completed with tag: $tag_version"'
                    sh 'echo "Waiting for the deployment to be ready..."'
                }
            }
        }
        stage('Notificar no Slack') {
            environment{
                SLACK_WEBHOOK = credentials('slack-credentials')
            }
            steps {
                script {
                    sh '''
                        curl -X POST -H 'Content-type: application/json' \
                        --data '{"text":"✅ Deploy realizado com sucesso!"}' \
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
                chuckNorris()
            }
        }
        failure {
            echo '❌ Build falhou. Mas Chuck Norris nunca falha.'
        }
    }

    triggers {
        githubPush()
    }
}
