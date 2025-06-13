# 🚀 Pipeline CI/CD com FastAPI, Docker, Jenkins e Kubernetes

Este projeto tem como objetivo principal ensinar e aplicar na prática os conceitos de **Integração Contínua (CI)** e **Entrega Contínua (CD)**. Desenvolvemos uma esteira de automação completa para uma aplicação backend (FastAPI) e um frontend simples, realizando o deploy automatizado em um cluster Kubernetes local via Jenkins.

Sempre que novas alterações são enviadas para o repositório Git, a pipeline é acionada automaticamente, construindo, testando (futuramente), publicando e implantando a nova versão da aplicação.

---

## 📌 Índice
1️⃣ [Visão Geral do Projeto](#1️⃣-visão-geral-do-projeto)  
2️⃣ [Tecnologias Utilizadas](#2️⃣-tecnologias-utilizadas)  
3️⃣ [Estrutura do Projeto](#3️⃣-estrutura-do-projeto)  
4️⃣ [Fase 1: Preparação do Projeto](#4️⃣-fase-1-preparação-do-projeto)  
5️⃣ [Fase 2: Conteinerização com Docker](#5️⃣-fase-2-conteinerização-com-docker)  
6️⃣ [Fase 3: Arquivos de Deploy no Kubernetes](#6️⃣-fase-3-arquivos-de-deploy-no-kubernetes)  
7️⃣ [Fase 4: Jenkins - Build e Push](#7️⃣-fase-4-jenkins---build-e-push)  
8️⃣ [Fase 5: Jenkins - Deploy no Kubernetes](#8️⃣-fase-5-jenkins---deploy-no-kubernetes)  
9️⃣ [Fase 6: Documentação](#9️⃣-fase-6-documentação)  
🔟 [Desafios Extras](#🔟-desafios-extras)  
1️⃣1️⃣ [Teste Final da Aplicação](#1️⃣1️⃣-teste-final-da-aplicação)  
✅ [Conclusão](#✅-conclusão)  

---

## 1️⃣ Visão Geral do Projeto
Este projeto demonstra a construção de uma pipeline de CI/CD para uma aplicação web Fullstack (Backend FastAPI + Frontend Estático). A automação abrange desde o versionamento do código no GitHub até a implantação contínua em um ambiente Kubernetes local, utilizando Jenkins como orquestrador.

---

## 2️⃣ Tecnologias Utilizadas
* **GitHub:** Versionamento de código e acionamento de pipeline via Webhooks.
* **FastAPI:** Framework web em Python para o desenvolvimento do Backend.
* **Python:** Linguagem de programação para o Backend.
* **Docker:** Para conteinerização da aplicação (Backend e Frontend).
* **Docker Hub:** Registro público de imagens Docker.
* **Jenkins:** Ferramenta de automação de CI/CD.
* **Kubernetes Local:** Rancher Desktop (ou Minikube/Kind) para orquestração de contêineres.
* **`kubectl`:** Ferramenta de linha de comando para interagir com o Kubernetes.
* **`trivy`:** Scanner de vulnerabilidades para imagens Docker (usado nos desafios extras).
* **`ngrok`:** Para expor o Jenkins local à internet e permitir que o GitHub envie Webhooks.

---

## 3️⃣ Estrutura do Projeto
O projeto está organizado da seguinte forma no repositório Git:

projeto-pipeline/  
├── src/  
│   ├── backend/  
│   │   ├── main.py  
│   │   ├── requirements.txt  
│   │   └── Dockerfile  
│   └── frontend/ # Contém arquivos HTML/CSS/JS (servidos pelo backend)  
│       └── index.html  
├── k8s/  
│   ├── backend-deployment.yaml # Renomeado para consistência  
│   └── backend-service.yaml    # Renomeado para consistência  
└── Jenkinsfile  

---

## 4️⃣ Fase 1: Preparação do Projeto
Nesta fase, o ambiente de desenvolvimento e os repositórios são preparados.

### 4.1. Repositório GitHub
* Criação de um **novo repositório** no GitHub (e.g., `Projeto-Pipeline`).
* Configuração de branches:
    * `dev`: Branch principal de desenvolvimento. Todas as novas funcionalidades e testes iniciais são feitos aqui.
    * `main` (ou `master`): Branch de produção. Recebe código apenas após validação completa na `dev`.
* **Link do Repositório:** `https://github.com/ManaraMarcelo/Projeto-Pipeline.git`

### 4.2. Conta no Docker Hub
* Criação de conta no Docker Hub para armazenar as imagens da aplicação.
* **Usuário Docker Hub:** `manaramarcelo`

### 4.3. Acesso ao Cluster Kubernetes Local
* Verificação do funcionamento do Rancher Desktop (ou alternativa) e do acesso ao cluster Kubernetes via `kubectl`.
    ```bash
    kubectl get nodes
    ```

### 4.4. Validação da Aplicação Local (Backend FastAPI)
* Instalação de dependências Python (`fastapi`, `uvicorn`, `httpx`, `pydantic`) e execução local do `main.py`.
    ```bash
    cd <SEU_REPOSITORIO_LOCAL>/src/backend/
    pip install -r requirements.txt
    uvicorn main:app --host 0.0.0.0 --port 8000
    ```
* Acesso no navegador: `http://localhost:8000`.

**Conteúdo de `src/backend/requirements.txt`: [requeriments.txt](https://github.com/ManaraMarcelo/Projeto-Pipeline/blob/main/src/requirements.txt)**

**Conteúdo de `src/backend/main.py`: [main.py](https://github.com/ManaraMarcelo/Projeto-Pipeline/blob/main/src/main.py)**

**Conteúdo de `src/frontend/index.html`: [index.html](https://github.com/ManaraMarcelo/Projeto-Pipeline/blob/main/src/frontend/index.html)**

## 5️⃣ Fase 2: Conteinerização com Docker

Nesta fase, o backend (FastAPI) é empacotado em uma imagem Docker e publicado. O frontend será empacotado junto ao backend nesta etapa.

### 5.1. Criação do Dockerfile (Backend com Frontend Estático)

No diretório `src/backend/`, crie o arquivo `Dockerfile`.

Você pode visualizar o conteúdo completo do Dockerfile [Dockerfile](https://github.com/ManaraMarcelo/Projeto-Pipeline/blob/main/src/Dockerfile).

### 5.2. Teste Local (Build e Execução)

No diretório `src/backend/`, execute os comandos para construir a imagem e rodar o contêiner localmente.

Acesse a aplicação via navegador em [http://localhost:8000](http://localhost:8000). Se funcionar, o contêiner está ok.

### 5.3. Publicação no Docker Hub

A imagem construída será publicada no Docker Hub através da pipeline Jenkins (Fase 4).

### 5.4. Versionamento

O Dockerfile e o código da aplicação são versionados no GitHub.

---

## 6️⃣ Fase 3: Arquivos de Deploy no Kubernetes

Nesta fase, definimos como a aplicação será implantada no cluster Kubernetes local.

### 6.1. Criação dos YAMLs de Deployment e Service

Crie a pasta `k8s/` na raiz do seu repositório para organizar os manifestos do Kubernetes.

Os arquivos criados são:

- `k8s/backend-deployment.yaml`
- `k8s/backend-service.yaml`

Você pode visualizar ambos os arquivos diretamente no GitHub:

📁 [Ver arquivos de deploy no Kubernetes (pasta k8s/)](https://github.com/ManaraMarcelo/Projeto-Pipeline/tree/main/k8s)

### 6.2. Aplicação Manual e Validação no Kubernetes

Para aplicar os manifestos, utilize os comandos `kubectl apply` apontando para os arquivos YAML.

Após a aplicação, verifique o status dos pods e serviços com os comandos `kubectl get pods` e `kubectl get service`.

#### 🔥 Dica de Segurança: 

Configure o **Firewall do Windows** para permitir tráfego na porta utilizada pelo `NodePort` (ex: 30001), garantindo o acesso à aplicação via `http://localhost:30001`.

---

## 7️⃣ Fase 4: Jenkins - Build e Push

Nesta fase, criamos a pipeline Jenkins para automatizar o build e o push da imagem Docker da aplicação.

### 7.1. Criação da Nova Pipeline no Jenkins

1. Acesse o Jenkins via [http://localhost:8080](http://localhost:8080).
2. Clique em **"Novo Item"**, nomeie como `fastapi-cicd-pipeline`, selecione **"Pipeline"** e clique em **"OK"**.
3. Configure como:
   - **Definition**: Pipeline script from SCM.
   - **SCM**: Git.
   - **URL do Repositório**: Insira a URL HTTPS do seu repositório GitHub.
   - **Branches to build**: `*/dev`
   - **Script Path**: `Jenkinsfile` (deve estar na raiz do repositório).

🔗 [Jenkinsfile](https://github.com/ManaraMarcelo/Projeto-Pipeline/blob/main/Jenkinsfile)

### 7.2. Jenkinsfile (Build e Push Stages)

O `Jenkinsfile` contém os estágios principais para:

- **Checkout do repositório**
- **Build da imagem Docker**
- **Push da imagem para o Docker Hub**

### 7.3. Automação com Webhook (Git Push Trigger)

Para que a pipeline seja executada automaticamente a cada push na branch `dev`:

#### No Jenkins:
- Vá até o job `fastapi-cicd-pipeline`.
- Clique em **"Configurar"** > "Acionadores de Build".
- Marque a opção **"GitHub hook trigger for GITScm polling"**.

#### No GitHub:
1. Acesse **Settings** do seu repositório.
2. Vá até **Webhooks** > **"Add webhook"**.
3. Configure:
   - **Payload URL**: `https://<SEU_URL_NGROK>/github-webhook/`
   - **Content type**: `application/json`
   - **Which events**: *Just the push event*
4. Clique em **"Add webhook"**.

⚠️ Certifique-se de que o **ngrok** esteja rodando com `ngrok http 8080`.

#### Teste:
Realize um `git push` para a branch `dev`. Acesse o Jenkins e verifique se a pipeline foi acionada automaticamente.

---

# 8️⃣ Fase 5: Jenkins - Deploy no Kubernetes

Nesta fase, a pipeline Jenkins é estendida para realizar o deploy da aplicação diretamente no Kubernetes.

---

## 8.1. Acesso do Jenkins ao `kubectl`

Para que o Jenkins consiga aplicar manifestos no Kubernetes:

- ✅ Certifique-se de que o `kubectl` está disponível no `PATH` do usuário Jenkins.
- ✅ O arquivo `kubeconfig` do Rancher Desktop foi copiado para `/var/lib/jenkins/.kube/config` com as permissões corretas.
- ✅ Crie uma credencial do tipo **"Texto Secreto"** no Jenkins com o conteúdo do `kubeconfig`.
  - ID da credencial: `kubeconfig`

---

## 8.2. Atualização do Jenkinsfile (Stage de Deploy)

Um novo **stage de deploy** é adicionado ao `Jenkinsfile`, utilizando o `kubectl` para aplicar os manifestos Kubernetes.

🔗 [Ver Jenkinsfile com Deploy no GitHub](https://github.com/ManaraMarcelo/Projeto-Pipeline/blob/main/Jenkinsfile)

Esse novo estágio faz:
- Substituição da **tag da imagem** no arquivo `backend-deployment.yaml`
- Aplicação dos arquivos de `Deployment` e `Service`
- Verificação do rollout do deployment (com timeout de 5 minutos)

📁 [Ver arquivos YAML no GitHub](https://github.com/ManaraMarcelo/Projeto-Pipeline/tree/main/k8s)

---

## 8.3. Teste da Pipeline e Estratégia de Branches

### ✅ Teste na Branch `dev`:
- Faça um `git push` para a branch `dev`.
- A pipeline será executada:
  - Build da imagem
  - Push para Docker Hub
  - Deploy para o Kubernetes

### 🚀 Deploy em Produção (`main`):
Após validação na `dev`, realize o merge com a branch `main`:

```bash
git checkout main
git merge dev
git push origin main


