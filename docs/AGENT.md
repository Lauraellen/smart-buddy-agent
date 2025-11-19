# 🤖 Azure AI Agent - Smart Buddy

Assistente inteligente que busca e analisa dados de alunos usando linguagem natural.

---

## 📋 Índice

1. [Visão Geral](#-visão-geral)
2. [Configuração](#-configuração)
3. [Exemplos de Uso](#-exemplos-de-uso)
4. [Evidências](#-evidências)

---

## 📋 Visão Geral

### O que é o Agent

O **Azure AI Agent** é um assistente inteligente baseado em **GPT-4o Mini** que:

- 🔍 **Entende perguntas em linguagem natural** - "Quem são os aniversariantes de hoje?"
- 🤖 **Busca dados automaticamente** - Usa ferramenta integrada ao Azure AI Search
- 📊 **Retorna JSON estruturado** - Dados prontos para envio de mensagens
- 💡 **Interpreta contexto** - Entende "baixa frequência", "dedicados", "aniversariantes"
- ⚡ **Resposta rápida** - ~2-3 segundos por consulta


### Arquitetura do Agent

```
┌─────────────────────────────────────────────────────────┐
│                    Azure Function                       │
│  Pergunta: "Liste os aniversariantes de hoje"          │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              Azure AI Agent (GPT-4o Mini)               │
│  System Instructions: "Use a ferramenta para buscar..." │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│         Tool: openiaSmartBuddy (AI Search)              │
│  Filter: "aniversariante_hoje eq true"                  │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│            Azure AI Search (gym-members-index)          │
│  Retorna: [{"nome": "Maria", "telefone": "3598..."}, ...]│
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                  Agent retorna JSON                     │
│  [{"nome": "Maria", "telefone": "35999999999"}, ...]    │
└─────────────────────────────────────────────────────────┘
```

---

## 🔧 Configuração

### System Instructions (Resumo)

**Principais regras configuradas:**

✅ **Ferramentas disponíveis:**
- `openiaSmartBuddy` - Busca alunos no Azure AI Search com filtros OData
- `smartbuddy_Tool` - Envia mensagens WhatsApp (máx 280 caracteres)

✅ **Campos de busca:**
- **SEARCHABLE** (use `search`): nome, email, telefone, objetivo, plano
- **FILTERABLE** (use `filter`): ativo, aniversariante_hoje, checkins_ultimo_mes, categoria_frequencia

✅ **Operadores OData:**
- `eq` (igual), `ne` (diferente), `gt` (maior), `ge` (maior/igual), `lt` (menor), `le` (menor/igual)
- `and` (E lógico), `or` (OU lógico)

✅ **Categorias de frequência:**
- **Dedicado:** ≥20 check-ins/mês
- **Regular:** 10-19 check-ins/mês  
- **Irregular:** <10 check-ins/mês

✅ **Tarefas automáticas:**
- Mensagens de aniversário personalizadas por categoria
- Cupons de recompensa para alunos dedicados
- Mensagens motivacionais com dados reais (nome, idade, objetivo)

✅ **Regras importantes:**
- Sempre usar ferramentas (nunca inventar dados)
- Personalizar mensagens com dados reais
- Confirmar antes de envios em massa
- Retornar resumos estruturados

**📄 [Ver prompt completo do Agent →](agent-instructions.md)**

### Tools Configuration

**O Agent usa 2 ferramentas integradas:**

| Tool | Componente Azure | Função |
|------|------------------|--------|
| `openiaSmartBuddy` | **[Azure AI Search →](SEARCH.md)** | Busca alunos com filtros OData |
| `smartbuddy_Tool` | **[Logic App + Twilio →](LOGIC-APP.md)** | Envia mensagens WhatsApp (máx 280 chars) |

**Como funciona:**

1. **Agent recebe pergunta** em linguagem natural
   - Exemplo: *"Envie mensagens de aniversário para os dedicados"*

2. **Agent decide quais tools usar:**
   - `openiaSmartBuddy` → Buscar aniversariantes com `filter: "aniversariante_hoje eq true and categoria_frequencia eq 'Dedicado'"`
   - `smartbuddy_Tool` → Enviar mensagem personalizada para cada aluno

3. **Tools executam ações:**
   - AI Search retorna lista de alunos
   - Logic App envia mensagens via Twilio

4. **Agent processa resultados:**
   - Formata resposta estruturada
   - Retorna resumo: "✅ 5 mensagens enviadas (Dedicados)"




## 💬 Exemplos de Uso

### Exemplo 1: Buscar Aniversariantes

**Pergunta:**
```
Liste todos os aniversariantes de hoje que estão ativos
```

**O que o Agent faz:**

1. ✅ Identifica: precisa buscar aniversariantes
2. ✅ Chama tool `openiaSmartBuddy`
3. ✅ Aplica filtro: `aniversariante_hoje eq true and status eq 'Ativo'`
4. ✅ Retorna JSON

**Resposta:**

```json
[
  {
    "nome": "Maria Silva",
    "telefone": "35999999999"
  },
  {
    "nome": "João Santos",
    "telefone": "35988888888"
  }
]
```

**Print da execução:**

![Playground Aniversariantes](/docs/prints/01-agent/04a-playground-aniversariantes.png)

---

### Exemplo 2: Buscar Baixa Frequência

**Pergunta:**
```
Quem está com baixa frequência e ativo?
```

**Filtro aplicado pelo Agent:**
```odata
categoria_frequencia eq 'Baixa frequência' and status eq 'Ativo'
```

**Resposta:**

```json
[
  {
    "nome": "Pedro Oliveira",
    "telefone": "35999887766"
  },
  {
    "nome": "Ana Costa",
    "telefone": "35988776655"
  }
]
```

**Print da execução:**

![Playground Baixa Frequência](/docs/prints/01-agent/04b-playground-baixa-freq.png)

---

### Exemplo 3: Buscar Alunos Dedicados

**Pergunta:**
```
Mostre os alunos dedicados (mais de 20 check-ins no mês)
```

**Filtro aplicado:**
```odata
categoria_frequencia eq 'Dedicado' and status eq 'Ativo'
```

**Resposta:**

```json
[
  {
    "nome": "Carlos Mendes",
    "telefone": "35977665544"
  },
  {
    "nome": "Julia Fernandes",
    "telefone": "35966554433"
  },
  {
    "nome": "Lucas Almeida",
    "telefone": "35955443322"
  }
]
```

**Print da execução:**

![Playground Dedicados](/docs/prints/01-agent/04c-playground-dedicados.png)

---

## 📱 Resultado Real - Mensagem Enviada

### Mensagem de Aniversário (WhatsApp)

![SMS Aniversário](/docs/prints/01-agent/04c-sms-aniversario.png)

**O que mostra:**
- ✅ Mensagem personalizada enviada via Twilio
- ✅ Nome do aluno inserido corretamente
- ✅ Formatação com emojis preservada
- ✅ Texto motivacional completo
- ✅ Recebida no WhatsApp do aluno

**Prova de que o sistema funciona end-to-end:**
1. Agent busca aniversariantes → `openiaSmartBuddy`
2. Dados retornados do AI Search
3. Mensagem personalizada gerada
4. Enviada via Logic App + Twilio
5. **Recebida no WhatsApp real** ✅

---

## 📸 Evidências

### Agent Overview

![Agent Overview](/docs/prints/01-agent/01-agent-overview.png)

**O que mostra:**
- Nome do Agent: `SmartBuddyAgent`
- Modelo: GPT-4o Mini
- Status: Deployed ✅
- ID do Agent: `asst_******************`

---

### Tools Configuradas no Agent

#### Tool 1: openiaSmartBuddy (Busca de Dados)

**O que é:** Azure Function HTTP que executa queries OData no Azure AI Search

- **Nome:** `openiaSmartBuddy`
- **Tipo:** Azure Function (HTTP Trigger)
- **Função:** Busca alunos com filtros OData no índice gym-members-index
- **Endpoint:** `https://func-gym-engagement.azurewebsites.net/api/openiaSmartBuddy`

**📄 [Ver documentação completa →](FUNCTION-OPENIA-SMARTBUDDY.md)**

![Tool Config](/docs/prints/01-agent/action-openia-smart-buddy.png)

---

#### Tool 2: smartbuddy_Tool (Envio de Mensagens)

**O que é:** Logic App que envia mensagens WhatsApp/SMS via Twilio

- **Nome:** `smartbuddy_Tool`
- **Tipo:** Logic App (HTTP Trigger)
- **Função:** Envia mensagens WhatsApp (máx 280 caracteres)
- **Endpoint:** Logic App URL

**📄 [Ver documentação completa →](LOGIC-APP.md)**

![Tool Config](/docs/prints/01-agent/logicapp-twilio-config.png)

---

### Knowledge (AI Search Index)

**O Agent NÃO usa Knowledge diretamente.** Em vez disso:

✅ **Acessa AI Search via tool** `openiaSmartBuddy` (Azure Function)  
✅ **Dados vêm do índice** `gym-members-index`  
✅ **Filtros OData** aplicados pela tool

**Por que não é "Knowledge"?**

| Knowledge (RAG) | AI Search via Tool (este projeto) |
|-----------------|-------------------------------------|
| Documentos (PDFs, textos) | Dados estruturados (campos, tipos) |
| Agent busca automaticamente | Agent chama tool explicitamente |
| Embedding vectors | Filtros OData |
| Para contexto geral | Para queries específicas |

**📄 [Ver documentação do AI Search →](SEARCH.md)**

---

## 📚 Próximos Passos

- **[← Voltar ao README principal](../README.md)**
- **[→ Configurar AI Search](SEARCH.md)**
- **[→ Ver Function Orquestradora](FUNCTION-GYM-ENGAGEMENT.md)**
- **[→ Ver Tool openiaSmartBuddy](FUNCTION-OPENIA-SMARTBUDDY.md)**

---

**🔗 Links Úteis:**
- 📖 [Azure AI Agent SDK Docs](https://learn.microsoft.com/azure/ai-services/agents/)
- 🎓 [GPT-4o Mini Overview](https://learn.microsoft.com/azure/ai-services/openai/concepts/models#gpt-4o-mini)
- 💰 [OpenAI Pricing](https://azure.microsoft.com/pricing/details/cognitive-services/openai-service/)