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
                    
                    // Define uma variável para armazenar o status da varredura
                    // Por padrão, assumimos que não há vulnerabilidades críticas
                    env.CRITICAL_VULNERABILITIES_FOUND = "false"
                    env.TRIVY_REPORT = ""

                    try {
                        // Roda o Trivy.
                        // 'set +e' garante que o script continue mesmo se o Trivy retornar erro (exit code 1)
                        // '|| true' também garante que o comando não cause uma falha de pipeline se trivy retornar exit code 1
                        def trivyOutput = sh(
                            script: "trivy image --severity HIGH,CRITICAL --format json manaramarcelo/meuapp-backend:${env.BUILD_ID} | tee trivy-report.json",
                            returnStdout: true,
                            returnStatus: true
                        )

                        env.TRIVY_REPORT = trivyOutput.stdout

                        // Analisa o JSON do Trivy para verificar se há vulnerabilidades HIGH ou CRITICAL
                        // Se o 'jq' não estiver instalado no seu agente Jenkins, você precisará instalá-lo:
                        // sudo apt-get install -y jq
                        def highCriticalCount = sh(
                            script: "jq '.Vulnerabilities[] | select(.Severity == \"HIGH\" or .Severity == \"CRITICAL\") | length' trivy-report.json | wc -l",
                            returnStdout: true
                        ).trim()

                        if (highCriticalCount.toInteger() > 0) {
                            env.CRITICAL_VULNERABILITIES_FOUND = "true"
                            echo "⚠️ Trivy encontrou ${highCriticalCount} vulnerabilidades HIGH/CRITICAL. Verifique o log."
                        } else {
                            echo "✅ Nenhuma vulnerabilidade HIGH/CRITICAL encontrada pelo Trivy."
                        }

                    } catch (err) {
                        // Em caso de erro na execução do Trivy (não de vulnerabilidade, mas erro de comando)
                        echo "❌ Erro ao executar o Trivy: ${err}"
                        env.CRITICAL_VULNERABILITIES_FOUND = "error" // Indicador de erro na execução do scanner
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
                            slackMessage = "⚠️ Pipeline FINALIZADA com AVISOS de SEGURANÇA para a build ${env.BUILD_ID} na branch ${env.BRANCH_NAME}!\nForam encontradas vulnerabilidades HIGH/CRITICAL na imagem Docker. Analise o log do Jenkins para mais detalhes: ${env.BUILD_URL}"
                        } else if (env.CRITICAL_VULNERABILITIES_FOUND == "false") {
                            slackMessage = "✅ Pipeline FINALIZADA com SUCESSO para a build ${env.BUILD_ID} na branch ${env.BRANCH_NAME}!\nSem vulnerabilidades HIGH/CRITICAL detectadas. Aplicação implantada no Kubernetes: ${env.BUILD_URL}"
                        } else { 
                            slackMessage = "❌ Pipeline FINALIZADA com ERRO no SCAN de SEGURANÇA para a build ${env.BUILD_ID} na branch ${env.BRANCH_NAME}!\nErro ao executar o Trivy. Analise o log do Jenkins: ${env.BUILD_URL}"
                        }
                        // Comando curl usando a variável interpolada e a mensagem condicional
                        sh "curl -X POST -H 'Content-type: application/json' --data '{\"text\":\"${slackMessage}\"}' \"${SLACK_WEBHOOK}\""
                    }
                }
            }
        }
    }
    triggers {
        githubPush()
    }
}
