# Cesta Básica App — README Completo
---

# 1. Arquitetura Geral

```
GitHub (código fonte)
      │
      ├── Docker Build local (Codespaces)
      │
      ├── Azure Container Registry (ACR)
      │        • cestabasica:latest
      │
      ├── Azure Blob Storage
      │        • data/       → dataset XLSX
      │        • artifacts/  → modelo treinado + scaler
      │
      └── Azure Container Apps
               • Executa o dashboard
               • Carrega dados e artefatos do Blob
               • Escalamento manual (ligar/desligar)
```

---

# 2. Estrutura do Repositório

```
cesta-basica-app/
│
├── app.py                # Dashboard em Dash
├── train_model.py        # Pipeline de treino + upload de artefatos
├── Dockerfile            # Build do container
├── requirements.txt      # Dependências
│
├── data/                 # Dataset local (somente para build local)
└── artifacts/            # Artefatos locais (gerados no build)
```

---

# 3. Tecnologias Utilizadas

- Python 3.11  
- Dash / Plotly  
- Scikit-Learn  
- Pandas / NumPy  
- Docker  
- Azure CLI  
- Azure Container Registry (ACR)  
- Azure Blob Storage  
- Azure Container Apps  
- GitHub Actions  

---

# 4. Deploy Manual — Fluxo Oficial do Projeto

## 4.1 Preparar o Ambiente (Codespaces)

Instalar Azure CLI:

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

Login na Azure:

```bash
az login
```

Se usar Service Principal:

```bash
az login --service-principal     --username $APP_ID     --password $PASSWORD     --tenant $TENANT_ID
```

Verificar a assinatura:

```bash
az account show
```

---

## 4.2 Build da imagem Docker

```bash
docker build -t cestabasica:latest .
```

---

## 4.3 Envio da imagem para o ACR

Tag da imagem:

```bash
docker tag cestabasica:latest <NOME_ACR>.azurecr.io/cestabasica:latest
```

Login no ACR:

```bash
az acr login --name <NOME_ACR>
```

Push:

```bash
docker push <NOME_ACR>.azurecr.io/cestabasica:latest
```

---

## 4.4 Atualizar o Container App para usar a nova imagem

```bash
az containerapp update   --name dashboard-cesta-basica   --resource-group rg-cesta-basica   --image <NOME_ACR>.azurecr.io/cestabasica:latest
```

---

# 5. Ligar e Desligar o Dashboard (minReplicas)

## Ligar (minReplicas = 1)

```bash
az containerapp update   --name dashboard-cesta-basica   --resource-group rg-cesta-basica   --set template.scale.minReplicas=1
```

## Desligar (minReplicas = 0)

```bash
az containerapp update   --name dashboard-cesta-basica   --resource-group rg-cesta-basica   --set template.scale.minReplicas=0
```

---

# 6. Verificar estado das revisões

```bash
az containerapp revision list   --name dashboard-cesta-basica   --resource-group rg-cesta-basica   -o table
```

Saída esperada:

- **Replicas = 1** → ligado  
- **Replicas = 0** → desligado  

---

# 7. Armazenamento — Azure Blob Storage

A aplicação usa os contêineres:

### data/
- Dataset XLSX utilizado no treinamento e no dashboard

### artifacts/
- model.pkl  
- scaler.pkl  
- outros artefatos gerados no build

Conexão realizada por variável de ambiente:

```
BLOB_CONNECTION_STRING
```

Configurada dentro do Azure Container Apps.

---

# 8. Treinamento Automático Durante o Build

O Dockerfile executa automaticamente:

1. Leitura do dataset  
2. Treinamento do modelo  
3. Geração de artefatos  
4. Upload para o Blob Storage  

Comandos executados pelo Dockerfile:

```bash
python train_model.py
```

---

# 9. Dashboard

O dashboard:

- Lê dataset do Blob  
- Carrega o modelo treinado  
- Exibe análises e gráficos  
- Roda integralmente dentro do Azure Container Apps  
- Atualiza automaticamente após publicação de nova imagem no ACR

---

# 10. GitHub Actions — Pipeline CI Completo

## Etapa 1 — Setup do Ambiente
Executa:
- checkout  
- instalação do Python  
- pip install  
- validação de importações  

## Etapa 2 — Pipeline de Treino
Executa:
- train_model.py  
- upload de artefatos  

## Etapa 3 — Build Docker
Executa:
- construção da imagem  
- smoke test  
- upload de logs  

## Etapa 4 — Finalização
Apenas registra conclusão da pipeline.

---

# Workflow REAL do projeto

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

---

# 11. Conclusão

O projeto entrega:

- Pipeline completo (dados → modelo → deploy)  
- Build automatizado com treinamento  
- Dashboard em produção via Azure Container Apps  
- Infraestrutura estável e reproduzível  
- GitHub Actions simples e funcional  
- Simplicidade operacional com minReplicas  

