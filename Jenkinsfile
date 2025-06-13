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
        stage('Scan com Trivy') {
            steps {
                script {
                    echo "🔍 Escaneando a imagem com Trivy (somente vulnerabilidades HIGH e CRITICAL)..."

                    env.CRITICAL_VULNERABILITIES_FOUND = "false"
                    env.TRIVY_REPORT = "" // Inicializa a variável para o relatório

                    try {
                        // Captura a saída e o status. A flag 'returnStatus: true' permite verificar o status do comando.
                        // A flag 'returnStdout: true' captura a saída.
                        // O '|| true' no script garante que o comando não cause uma exceção Groovy,
                        // mas ainda permite que 'returnStatus' capture o código de saída real do Trivy.
                        def trivyResult = sh(
                            script: "trivy image --severity HIGH,CRITICAL --format json manaramarcelo/meuapp-backend:${env.BUILD_ID} | tee trivy-report.json || true",
                            returnStdout: true,
                            returnStatus: true
                        )

                        env.TRIVY_REPORT = trivyResult.stdout // Atribui a saída para a variável de ambiente

                        // Verifica o status do Trivy para determinar se encontrou vulnerabilidades (0 = sem high/critical, 1 = com high/critical)
                        if (trivyResult.status == 1) { // Se o Trivy saiu com 1, significa que encontrou HIGH/CRITICAL
                            env.CRITICAL_VULNERABILITIES_FOUND = "true"
                            echo "⚠️ Trivy encontrou vulnerabilidades HIGH/CRITICAL. Verifique o log."
                        } else if (trivyResult.status == 0) {
                            echo "✅ Nenhuma vulnerabilidade HIGH/CRITICAL encontrada pelo Trivy."
                        } else { // Outros códigos de erro do Trivy (ex: Trivy não encontrado, etc.)
                            echo "❌ Erro inesperado na execução do Trivy (código: ${trivyResult.status}). Analise o log."
                            env.CRITICAL_VULNERABILITIES_FOUND = "error"
                        }
                    } catch (err) {
                        // Captura erros se o comando 'sh' em si falhar (ex: trivy ou jq não instalados)
                        echo "❌ Erro CRÍTICO ao executar o Trivy/jq: ${err}"
                        env.CRITICAL_VULNERABILITIES_FOUND = "error"
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
            steps {
                withCredentials([string(credentialsId: 'slack-credentials', variable: 'SLACK_WEBHOOK')]) {
                    script {
                        def slackMessage = ""
                        if (env.CRITICAL_VULNERABILITIES_FOUND == "true") {
                            slackMessage = "⚠️ Pipeline FINALIZADA com AVISOS de SEGURANÇA para a build ${env.BUILD_ID} na branch ${env.BRANCH_NAME ?: 'main'}!\nForam encontradas vulnerabilidades HIGH/CRITICAL na imagem Docker. Analise o log do Jenkins para mais detalhes: ${env.BUILD_URL}"
                        } else if (env.CRITICAL_VULNERABILITIES_FOUND == "false") {
                            slackMessage = "✅ Pipeline FINALIZADA com SUCESSO para a build ${env.BUILD_ID} na branch ${env.BRANCH_NAME ?: 'main'}!\nSem vulnerabilidades HIGH/CRITICAL detectadas. Aplicação implantada no Kubernetes: ${env.BUILD_URL}"
                        } else {
                            slackMessage = "❌ Pipeline FINALIZADA com ERRO no SCAN de SEGURANÇA para a build ${env.BUILD_ID} na branch ${env.BRANCH_NAME ?: 'main'}!\nErro ao executar o Trivy. Analise o log do Jenkins: ${env.BUILD_URL}"
                        }

                        sh "curl -X POST -H 'Content-type: application/json' --data '{\"text\":\"${slackMessage}\"}' \"${SLACK_WEBHOOK}\""
                    }
                }
            }
        }
        stage('Declarative: Post Actions') {
            steps {
                echo "Build finalizado. Estado: ${currentBuild.result}"
            }
        }
    }
    triggers {
        githubPush()
    }
}
