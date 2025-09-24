# Projeto 4: CI/CD com GitHub Actions, Docker e ArgoCD

Este repositório documenta a implementação de um pipeline completo de CI/CD (Integração Contínua e Entrega Contínua) para uma aplicação FastAPI. O projeto utiliza o GitHub Actions para automação de build e push de imagens Docker, e o ArgoCD para realizar o deploy contínuo em um ambiente Kubernetes, seguindo as melhores práticas de GitOps.

<div align="center">
    <img src="https://skillicons.dev/icons?i=github,githubactions,docker,kubernetes" />
    <img src="https://cdn.jsdelivr.net/gh/homarr-labs/dashboard-icons/svg/argo-cd.svg" width=40px />
</div>

## Tabela de Conteúdo

1.  [Visão Geral do Projeto](#1-visão-geral-do-projeto)
2.  [Pré-requisitos](#2-pré-requisitos)
3.  [Etapa 1: Configuração da Aplicação e Repositórios](#3-etapa-1-configuração-da-aplicação-e-repositórios)
4.  [Etapa 2: Criação dos Manifestos Kubernetes](#4-etapa-2-criação-dos-manifestos-kubernetes)
5.  [Etapa 3: Configuração do Pipeline CI/CD com GitHub Actions](#5-etapa-3-configuração-do-pipeline-cicd-com-github-actions)
6.  [Etapa 4: Deploy da Aplicação com ArgoCD](#6-etapa-4-deploy-da-aplicação-com-argocd)
7.  [Etapa 5: Validação do Fluxo End-to-End](#7-etapa-5-validação-do-fluxo-end-to-end)

---

## 1. Visão Geral do Projeto

O objetivo deste projeto é automatizar todo o ciclo de vida de uma aplicação, desde o `git push` de uma alteração no código até sua implantação em um cluster Kubernetes. Demonstra-se um fluxo de trabalho GitOps robusto, onde o Git atua como a fonte tanto para o código da aplicação quanto para a infraestrutura.

Este projeto prático aborda como empresas modernas mantêm agilidade e confiabilidade em seus deploys, integrando ferramentas líderes de mercado para criar um pipeline automatizado, seguro e observável.

* **CI (Integração Contínua) com GitHub Actions:** Automatiza o build da aplicação em um contêiner Docker e o envio da imagem para um registro (Docker Hub) a cada novo commit.
* **CD (Entrega Contínua) com ArgoCD:** Utiliza a metodologia GitOps para monitorar o repositório de manifestos. Ao detectar uma alteração (como uma nova tag de imagem), o ArgoCD atualiza automaticamente a aplicação no cluster Kubernetes.
* **Kubernetes:** Orquestra a execução dos contêineres, garantindo escalabilidade, resiliência e gerenciamento eficiente dos recursos da aplicação.

---

## 2. Pré-requisitos

Antes de começar, garanta que os seguintes softwares e contas estão configurados:

* Conta no GitHub (com repositórios públicos)
* Conta no Docker Hub com um Token de Acesso gerado
* Rancher Desktop (ou semelhantes) instalado e com Kubernetes habilitado
* ArgoCD instalado no cluster local
* Python 3 e Docker instalados e funcionando localmente

---

## 3. Etapa 1: Configuração da Aplicação e Repositórios

Nesta etapa, criamos os dois repositórios Git que formam a base do nosso fluxo GitOps e a aplicação FastAPI.

1.  **Crie o Repositório da Aplicação (`hello-app`):**
    * Crie um repositório público no GitHub chamado `hello-app`.
    * Dentro dele, adicione os seguintes arquivos:
        * `main.py` (Aplicação FastAPI):
            ```python
            from fastapi import FastAPI
            app=FastAPI()
            @app.get("/")
            async def root():
                return {"message": "Hello World"}
            ```
        * `Dockerfile` (Instruções para construir a imagem):
            ```dockerfile
            FROM python:3.9-slim
            WORKDIR /app
            RUN pip install --no-cache-dir "fastapi" "uvicorn[standard]"
            COPY . .
            CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "80"]
            ```

2.  **Crie o Repositório de Manifestos (`hello-manifests`):**
    * Crie um segundo repositório público no GitHub chamado `hello-manifests`. Este repositório ficará vazio por enquanto.

---

## 4. Etapa 2: Criação dos Manifestos Kubernetes

No repositório `hello-manifests`, criamos os arquivos que descrevem como nossa aplicação deve ser executada no Kubernetes.

1.  **Crie o `deployment.yaml`:**
    * Este arquivo gerencia os Pods da aplicação. Substitua `SEU-USUARIO-DOCKERHUB` pelo seu usuário.
        ```yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: hello-app
        spec:
          replicas: 2
          selector:
            matchLabels:
              app: hello-app
          template:
            metadata:
              labels:
                app: hello-app
            spec:
              containers:
              - name: hello-app-container
                image: SEU-USUARIO-DOCKERHUB/hello-app:latest
                ports:
                - containerPort: 80
        ```

2.  **Crie o `service.yaml`:**
    * Este arquivo expõe os Pods através de um ponto de acesso de rede estável.
        ```yaml
        apiVersion: v1
        kind: Service
        metadata:
          name: hello-app-service
        spec:
          selector:
            app: hello-app
          ports:
            - protocol: TCP
              port: 8080
              targetPort: 80
        ```
3.  **Faça o commit e push** desses dois arquivos para o repositório `hello-manifests`.

---

## 5. Etapa 3: Configuração do Pipeline CI/CD com GitHub Actions

Agora, configuramos o pipeline que automatiza o build e a atualização dos manifestos.

1.  **Configure os Segredos no Repositório `hello-app`:**
    * Vá em **Settings > Secrets and variables > Actions** e crie os seguintes segredos:
        * `DOCKER_USERNAME`: Seu nome de usuário do Docker Hub.
        * `DOCKER_PASSWORD`: Seu token de acesso do Docker Hub.
        * `SSH_PRIVATE_KEY`: Uma chave SSH privada que tem permissão de escrita (via Deploy Key) no repositório `hello-manifests`.

2.  **Crie o Arquivo de Workflow:**
    * No repositório `hello-app`, crie a estrutura `.github/workflows/` e, dentro dela, o arquivo `main.yml`.
    * Substitua `SEU-USUARIO-GITHUB` e `SEU-USUARIO-DOCKERHUB` pelos seus respectivos nomes.
        ```yaml
        name: CI-CD Pipeline
        on:
          push:
            branches:
              - main
        jobs:
          build-and-push:
            runs-on: ubuntu-latest
            outputs:
              image_tag: ${{ steps.meta.outputs.version }}
            steps:
              - name: Checkout repository
                uses: actions/checkout@v4
              - name: Log in to Docker Hub
                uses: docker/login-action@v3
                with:
                  username: ${{ secrets.DOCKER_USERNAME }}
                  password: ${{ secrets.DOCKER_PASSWORD }}
              - name: Extract metadata for Docker
                id: meta
                uses: docker/metadata-action@v5
                with:
                  images: ${{ secrets.DOCKER_USERNAME }}/hello-app
                  tags:
                    type=sha,prefix=,format=short
              - name: Build and push Docker image
                uses: docker/build-push-action@v5
                with:
                  context: .
                  push: true
                  tags: ${{ steps.meta.outputs.tags }}
          update-manifest:
            needs: build-and-push
            runs-on: ubuntu-latest
            steps:
              - name: Checkout manifests repository
                uses: actions/checkout@v4
                with:
                  repository: SEU-USUARIO-GITHUB/hello-manifests
                  ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
              - name: Update image tag in manifest
                run: |
                  sed -i 's|image: .*|image: ${{ secrets.DOCKER_USERNAME }}/hello-app:${{ needs.build-and-push.outputs.image_tag }}|g' deployment.yaml
              - name: Commit and push changes
                run: |
                  git config --global user.name "GitHub Actions"
                  git config --global user.email "actions@github.com"
                  git add deployment.yaml
                  git commit -m "Update image to tag ${{ needs.build-and-push.outputs.image_tag }}" || echo "No changes to commit"
                  git push
        ```
3.  **Faça o commit e push** da pasta `.github` para o repositório `hello-app` para ativar o pipeline.

---

## 6. Etapa 4: Deploy da Aplicação com ArgoCD

Com o pipeline CI/CD ativo, conectamos o ArgoCD ao nosso repositório de manifestos.

1.  **Acesse a Interface do ArgoCD:**
    * Obtenha a senha com `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`.
    * Exponha a porta com `kubectl port-forward svc/argocd-server -n argocd 8080:443`.
    * Acesse `https://localhost:8080` e faça login com o usuário `admin`.

2.  **Conecte o Repositório `hello-manifests`**:
    * Na interface, vá em **Settings > Repositories** e conecte seu repositório `hello-manifests` via HTTPS.

3.  **Crie o App no ArgoCD**:
    * Clique em **+ NEW APP** e preencha os campos:
        * **Application Name:** `hello-app`
        * **Project Name:** `default`
        * **Sync Policy:** `Automatic` (com `Prune Resources` e `Self Heal` ativados).
        * **Repository URL:** Selecione o repositório `hello-manifests`.
        * **Revision:** `HEAD`
        * **Path:** `.`
        * **Cluster URL:** `https://kubernetes.default.svc`
        * **Namespace:** `default`
    * Clique em **CREATE**. O ArgoCD irá sincronizar automaticamente e implantar a aplicação.

---

## 7. Etapa 5: Validação do Fluxo End-to-End

Por fim, validamos todo o ciclo, desde a alteração do código até o deploy.

1.  **Acesse a Aplicação Inicial:**
    * Execute o port-forward para o serviço:
        ```bash
        kubectl port-forward service/hello-app-service 8080:8080
        ```
    * Acesse `http://localhost:8080` no navegador. A mensagem "Hello World" deve aparecer.

2.  **Teste o Pipeline Completo:**
    * Altere a mensagem no arquivo `main.py` do repositório `hello-app` para algo novo.
    * Faça o `commit` e `push` da alteração.
    * **Observe a automação:**
        1.  O **GitHub Actions** irá iniciar, construir uma nova imagem e atualizar o `deployment.yaml` no `hello-manifests`.
        2.  O **ArgoCD** irá detectar a mudança, entrar em estado `OutOfSync` e sincronizar automaticamente, atualizando os pods para a nova versão.

3.  **Valide a Mudança:**
    * Acesse `http://localhost:8080` novamente. A nova mensagem deverá ser exibida, confirmando que todo o pipeline CI/CD está funcionando.
