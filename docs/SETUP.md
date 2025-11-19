# 🚀 Guia de Setup e Instalação - Smart Buddy

Este guia mostra como configurar todo o ambiente Azure do zero.

---

## 📋 Índice

1. [Pré-requisitos](#-pré-requisitos)
2. [Criar Resource Group](#-criar-resource-group)
3. [Configurar Azure SQL Database](#-configurar-azure-sql-database)
4. [Configurar Azure AI Search](#-configurar-azure-ai-search)
5. [Configurar Azure AI Foundry](#-configurar-azure-ai-foundry)
6. [Configurar Logic App + Twilio](#-configurar-logic-app--twilio)
7. [Criar Azure Function](#-criar-azure-function)
8. [Testes](#-testes)

---

## 📋 Pré-requisitos

### Contas Necessárias

- [ ] **Conta Azure** com créditos ativos
  - Se estudante: [Azure for Students](https://azure.microsoft.com/free/students/)
  - Trial gratuito: [Azure Free Trial](https://azure.microsoft.com/free/)

- [ ] **Azure AI Foundry** (AI Studio)
  - Acesse: https://ai.azure.com/
  - Use mesma conta Azure

- [ ] **Conta Twilio**
  - Trial: https://www.twilio.com/try-twilio
  - Créditos grátis: $15 USD

### Ferramentas Opcionais

- **Azure CLI** - Para criar recursos via linha de comando
  - Instalação: https://learn.microsoft.com/cli/azure/install-azure-cli
  - Alternativa: Usar Azure Portal (interface web)

---

## 🏗️ Criar Resource Group

### Via Azure Portal

1. Acesse: https://portal.azure.com/
2. Pesquise: "Resource groups"
3. Clique em **+ Create**
4. Preencha:
   - **Subscription:** Selecione sua assinatura
   - **Resource group:** `rg-smartgym-engajamento`
   - **Region:** Brazil South

5. **Review + create** → **Create**

### Via Azure CLI

```bash
# Login no Azure
az login

# Criar resource group
az group create \
  --name rg-smartgym-engajamento \
  --location brazilsouth

# Verificar
az group show --name rg-smartgym-engajamento
```

---

## 🗄️ Configurar Azure SQL Database

### 1. Criar SQL Server

**Via Portal:**
1. Pesquise: "SQL servers"
2. **+ Create**
3. Preencha:
   - **Server name:** `sql-smartgym` (deve ser único globalmente)
   - **Location:** Brazil South
   - **Authentication:** SQL authentication
   - **Admin login:** `sqladmin`
   - **Password:** `SuaSenhaSegura123!`

**Via CLI:**
```bash
az sql server create \
  --name sql-smartgym \
  --resource-group rg-smartgym-engajamento \
  --location brazilsouth \
  --admin-user sqladmin \
  --admin-password "SuaSenhaSegura123!"
```

### 2. Configurar Firewall

```bash
# Permitir serviços do Azure
az sql server firewall-rule create \
  --resource-group rg-smartgym-engajamento \
  --server sql-smartgym \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Permitir seu IP (para desenvolvimento)
az sql server firewall-rule create \
  --resource-group rg-smartgym-engajamento \
  --server sql-smartgym \
  --name AllowMyIP \
  --start-ip-address SEU_IP \
  --end-ip-address SEU_IP
```

### 3. Criar Database

```bash
az sql db create \
  --name db-smartgym \
  --server sql-smartgym \
  --resource-group rg-smartgym-engajamento \
  --service-objective Basic \
  --backup-storage-redundancy Local
```

### 4. Popular Banco de Dados

**Ver:** [📊 DATABASE.md](DATABASE.md) para scripts SQL completos de:
- Criação das tabelas `alunos` e `checkins`
- Criação da tabela materializada `alunos_dados_completos`
- Inserção de dados de exemplo

---

## 🔍 Configurar Azure AI Search

### 1. Criar Search Service

```bash
az search service create \
  --name srch-gym-buddy-agent \
  --resource-group rg-smartgym-engajamento \
  --location brazilsouth \
  --sku basic
```

### 2. Criar Data Source (SQL)

**Via Portal:**
1. Acesse: `srch-gym-buddy` → **Import data**
2. **Data Source:** Azure SQL Database
3. **Connection string:**
   ```
   Server=tcp:sql-smartgym.database.windows.net,1433;
   Initial Catalog=db-smartgym;
   User ID=sqladmin;
   Password=SuaSenhaSegura123!;
   ```
4. **Table/View:** `alunos_dados_completos`

### 3. Criar Index

**Campos importantes:**
- `id` (Key)
- `nome` (Searchable, Retrievable)
- `telefone` (Retrievable)
- `categoria_frequencia` (Filterable, Facetable)
- `aniversariante_hoje` (Filterable)
- `ativo` (Filterable)

### 4. Criar Indexer

**Schedule:** Every 1 hour
**Start time:** Immediate

---

## 🤖 Configurar Azure AI Foundry

### 1. Criar Project

1. Acesse: https://ai.azure.com/
2. **+ New project**
3. Nome: `smartgym-project`
4. **Create new hub** (se necessário)

### 2. Criar Agent

1. **Build** → **Agents** → **+ Create**
2. Configurações:
   - **Name:** Smart Buddy
   - **Model:** GPT-4o Mini
   - **System message:** 
     ```
     Você é um assistente especializado em buscar dados de alunos de academia.
     Use a ferramenta 'openiaSmartBuddy' para buscar informações no Azure AI Search.
     Sempre retorne os dados em formato JSON.
     ```

### 3. Adicionar Tool (Azure AI Search)

1. **Tools** → **+ Add tool**
2. **Type:** Azure AI Search
3. **Configuration:**
   - **Tool name:** `openiaSmartBuddy`
   - **Search service:** `srch-gym-buddy`
   - **Index:** `gym-members-index`
   - **API key:** [copiar do Portal]

---

## 📱 Configurar Logic App + Twilio

### 1. Criar Conta Twilio

1. https://www.twilio.com/try-twilio
2. Verificar celular
3. Obter número Twilio (trial: +1...)
4. Copiar credenciais:
   - **Account SID**
   - **Auth Token**

### 2. Criar Logic App

**Via Portal:**
1. Pesquise: "Logic apps"
2. **+ Add**
3. Configurações:
   - **Name:** `logic-app-twilio-sms`
   - **Type:** Consumption
   - **Region:** Brazil South

### 3. Configurar Workflow

**Trigger:**
- Type: **When a HTTP request is received**
- Method: **POST**
- Schema:
  ```json
  {
    "type": "object",
    "properties": {
      "telefone": {"type": "string"},
      "mensagem": {"type": "string"}
    }
  }
  ```

**Action:**
- Type: **Twilio - Send Message (SMS)**
- Connection:
  - Account SID: [seu SID]
  - Auth Token: [seu token]
- Configuration:
  - **From:** [número Twilio: +1...]
  - **To:** `concat('+55', triggerBody()?['telefone'])`
  - **Body:** `triggerBody()?['mensagem']`

### 4. Copiar URL do Trigger

Após salvar, copie a **HTTP POST URL**. Será usada na Function.

---

## ⚡ Criar Azure Function

### 1. Criar Storage Account

```bash
az storage account create \
  --name stgymengagement \
  --resource-group rg-smartgym-engajamento \
  --location brazilsouth \
  --sku Standard_LRS
```

### 2. Criar Function App

```bash
az functionapp create \
  --name func-gym-engagement \
  --resource-group rg-smartgym-engajamento \
  --consumption-plan-location brazilsouth \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --storage-account stgymengagement \
  --os-type Linux
```

### 3. Configurar Application Settings

```bash
az functionapp config appsettings set \
  --resource-group rg-smartgym-engajamento \
  --name func-gym-engagement \
  --settings \
    PROJECT_ENDPOINT="https://...ai.azure.com" \
    AGENT_ID="asst_..." \
    LOGIC_APP_TWILIO_URL="https://...logic.azure.com:443/..."
```

### 4. Deploy do Código

**Código fonte:** https://github.com/Lauraellen/gym-engagement-function

O deploy pode ser feito via:
- VS Code (Azure Functions Extension)
- Azure CLI (`func azure functionapp publish`)
- GitHub Actions (CI/CD)

**Ver:** [⚡ FUNCTION-GYM-ENGAGEMENT.md](FUNCTION-GYM-ENGAGEMENT.md) para detalhes da configuração.

---

## ✅ Testes

### 1. Testar Agent no Playground

1. AI Foundry → Agents → Smart Buddy → **Test**
2. Perguntas:
   ```
   Liste os aniversariantes de hoje
   Quem está com baixa frequência?
   Mostre os alunos dedicados
   ```

### 2. Testar Logic App

**Via Portal:**
1. Logic App → **Run Trigger**
2. Body:
   ```json
   {
     "telefone": "35999999999",
     "mensagem": "Teste de SMS 📱"
   }
   ```

### 3. Verificar SMS

Abra o celular e veja se recebeu a mensagem!

### 4. Verificar Execução da Function

**Via Portal:**
1. Function App → **Monitor**
2. Verificar logs de execução
3. Checar Application Insights para métricas

---

## 📚 Próximos Passos

- [Entender a Arquitetura →](ARCHITECTURE.md)
- [Estrutura do Banco →](DATABASE.md)
- [Configuração do Agent →](AGENT.md)
- [Detalhes da Function →](FUNCTION-GYM-ENGAGEMENT.md)

---

**⬅️ [Voltar ao README principal](../README.md)**
