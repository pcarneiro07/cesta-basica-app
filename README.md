Cesta Básica App: Análise e PrevisãoEste projeto implementa um dashboard interativo para análise da Cesta Básica, utilizando Machine Learning (XGBoost) para previsões e rodando 100% na Azure Container Apps.Arquitetura GeralO projeto utiliza uma arquitetura serverless e baseada em containers na Azure:GitHub (código fonte)Docker Build local (Codespaces)Azure Container Registry (ACR)cestabasica:latestAzure Blob Storagedata/ $\to$ dataset XLSXartifacts/ $\to$ modelo treinado + scalerAzure Container AppsExecuta o dashboard em Dash/Plotly.Carrega dados e artefatos do Blob Storage.Escalamento manual (ligar/desligar).Estrutura do RepositórioPlaintextcesta-basica-app/
│ ├── app.py → Dashboard em Dash
│ ├── train_model.py → Pipeline de treino + upload de artefatos
│ ├── Dockerfile → Build do container
│ ├── requirements.txt → Dependências da aplicação
│ ├── data/ → Dataset local (somente para build local)
│ └── artifacts/ → Artefatos locais (gerados no build)
Tecnologias UtilizadasPython 3.11Dash / PlotlyScikit-LearnPandas / NumPyAzure CLIAzure Container Registry (ACR)Azure Blob StorageAzure Container AppsDockerGitHub Actions: Pipeline de Integração Contínua (CI)O fluxo de CI é acionado em todo push para a branch main ou em qualquer pull_request para a main. Ele garante a validade do ambiente, treina o modelo e verifica o build do Docker.Fluxo do ci.ymlsetup: Instala o Python 3.11, faz o checkout, instala as dependências e valida as importações essenciais.train: Dependente do setup. Executa o script train_model.py (treino do modelo) e publica os artefatos de treino.docker-build: Dependente do train. Faz o build da imagem Docker localmente, realiza um smoke test e publica os logs de histórico do Docker.finalize: Dependente do docker-build. Confirma o sucesso de todo o pipeline de CI.Arquivo ci.ymlYAMLname: CI - Cesta Básica App

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  setup:
    name: Setup do Ambiente
    runs-on: ubuntu-latest

    steps:
      - name: Checkout do repositório
        uses: actions/checkout@v4

      - name: Instalar Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Instalar dependências
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Validar importações
        run: |
          python - << 'EOF'
          import pandas, numpy, dash, plotly, xgboost
          print("Imports OK")
          EOF

  train:
    name: Executar Pipeline de Treino
    runs-on: ubuntu-latest
    needs: setup

    steps:
      - name: Checkout do repositório
        uses: actions/checkout@v4

      - name: Instalar Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Instalar dependências
        run: pip install -r requirements.txt

      - name: Rodar train_model.py
        run: |
          echo "Executando pipeline de treino..."
          python train_model.py

      - name: Publicar artefatos do treino
        uses: actions/upload-artifact@v4
        with:
          name: artifacts-treino
          path: artifacts/

  docker-build:
    name: Build da Imagem Docker
    runs-on: ubuntu-latest
    needs: train
  
    steps:
      - name: Checkout do repositório
        uses: actions/checkout@v4
  
      - name: Build da imagem Docker
        run: |
          docker build -t cestabasica:ci .
  
      - name: Teste rápido (smoke test)
        run: |
          docker run --rm cestabasica:ci python -c "print('Container OK')"
  
      - name: Publicar logs do Docker
        run: |
          docker history cestabasica:ci > docker_history.txt
          cat docker_history.txt
  
      - name: Upload dos logs
        uses: actions/upload-artifact@v4
        with:
          name: docker-logs
          path: docker_history.txt


  finalize:
    name: Finalização
    runs-on: ubuntu-latest
    needs: [docker-build]

    steps:
      - name: Mensagem final
        run: echo "Pipeline CI concluído com sucesso!"
Deploy Manual — Fluxo Oficial do ProjetoEste é o fluxo oficial, testado e estável usado para implantar a aplicação após o CI:1. Preparar o ambienteInstalar Azure CLI (somente no Codespaces):Bashcurl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
Login:Bashaz login
2. Build da imagem DockerBashdocker build -t cestabasica:latest .
3. Envio para o Azure Container Registry (ACR)Substitua <ACR_NAME> pelo nome do seu registro:Bashdocker tag cestabasica:latest <ACR_NAME>.azurecr.io/cestabasica:latest
docker push <ACR_NAME>.azurecr.io/cestabasica:latest
4. Atualizar o Container AppAtualiza o Container App para usar a nova imagem.Bashaz containerapp update \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  --image <ACR_NAME>.azurecr.io/cestabasica:latest
Como Ligar e Desligar o DashboardO Container App ajusta o escalamento via minReplicas.Ligar o dashboard (minReplicas=1)Bashaz containerapp update \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  --set template.scale.minReplicas=1
Desligar o dashboard (minReplicas=0)Bashaz containerapp update \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  --set template.scale.minReplicas=0
Verificar estado das revisõesBashaz containerapp revision list \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  -o table
Resultado: Replicas = 1 (ligado) / Replicas = 0 (desligado)Armazenamento na AzureA aplicação lê seus recursos e persiste os artefatos de ML a partir do Azure Blob Storage:Container data/: Dataset XLSXContainer artifacts/: Modelo treinado, Scaler e outros artefatos.A conexão é configurada no Container App via variável de ambiente: BLOB_CONNECTION_STRING.Treinamento Automático no BuildO script train_model.py é executado automaticamente durante o docker build. Ele:Lê o XLSX (local ou do Blob).Treina o modelo.Salva artefatos localmente (para a imagem Docker).Realiza o upload dos artefatos para o Blob Storage.Dashboard FinalO Dashboard:Lê o dataset e os artefatos do Blob.Aplica o modelo.Exibe análises em Dash/Plotly.Roda 100% dentro do Azure Container Apps.Atualiza automaticamente ao publicar nova imagem no ACR.ConclusãoEste projeto entrega:Pipeline completo (dados + modelo + deploy)Infraestrutura Azure funcionalDeploy estável em Container AppsReprodutibilidade via DockerSimplicidade operacional (ligar/desligar por 1 comando)
