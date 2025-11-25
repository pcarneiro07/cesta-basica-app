# 📌 Arquitetura Geral

text
GitHub (código fonte)
│
├── Docker Build local (Codespaces)
│
├── Azure Container Registry (ACR)
│   • cestabasica:latest
│
├── Azure Blob Storage
│   • data/      → dataset XLSX
│   • artifacts/ → modelo treinado + scaler
│
└── Azure Container Apps
    • Executa o dashboard
    • Carrega dados e artefatos do Blob
    • Escalamento manual (ligar/desligar)


---

## 📁 Estrutura do Repositório

text
cesta-basica-app/
│
├── app.py              → Dashboard em Dash
├── train_model.py      → Pipeline de treino + upload de artefatos
├── Dockerfile          → Build do container
├── requirements.txt    → Dependências da aplicação
├── data/               → Dataset local (somente para build local)
└── artifacts/          → Artefatos locais (gerados no build)


---

## 🔧 Tecnologias Utilizadas

* *Python 3.11*
* *Dash / Plotly*
* *Scikit-Learn*
* *Pandas / NumPy*
* *Azure CLI*
* *Azure Container Registry (ACR)*
* *Azure Blob Storage*
* *Azure Container Apps*
* *Docker*

---

## ⚙️ Deploy Manual — Fluxo Oficial do Projeto

Este é o fluxo oficial, testado e estável usado para implantar a aplicação.

### 1️⃣ Preparar o ambiente

Instalar Azure CLI (somente no Codespaces):
bash
curl -sL [https://aka.ms/InstallAzureCLIDeb](https://aka.ms/InstallAzureCLIDeb) | sudo bash


Login:
bash
az login


### 2️⃣ Build da imagem Docker

bash
docker build -t cestabasica:latest .


### 3️⃣ Envio para o Azure Container Registry

bash
docker tag cestabasica:latest .azurecr.io/cestabasica:latest
docker push .azurecr.io/cestabasica:latest


### 4️⃣ Atualizar o Container App para usar a nova imagem

bash
az containerapp update \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  --image .azurecr.io/cestabasica:latest


---

## 🚀 Como Ligar e Desligar o Dashboard

O Container App não usa start/stop tradicionais — ajustamos minReplicas.

### ▶️ Ligar o dashboard
bash
az containerapp update \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  --set template.scale.minReplicas=1


### ⏹️ Desligar o dashboard
bash
az containerapp update \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  --set template.scale.minReplicas=0


### 📡 Verificar estado das revisões
bash
az containerapp revision list \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  -o table


*Resultado esperado:*
* Replicas = 1 → ligado
* Replicas = 0 → desligado

---

## 🗂️ Armazenamento na Azure

A aplicação lê tudo a partir de:

1.  *Container data/*
    * Dataset XLSX
2.  *Container artifacts/*
    * Modelo treinado (model.pkl)
    * Scaler
    * Outros artefatos gerados

A conexão é feita via:
BLOB_CONNECTION_STRING
Que fica configurada no Azure Container Apps como variável de ambiente.

---

## 🧪 Treinamento Automático no Build

Durante o docker build, o script train_model.py é executado automaticamente.

Ele:
1.  Lê o XLSX (local ou do Blob).
2.  Treina o modelo.
3.  Salva artefatos localmente.
4.  Realiza o upload para o Blob Storage.

---

## 🎨 Dashboard Final

O dashboard:
* lê o dataset do Blob,
* aplica o modelo,
* exibe análises,
* roda 100% dentro do Azure Container Apps,
* atualiza automaticamente sempre que a imagem é publicada novamente no ACR.

---

## 📌 Conclusão

Este projeto entrega:
* ✅ Pipeline completo (dados + modelo + deploy)
* ✅ Infraestrutura Azure funcional
* ✅ Deploy estável em Container Apps
* ✅ Reprodutibilidade via Docker
* ✅ Simplicidade operacional (ligar/desligar por 1 comando)
