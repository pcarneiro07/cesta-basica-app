# Dashboard Cesta Básica — Operações Essenciais (Azure Container Apps)

Este documento descreve **somente os comandos realmente necessários** para operar o dashboard no Azure, incluindo:

- Instalar Azure CLI no Codespace  
- Autenticar  
- Ligar o dashboard  
- Desligar corretamente  
- Verificar o status  

---

## 1. Instalar Azure CLI (se o Codespace abrir sem `az`)
Execute apenas se o comando `az` retornar “command not found”:

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
Verificar:

bash
Copiar código
az version
2. Autenticar no Azure
bash
Copiar código
az login --use-device-code
Confirmar assinatura ativa:

bash
Copiar código
az account show
3. Verificar as revisões e estado do Container App
bash
Copiar código
az containerapp revision list \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  -o table
4. Ligar o Dashboard (subir 1 réplica)
bash
Copiar código
az containerapp update \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  --set template.scale.minReplicas=1
Confirmar:

bash
Copiar código
az containerapp revision list \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  -o table
Se Replicas = 1, o dashboard está online.

Acessar:

arduino
Copiar código
https://dashboard-cesta-basica.livelyisland-1ffcbd75.brazilsouth.azurecontainerapps.io/
5. Desligar o Dashboard completamente
5.1. Deixar o app no modo de revisão única (evita criar revisões novas ao ligar/desligar)
bash
Copiar código
az containerapp revision set-mode \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  --mode single
5.2. Definir réplica mínima como zero
bash
Copiar código
az containerapp update \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  --set template.scale.minReplicas=0
5.3. Desativar a revisão ativa (ESTE é o passo que desliga de verdade)
Primeiro, veja quais revisões existem:

bash
Copiar código
az containerapp revision list \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  -o table
Depois, escolha a revisão com Replicas = 1 e desative:

bash
Copiar código
az containerapp revision deactivate \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  --revision <NOME_DA_REVISAO>
5.4. Verificar que está realmente desligado
bash
Copiar código
az containerapp revision list \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  -o table
O dashboard está desligado quando:

ini
Copiar código
Replicas = 0
E o link passa a retornar Error 404 — App Stopped.

6. Resumo rápido (cola para uso diário)
Ligar
bash
Copiar código
az containerapp update -n dashboard-cesta-basica -g rg-cesta-basica --set template.scale.minReplicas=1
Desligar
bash
Copiar código
az containerapp revision set-mode -n dashboard-cesta-basica -g rg-cesta-basica --mode single
az containerapp update -n dashboard-cesta-basica -g rg-cesta-basica --set template.scale.minReplicas=0
az containerapp revision deactivate -n dashboard-cesta-basica -g rg-cesta-basica --revision <REVISION>
Status
bash
Copiar código
az containerapp revision list -n dashboard-cesta-basica -g rg-cesta-basica -o table
7. URL do Dashboard
arduino
Copiar código
https://dashboard-cesta-basica.livelyisland-1ffcbd75.brazilsouth.azurecontainerapps.io/
