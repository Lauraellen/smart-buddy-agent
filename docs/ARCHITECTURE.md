# 🏗️ Arquitetura e Componentes - Smart Buddy

Documentação técnica detalhada da arquitetura do sistema.

---

## 📋 Índice

1. [Visão Geral](#-visão-geral)
2. [Diagrama de Arquitetura](#-diagrama-de-arquitetura)
3. [Componentes Detalhados](#-componentes-detalhados)
4. [Fluxo de Dados](#-fluxo-de-dados)
5. [Integrações](#-integrações)
6. [Segurança](#-segurança)

---

## 🎯 Visão Geral

O Smart Buddy é uma solução **serverless** que combina:
- **Azure Functions** (orquestração)
- **Azure AI Agent** (inteligência)
- **Azure AI Search** (busca otimizada)
- **Azure SQL** (persistência)
- **Twilio** (comunicação)

### Padrões Arquiteturais

- ✅ **Serverless** - Sem gerenciamento de infraestrutura
- ✅ **Event-Driven** - Timer trigger diário
- ✅ **Microservices** - Componentes independentes
- ✅ **API-First** - Comunicação via HTTP/REST

---

## 📊 Diagrama de Arquitetura

![Arquitetura Completa](/docs/prints/08-architecture/arq-agente.drawio.png)

### Fluxo Principal

```
Timer (9h) → Function → Agent → AI Search → SQL Database
                ↓
          Logic App → Twilio → SMS
```

## 🔧 Componentes Detalhados

### 1️⃣ Azure Function (Orquestrador)

**Responsabilidades:**
- Executar diariamente às 6h
- Coordenar busca de alunos (via Agent)
- Enviar mensagens (via Logic App)
- Gerar relatório de execução
- Logging e monitoramento

**Tecnologias:**
- **Runtime:** Python 3.12
- **Hosting:** Consumption Plan (serverless)
- **Trigger:** Timer (cron expression)
- **Output:** HTTP (Logic App)

**Funções:**
- `EngajamentoDiarioAutomatico` - Timer trigger (9h)
- `EngajamentoDiarioManual` - HTTP trigger (testes)
- `TestarConfiguracao` - Health check

**📄 [Ver documentação completa →](FUNCTION-GYM-ENGAGEMENT.md)**

---

### 2️⃣ Azure AI Agent (Smart Buddy)

**Responsabilidades:**
- Interpretar perguntas em linguagem natural
- Traduzir para queries OData
- Executar busca no AI Search
- Retornar resultados estruturados (JSON)

**Tecnologias:**
- **Modelo:** GPT-4o Mini (rápido e econômico)
- **Platform:** Azure AI Foundry
- **Tool:** Azure AI Search connector

**Capacidades:**
- Entende linguagem natural
- Aplica filtros complexos
- Retorna JSON estruturado
- Context-aware (mantém contexto da conversa)

**Exemplos de Interação:**

**Input (humano):**
```
"Liste todos os aniversariantes de hoje que estão ativos"
```

**Processamento (Agent):**
- Identifica intent: buscar aniversariantes
- Aplica filtros: `aniversariante_hoje eq true and ativo eq true`
- Chama tool `openiaSmartBuddy`

**Output (JSON):**
```json
[
  {
    "nome": "Maria Santos",
    "telefone": "35999999999",
    "aniversariante_hoje": true
  }
]
```

**Tools Configuradas:**
- `openiaSmartBuddy` - Busca no AI Search com filtros OData
- `smartbuddy_Tool` - Envio de mensagens via Logic App

**📄 [Ver documentação completa →](AGENT.md)**

---

### 3️⃣ Azure AI Search

**Responsabilidades:**
- Indexar dados do SQL Database
- Executar buscas full-text
- Aplicar filtros OData
- Retornar resultados ranqueados

**Configuração:**

**Configuração:**
- **Index:** `gym-members-index`
- **Campos principais:** id, nome, telefone, categoria_frequencia, aniversariante_hoje
- **Data Source:** Azure SQL Database (tabela materializada)
- **Sync:** Indexer a cada 5 minutos
- **Filtros:** OData (eq, ne, gt, and, or)

**📄 [Ver documentação completa →](SEARCH.md)**

---

### 4️⃣ Azure SQL Database

**Responsabilidades:**
- Armazenar dados de alunos
- Registrar check-ins
- Calcular métricas de frequência
- Fornecer dados para o Search

**Estrutura:**
- **Tabelas:** `alunos`, `checkins`
- **Tabela materializada:** `vw_alunos_completos` (otimizada para AI Search)
- **Sync:** Indexer sincroniza a cada 5 minutos
- **Métricas:** categoria_frequencia, aniversariante_hoje calculadas automaticamente

**📄 [Ver documentação completa →](DATABASE.md)**

---

### 5️⃣ Logic App (Twilio SMS)

**Responsabilidades:**
- Receber requisições HTTP da Function
- Adicionar código do país (+55)
- Enviar SMS via Twilio
- Retornar status

**Workflow:**
1. Recebe HTTP POST com telefone e mensagem
2. Adiciona código do país (+55)
3. Envia via Twilio API
4. Retorna status (200 OK)

**📄 [Ver documentação completa →](LOGIC-APP.md)**

---

### 6️⃣ Application Insights

**Responsabilidades:**
- Coletar logs da Function
- Métricas de performance
- Alertas de falha
- Dashboards de monitoramento

**Métricas Coletadas:**
- Execuções diárias
- Duração média
- Taxa de sucesso/falha
- Mensagens enviadas
- Erros e exceções

---

## 🔄 Fluxo de Dados Simplificado

### Execução Diária (6h da manhã)

**1. Timer Dispara (06:00)**
- Azure Function inicia automaticamente
- Timezone: America/Sao_Paulo (UTC-3)

**2. Busca de Alunos (06:00-06:01)**
- Function envia pergunta em linguagem natural para Agent
- Agent traduz para filtro OData
- AI Search executa query no índice
- Retorna JSON com dados dos alunos

**Exemplos de filtros:**
- Aniversariantes: `aniversariante_hoje eq true and status eq 'Ativo'`
- Baixa frequência: `categoria_frequencia eq 'Irregular' and status eq 'Ativo'`
- Dedicados: `categoria_frequencia eq 'Dedicado' and status eq 'Ativo'`

**3. Envio de Mensagens (06:01-06:02)**
- Function monta mensagem personalizada para cada aluno
- Envia HTTP POST para Logic App
- Logic App adiciona +55 e envia via Twilio
- SMS entregue ao aluno

**4. Relatório Final (06:02)**
```json
{
  "data_execucao": "19/11/2025",
  "aniversariantes": {"total": 3, "enviados": 3},
  "baixa_frequencia": {"total": 12, "enviados": 12},
  "dedicados": {"total": 8, "enviados": 8},
  "total_mensagens": 23,
  "duracao": "2min 15s"
}
```

---

## 🔗 Integrações

### Function → AI Agent
- **Protocol:** HTTPS
- **Auth:** Azure Identity (Managed Identity)
- **Format:** JSON
- **SDK:** `azure-ai-projects`

### AI Agent → AI Search
- **Protocol:** HTTPS
- **Auth:** API Key
- **Format:** OData + JSON
- **Tool:** Azure AI Search connector

### AI Search → SQL Database
- **Protocol:** TDS (Tabular Data Stream)
- **Auth:** SQL Authentication
- **Sync:** Indexer (pull-based)
- **Frequency:** Every 1 hour

### Function → Logic App
- **Protocol:** HTTPS (POST)
- **Auth:** None (public URL)
- **Format:** JSON
- **Timeout:** 30s

### Logic App → Twilio
- **Protocol:** HTTPS
- **Auth:** Account SID + Auth Token
- **Format:** Form-encoded
- **API:** Twilio REST API v2010

---

## 📦 Recursos Azure

### Resource Group Overview

![Resource Group](/docs/prints/08-architecture/28-resource-group-all.png)

**Recursos provisionados:**
- ✅ Azure Function (func-gym-engagement)
- ✅ AI Search (srch-gym-buddy)
- ✅ SQL Database (db-smartgym)
- ✅ Logic App (logic-app-twilio-sms)
- ✅ AI Services (laura-mhvbzzym-swedencentral)
- ✅ Storage Account (stgymengagement)
- ✅ Application Insights

---

## 💰 Custos

### Cost Analysis

![Cost Analysis](/docs/prints/08-architecture/29-cost-analysis.png)

**Breakdown mensal estimado:**
- Azure Function (Consumption): ~R$ 0 (free tier)
- AI Search (Free): R$ 0
- SQL Database (Basic): ~R$ 30
- Logic App (Consumption): ~R$ 5
- AI Services (GPT-4o Mini): ~R$ 10
- Twilio (trial): R$ 0
- **Total estimado: ~R$ 45/mês**

---

## 🔒 Segurança

### Autenticação e Autorização

**Azure Function:**
- Managed Identity (para chamar AI Agent)
- Application Settings (credenciais criptografadas)

**AI Agent:**
- API Key rotation
- RBAC (Role-Based Access Control)

**SQL Database:**
- SQL Authentication
- Firewall rules (apenas Azure services)
- SSL/TLS obrigatório

**AI Search:**
- API Key (admin + query keys)
- CORS configurado

**Logic App:**
- Trigger URL (SAS token)
- IP restrictions (opcional)

**Twilio:**
- Account SID + Auth Token
- Webhook validation (futuro)

### Dados Sensíveis

**Application Settings (criptografadas):**
- PROJECT_ENDPOINT
- AGENT_ID
- LOGIC_APP_TWILIO_URL
- Twilio SID/Token (no Logic App)

**Não versionados:**
- `local.settings.json` (no .gitignore)
- Credenciais de banco
- API keys

### Boas Práticas

✅ **Secrets em Application Settings** (não no código)  
✅ **Managed Identity** para comunicação entre serviços Azure  
✅ **Firewall rules** no SQL Database  
✅ **SSL/TLS** obrigatório em todas as conexões  
✅ **API Key rotation** periódica  
✅ **Least privilege** (permissões mínimas necessárias)  

---

## 📚 Documentação Relacionada

- **[🤖 Azure AI Agent](AGENT.md)** - Configuração e uso do Agent
- **[🔍 Azure AI Search](SEARCH.md)** - Índice e queries OData
- **[⚡ Function Orquestradora](FUNCTION-GYM-ENGAGEMENT.md)** - Timer trigger
- **[🔧 Function openiaSmartBuddy](FUNCTION-OPENIA-SMARTBUDDY.md)** - Tool do Agent
- **[📱 Logic App + Twilio](LOGIC-APP.md)** - Envio de mensagens
- **[📊 Database](DATABASE.md)** - Schema e queries SQL
- **[🔧 Setup](SETUP.md)** - Instalação do zero

---

**⬅️ [Voltar ao README principal](../README.md)**
