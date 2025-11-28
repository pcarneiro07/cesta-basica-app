# Cesta Básica App  
Dashboard de análise e previsão de preços utilizando Machine Learning e infraestrutura em nuvem na Azure.

## Visão Geral
O Cesta Básica App integra processamento de dados, treinamento de modelo, automação via GitHub Actions e execução em ambiente de produção utilizando Azure Container Apps. O sistema foi projetado para ser simples, reprodutível e escalável.

## Arquitetura Geral
A arquitetura segue o fluxo:
- O código-fonte é mantido no GitHub.  
- O build é executado localmente através de Docker.  
- A imagem gerada é enviada ao Azure Container Registry (ACR).  
- O Azure Blob Storage armazena datasets e artefatos do modelo.  
- O Azure Container Apps executa o dashboard consumindo imagem e artefatos.

## Estrutura do Repositório
- app.py — dashboard principal  
- train_model.py — pipeline de treinamento  
- requirements.txt — dependências  
- Dockerfile — definição da imagem  
- data/ — dados usados localmente  
- artifacts/ — artefatos do modelo

## Armazenamento
O Azure Blob Storage contém:
- data/ — datasets  
- artifacts/ — modelo treinado e transformadores

A aplicação utiliza uma variável de ambiente configurada no Azure Container Apps para acessar o storage:
BLOB_CONNECTION_STRING

## Treinamento Automático
Durante a construção da imagem Docker, o script de treinamento é executado automaticamente. Isso garante que toda nova versão da aplicação use um modelo atualizado.

## Execução no Azure Container Apps
O Container Apps:
- Obtém a imagem mais recente do ACR  
- Carrega os artefatos do Blob Storage  
- Executa o dashboard  
- Permite ligar ou desligar a aplicação ajustando minReplicas  

## GitHub Actions — Pipeline Oficial (CI)
A automação existente no repositório segue exatamente o workflow abaixo:

### Fases da Pipeline

#### 1. Setup do Ambiente
- Checkout do repositório  
- Instalação do Python  
- Instalação das dependências  
- Validação de importações

#### 2. Execução do Pipeline de Treino
- Instalação do Python e dependências  
- Execução do script train_model.py  
- Upload dos artefatos gerados para consulta posterior

#### 3. Build da Imagem Docker
- Construção da imagem  
- Execução de um teste rápido (smoke test)  
- Publicação do histórico da imagem como artefato  

#### 4. Finalização
- Mensagem de conclusão no GitHub Actions

### Conteúdo do Workflow CI (para referência explicativa)

```yaml
name: CI - Cesta Básica App

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
```

## Benefícios
- Pipeline completo automatizado  
- Modelo atualizado automaticamente  
- Dashboard reprodutível e padronizado  
- Armazenamento centralizado  
- Fluxo CI simples e transparente  

---

Caso deseje incluir imagens, diagramas ou roadmap, posso adicionar.
