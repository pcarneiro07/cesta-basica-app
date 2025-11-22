========================================
1. INSTALAR / ATIVAR AZURE CLI NO CODESPACE
========================================

# Atualizar lista de pacotes
sudo apt-get update

# Instalar dependências (necessário para WSL/Codespaces)
sudo apt-get install -y ca-certificates curl apt-transport-https lsb-release gnupg

# Baixar chave do repositório da Microsoft
curl -sL https://packages.microsoft.com/keys/microsoft.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/microsoft.gpg

# Adicionar repositório do Azure CLI
AZ_REPO=$(lsb_release -cs)
echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $AZ_REPO main" | sudo tee /etc/apt/sources.list.d/azure-cli.list

# Instalar Azure CLI
sudo apt-get update
sudo apt-get install -y azure-cli

# Verificar instalação
az version


========================================
2. LOGIN NA AZURE
========================================

az login --use-device-code


========================================
3. LIGAR O DASH (CONTAINER APP)
========================================

# Definir o número mínimo de réplicas como 1
az containerapp update \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  --set template.scale.minReplicas=1

# (Opcional) garantir que a revisão mais recente receba 100% do tráfego
LATEST=$(az containerapp revision list \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  --query "[?properties.active].name | [-1]" \
  -o tsv)

az containerapp ingress traffic set \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  --revision-weight ${LATEST}=100


========================================
4. DESLIGAR O DASH (CONTAINER APP)
========================================

az containerapp update \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  --set template.scale.minReplicas=0


========================================
5. VERIFICAR O ESTADO DAS RÉPLICAS
========================================

az containerapp revision list \
  --name dashboard-cesta-basica \
  --resource-group rg-cesta-basica \
  -o table

# Se todas as revisões estiverem Replicas=0 → APP DESLIGADO
# Se existir uma revisão com Replicas=1 → APP LIGADO
