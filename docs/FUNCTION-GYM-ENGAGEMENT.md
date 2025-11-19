# ⚡ Azure Function - Smart Buddy

Orquestrador principal do sistema de engajamento automático.

---
## 📋 Visão Geral
### O que faz

A Azure Function é o **cérebro orquestrador** do Smart Buddy. Ela:

- ⏰ **Executa automaticamente** todos os dias às 6h da manhã
- 🤖 **Coordena o Agent** para buscar alunos (aniversariantes, baixa frequência, dedicados)
- 📱 **Aciona o Logic App** para envio de mensagens via Twilio
- 📊 **Gera relatórios** de execução com estatísticas
- 🔍 **Registra logs** no Application Insights para monitoramento


## 🔧 Configuração
### Timer Trigger (Cron Expression)

**Schedule configurado:**

```python
@app.schedule(schedule="0 0 9 * * *", 
              arg_name="myTimer", 
              use_monitor=True)
def EngajamentoDiarioAutomatico(myTimer: func.TimerRequest):
    logging.info('Iniciando engajamento diário automático...')
```

**⚠️ IMPORTANTE:**

- Timezone: **UTC** (horário de Brasília = UTC-3)
- 9h UTC = 6h Brasília (horário de verão pode variar)
- Ajustar para: `"0 0 12 * * *"` para 9h Brasília

### Application Settings

**Variáveis de ambiente configuradas:**

```python
# Azure AI Agent
AZURE_OPENAI_ENDPOINT = "https://smartgym-aiservices.openai.azure.com/"
AZURE_OPENAI_API_KEY = "***************************"
AGENT_ID = "asst_******************"

# Logic App (Twilio)
LOGIC_APP_URL = "https://prod-**.logic.azure.com:443/workflows/***"
```

**Como configurar no Azure Portal:**

1. Function App → Configuration → Application settings
2. Adicionar cada variável
3. ⚠️ **Nunca** commitar chaves no código
4. Usar **Azure Key Vault** em produção (recomendado)

**Configuração local (local.settings.json):**

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "python",
    "AZURE_OPENAI_ENDPOINT": "https://...",
    "AZURE_OPENAI_API_KEY": "***",
    "AGENT_ID": "asst_***",
    "LOGIC_APP_URL": "https://..."
  }
}
```

### Código Fonte

**📂 Repositório GitHub:**

👉 **[Ver código completo da Function →](https://github.com/Lauraellen/gym-engagement-function)**

**Estrutura do projeto:**

```
SmartBuddyFunction/
├── function_app.py           # Timer + HTTP triggers
├── requirements.txt          # Dependências Python
├── host.json                 # Configuração runtime
└── local.settings.json       # Variáveis locais (não commitar!)
```

## 📸 Evidências
### Function Overview
![Function Overview](/docs/prints/05-function/17-function-overview.png)


- gym-engagement-function
- Runtime: Python 3.12
- Hosting Plan: Consumo Flexível
- Região: Central US
- Status: Running ✅

**Funções configuradas:**
- ⏰ `EngajamentoDiarioAutomatico` - Timer trigger
- 🔧 `EngajamentoDiarioManual` - HTTP trigger
- 💚 `TestarConfiguracao` - Health check

---
### Timer Configuration
![Timer Config](/docs/prints/05-function/19-function-timer-config.png)

- Schedule: `0 0 9 * * *`
- Next execution: [próxima data/hora]
- Timezone: UTC
- Use monitor: Enabled ✅
---

### Application Settings

![App Settings](/docs/prints/05-function/20-function-app-settings.png)

**⚠️ ATENÇÃO:** Credenciais foram ocultadas

**Variáveis configuradas:**
- `PROJECT_ENDPOINT`
- `AGENT_API_KEY`
- `AGENT_ID`
- `LOGIC_APP_TWILIO_URL`
- `AGENT_ENDPOINT`
---
