# 📱 Logic App + Twilio - Smart Buddy

Orquestração de envio de mensagens via SMS/WhatsApp.

---

## 📋 Índice

1. [Visão Geral](#-visão-geral)
2. [Configuração](#-configuração)
3. [Workflow](#-workflow)
4. [Evidências](#-evidências)
---

## 📋 Visão Geral

### O que é o Logic App

O **Azure Logic App** é o orquestrador de mensagens que:

- 📞 **Recebe requisições HTTP** - Trigger da Azure Function
- 🔄 **Formata payload** - Adiciona prefixo `+55` ao telefone
- 📱 **Envia via Twilio** - SMS ou WhatsApp
- 📊 **Registra logs** - Histórico de execuções
- ⚡ **Serverless** - Pay-per-execution


### Arquitetura do Logic App

```
┌─────────────────────────────────────────────────────────┐
│              Azure Function                             │
│  POST https://logic-app-url.azure.com                   │
│  Body: {"telefone": "35999999999", "mensagem": "..."}   │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│          Logic App: HTTP Trigger                        │
│  Recebe: {"telefone": "35999999999", "mensagem": ...}   │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│          Action: Compose (Formatar)                     │
│  Adiciona prefixo: "+55" + telefone                      │
│  Resultado: "+35999999999"                             │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│          Action: Twilio - Send SMS                      │
│  From: +1234567890 (número Twilio)                      │
│  To: +35999999999                                      │
│  Body: "🎉 Parabéns Maria! Hoje é seu dia especial!"    │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                  Twilio API                             │
│  Envia SMS/WhatsApp para o telefone                    │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│            Response: 200 OK                             │
│  {"sid": "SM...", "status": "queued"}                   │
└─────────────────────────────────────────────────────────┘
```

---

## 🔧 Configuração

### HTTP Trigger

**Endpoint configurado:**

```
POST https://prod-03.swedencentral.logic.azure.com:443/workflows/0f44adb3bd514841a14690ffb1cee01b/triggers/When_a_HTTP_request_is_received/paths/invoke?api-version=2016-10-01&sp=%2Ftriggers%2FWhen_a_HTTP_request_is_received%2Frun&sv=1.0&sig=***
```


**Request Body esperado:**

```json
{
  "telefone": "35999999999",
  "mensagem": "🎉 Parabéns Maria! Hoje é seu dia especial!"
}
```

**Schema do Trigger:**

```json
{
  "type": "object",
  "properties": {
    "telefone": {
      "type": "string",
      "description": "Telefone sem o +55 (DDD + número)"
    },
    "mensagem": {
      "type": "string",
      "description": "Mensagem personalizada para o aluno"
    }
  },
  "required": ["telefone", "mensagem"]
}
```

### Twilio Configuration

**Credenciais necessárias:**

| Propriedade | Valor | Onde encontrar |
|-------------|-------|----------------|
| **Account SID** | `AC******************` | Twilio Console → Account Info |
| **Auth Token** | `******************` | Twilio Console → Account Info |
| **From Number** | `+1234567890` | Twilio Console → Phone Numbers |
| **Messaging Service SID** | `MG******************` | (Opcional) Para SMS |

**Configuração no Logic App:**

1. Designer → Add action → Twilio
2. Connection name: `twilio-connection`
3. Account SID: `AC...`
4. Auth Token: `***`
5. Save connection

### Action: Compose (Formatar Telefone)

**Expressão configurada:**

```json
concat('+55', triggerBody()?['telefone'])
```

**Exemplo:**

| Input | Output |
|-------|--------|
| `35999999999` | `+5535999999999` |
| `35999999999` | `+5535999999999` |

**Por que adicionar +55:**

- ✅ Twilio exige formato **E.164** (padrão internacional)
- ✅ `+` obrigatório
- ✅ `55` = código do Brasil
- ✅ Sem `+55` → Erro 21211 (Invalid 'To' Phone Number)

### Action: Twilio - Send Message

**Configuração:**

```json
{
  "From": "+1234567890",
  "To": "@{outputs('Compose_Telefone')}",
  "Body": "@{triggerBody()?['mensagem']}"
}
```

**Campos dinâmicos:**

- **From:** Número Twilio (fixo)
- **To:** Resultado da action Compose (`+5535999999999`)
- **Body:** Mensagem do trigger (personalizada por aluno)

---

## 🔄 Workflow
## 🔄 Workflow

### Execução Exemplo

**Input (da Function):**

```json
{
  "telefone": "35999999999",
  "mensagem": "🎉 Parabéns Maria Silva! Hoje é seu dia especial! 🎂"
}
```

**Step 1 - Compose:**

```json
{
  "telefoneFormatado": "+5535999999999"
}
```

**Step 2 - Twilio Send:**

```json
{
  "from": "+1234567890",
  "to": "+5535999999999",
  "body": "🎉 Parabéns Maria Silva! Hoje é seu dia especial! 🎂"
}
```

**Output (para Function):**

```json
{
  "sid": "SM1234567890abcdef",
  "status": "queued",
  "dateCreated": "2024-11-19T09:00:15Z"
}
```
---

## 📸 Evidências

### Logic App Overview

![Logic App Overview](/docs/prints/04-logic-app/logic-app-overview.png)


- Nome: `smart-buddy`
- Status: Enabled ✅
- Região: Sweden Central
- Trigger: HTTP Request
---

### Workflow Designer

![Workflow Designer](/docs/prints/04-logic-app/13-logicapp-trigger-schema.png)

**Fluxo visual:**
1. ⚡ HTTP Trigger (manual)
2. 🔄 Compose (formatar telefone)
3. 📱 Twilio - Send Message
4. ✅ Response (200 OK)


**Configuração visível Twilio:**
- Connection name: `HTTP`
- Status: Connected ✅
- From number: `+17609735353`
- URI: `https://api.twilio.com/2010-04-01/Accounts/[CHAVE-TWILIO]/Messages.json`

---

### Runs History - Success

![Run Details - Success](/docs/prints/04-logic-app/logs-mensagens-twilio.png)


**Detalhes da execução:**
- Step 1: HTTP Trigger ✅
- Step 2: Compose ✅
- Step 3: Twilio Send ✅ (Status: Delivered)
- Total time: 2.3s

---

### SMS - Success

![SMS](/docs/prints/04-logic-app/mensagens-recebidas-twilio.png)

---

## 📚 Próximos Passos
## 📚 Próximos Passos

- **[← Voltar ao README principal](../README.md)**
- **[→ Ver Function Orquestradora](FUNCTION-GYM-ENGAGEMENT.md)**
- **[→ Ver Agent](AGENT.md)**

---

**🔗 Links Úteis:**

- 📖 [Azure Logic Apps Docs](https://learn.microsoft.com/azure/logic-apps/)
- 🎓 [Twilio SMS API](https://www.twilio.com/docs/sms/api)
- 🔧 [Twilio Error Codes](https://www.twilio.com/docs/api/errors)
- 💰 [Twilio Pricing](https://www.twilio.com/sms/pricing/br)