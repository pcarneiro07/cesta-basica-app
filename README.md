# Arquitetura Geral

A arquitetura da aplicação é composta por quatro pilares principais: controle de versão, construção de imagem, armazenamento de dados e execução em ambiente gerenciado. Cada componente possui uma função específica e interdependente para garantir reprodutibilidade, escalabilidade e simplicidade operacional.

## GitHub (código fonte)
O repositório GitHub centraliza toda a lógica da aplicação, incluindo:
- Código do dashboard,
- Pipeline de treinamento,
- Dockerfile,
- Dependências,
- Workflow de GitHub Actions.

O GitHub é o ponto inicial do fluxo, servindo como fonte única da verdade e acionador das automações.

## Docker Build local (Codespaces)
O processo de construção da imagem da aplicação é feito diretamente no Codespaces. Ele realiza:
- Instalação de dependências,
- Execução automática do script de treinamento,
- Geração dos artefatos de modelo,
- Empacotamento completo do ambiente dentro da imagem.

Esse build garante que toda atualização gere um modelo treinado e pronto para uso.

## Azure Container Registry (ACR)
Após o build, a imagem final é enviada para o Azure Container Registry.  
Ele armazena e versiona as imagens que serão consumidas pelo Container Apps.

Imagem principal:
- cestabasica:latest — utilizada em produção.

## Azure Blob Storage
O Blob Storage armazena dados e artefatos utilizados pelo dashboard.

Contêineres utilizados:
- **data/** — contém o dataset XLSX utilizado para treinamento e análise.
- **artifacts/** — contém modelo treinado, scaler e demais artefatos gerados pelo pipeline.

A comunicação ocorre via `BLOB_CONNECTION_STRING`, configurada no Container Apps como variável de ambiente.

## Azure Container Apps
O Container Apps consome:
- A imagem publicada no ACR,
- Os dados e artefatos do Blob Storage.

Ele executa o dashboard e permite escalonamento manual por meio de ajuste de réplicas (minReplicas).

---

# Estrutura do Repositório

O repositório segue uma organização clara e orientada ao fluxo de modelagem e deploy.

Diretórios e arquivos principais:

- **app.py** — código principal do dashboard, responsável por leitura de dados, carregamento de modelo, inferência e visualização.
- **train_model.py** — pipeline completo de preparação de dados, treinamento e upload dos artefatos.
- **Dockerfile** — define a imagem da aplicação, incluindo treinamento automático durante o build.
- **requirements.txt** — especifica dependências do projeto.
- **data/** — dataset local usado apenas durante o desenvolvimento.
- **artifacts/** — local onde ficam artefatos gerados quando o projeto é testado localmente.

---

# Tecnologias Utilizadas

A solução integra diversas ferramentas e serviços:

- Python 3.11  
- Dash e Plotly para visualização  
- Scikit-Learn para modelagem  
- Pandas e NumPy para análise de dados  
- Docker para empacotamento  
- Azure CLI para operações de nuvem  
- Azure Container Registry para armazenamento de imagens  
- Azure Blob Storage para dados e artefatos  
- Azure Container Apps para execução em produção  
- GitHub Actions para automação (CI)

---

# Deploy Manual — Fluxo Oficial

O deploy manual segue um fluxo padronizado, refletindo os passos utilizados em desenvolvimento e operação.

1. Preparação do ambiente com Azure CLI.  
2. Construção da imagem Docker localmente.  
3. Envio da imagem para o Azure Container Registry.  
4. Atualização do Container Apps para utilizar a nova imagem.

Esse fluxo garante que qualquer alteração do projeto resulte em uma nova versão funcional com modelo atualizado.

---

# Operação do Dashboard

A operação da aplicação é inteiramente baseada em ajuste de réplicas dentro do Container Apps:

- Quando minReplicas = 1 → dashboard ativo  
- Quando minReplicas = 0 → dashboard desligado  

Esse mecanismo substitui os comandos tradicionais de start/stop e reduz custos operacionais.

---

# Armazenamento na Azure

A aplicação depende de dois contêineres de Blob Storage:

### data/
- Armazena datasets XLSX
- Utilizado tanto para treino quanto para carregamento no dashboard

### artifacts/
- Contém arquivos como:
  - model.pkl  
  - scaler  
  - métricas, logs ou artefatos adicionais  
- Atualizado automaticamente durante o build

A aplicação utiliza sempre a versão mais recente desses arquivos.

---

# Treinamento Automático no Build

O processo de treinamento ocorre durante a construção da imagem Docker:

1. O script `train_model.py` lê o dataset (local ou do Blob).  
2. O modelo é treinado.  
3. Os artefatos são gerados localmente.  
4. Os artefatos são automaticamente enviados ao contêiner `artifacts/` do Blob Storage.

Isso garante consistência total entre modelo, imagem e dashboard.

---

# Dashboard Final

O dashboard executa:

- Carregamento do dataset atualizado do Blob Storage  
- Carregamento do modelo treinado  
- Aplicação do modelo sobre novos dados  
- Visualização completa dos resultados  
- Execução 100% no Azure Container Apps  
- Atualização automática sempre que a imagem é atualizada no ACR

---

# GitHub Actions — Descrição do Workflow

O workflow existente no repositório segue quatro grandes etapas:

## Etapa 1 — Setup do Ambiente
- Checkout do repositório  
- Instalação do Python  
- Instalação das dependências  
- Validação das importações  
Isso garante que o ambiente está íntegro antes de prosseguir.

## Etapa 2 — Pipeline de Treino
- Execução do pipeline de treinamento real (`train_model.py`)  
- Geração dos artefatos  
- Upload dos artefatos como histórico dentro do próprio GitHub Actions  
Essa etapa garante que sempre exista um registro do modelo treinado.

## Etapa 3 — Build da Imagem Docker
- Build completo da imagem  
- Execução de teste rápido (smoke test) para validar a imagem  
- Publicação do histórico do Docker como artefato  
Permite auditoria e transparência do processo.

## Etapa 4 — Finalização
- Apenas uma confirmação textual  
- Indica sucesso completo da pipeline

O fluxo não publica a imagem no ACR automaticamente, pois você optou por manter o deploy manual. Porém, toda a base CI está completamente funcional e validada.

---

# Conclusão

Este projeto integra:
- Pipeline completo de dados e modelagem,  
- Build automatizado e treinado,  
- Armazenamento estruturado na Azure,  
- Dashboard executado em Container Apps,  
- Automatização via GitHub Actions,  
- Infraestrutura robusta e simples de operar.

O fluxo garante reprodutibilidade, estabilidade e controle total sobre cada etapa do ciclo de vida da aplicação.

