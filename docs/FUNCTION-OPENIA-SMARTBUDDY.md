# 🔍 Azure Function - openiaSmartBuddy

Function que executa queries OData no Azure AI Search. É a **tool** usada pelo Agent para buscar dados de alunos.

---

## 📋 Índice

1. [Visão Geral](#-visão-geral)
2. [Configuração](#-configuração)
3. [Como Funciona](#-como-funciona)
4. [Exemplos de Uso](#-exemplos-de-uso)
5. [Evidências](#-evidências)

---

## 📋 Visão Geral

### O que é

A **openiaSmartBuddy** é uma **Azure Function HTTP** que:

- 🔗 **Conecta o Agent ao AI Search** - Interface entre GPT-4o Mini e índice de dados
- 🔍 **Executa queries OData** - Busca filtrada no índice gym-members-index
- 📊 **Retorna JSON estruturado** - Dados prontos para processamento pelo Agent
- ⚡ **Performance otimizada** - Cache e timeout configurados

### Por que é necessária?

O Azure AI Agent **não acessa diretamente** o AI Search. Precisa de uma **tool** (função) para isso.

**Fluxo:**

```
Agent GPT-4o Mini
    ↓
  Chama tool: openiaSmartBuddy
    ↓
  Function HTTP (esta)
    ↓
  Azure AI Search (gym-members-index)
    ↓
  Retorna JSON → Agent
```

## 🔧 Configuração

### HTTP Trigger

**Método:** POST  
**URL:** `https://func-gym-search-gmdgc3bugkgxdqhv.centralus-01.azurewebsites.net/api/search?code=******`  
**Autenticação:** Function Key (level: function)

**Headers necessários:**

```http
Content-Type: application/json
x-functions-key: [FUNCTION_KEY]
```

### Request Body

**Formato esperado:**

```json
{
  "search": "*",
  "filter": "aniversariante_hoje eq true and status eq 'Ativo'",
  "top": 100,
  "count": true
}
```

**Parâmetros:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `search` | string | Não | Busca full-text (use `*` para todos) |
| `filter` | string | Não | Filtro OData (ex: `categoria_frequencia eq 'Dedicado'`) |
| `top` | int | Não | Limite de resultados (padrão: 100) |
| `count` | bool | Não | Incluir total de registros (padrão: true) |

### Response

**Sucesso (200 OK):**

```json
{
  "@odata.count": 3,
  "value": [
    {
      "id": "123",
      "nome": "Maria Silva",
      "telefone": "35999999999",
      "email": "maria@example.com",
      "categoria_frequencia": "Dedicado",
      "checkins_mes": 22,
      "aniversariante_hoje": true,
      "status": "Ativo"
    },
    {
      "id": "456",
      "nome": "João Santos",
      "telefone": "35988888888",
      "email": "joao@example.com",
      "categoria_frequencia": "Regular",
      "checkins_mes": 15,
      "aniversariante_hoje": true,
      "status": "Ativo"
    }
  ]
}
```

**Erro (400 Bad Request):**

```json
{
  "error": "Filtro OData inválido",
  "details": "Syntax error at position 10"
}
```

**Erro (500 Internal Server Error):**

```json
{
  "error": "Erro ao conectar no AI Search",
  "details": "Connection timeout"
}
```

### Application Settings

**Variáveis de ambiente necessárias:**

```bash
SEARCH_ENDPOINT="https://srch-gym-buddy.search.windows.net"
SEARCH_API_KEY="***************************"
SEARCH_INDEX_NAME="index-gym-members"
```

**Como configurar:**

1. Portal Azure → Function App → Configuration
2. Application settings → New application setting
3. Adicionar as 3 variáveis acima
4. Save → Restart function

### Código Fonte

**📂 Repositório GitHub:**

👉 **[Ver código completo →](https://github.com/Lauraellen/func-gym-search)**
---

## 🔄 Como Funciona

### Fluxo Completo (passo a passo)

**1. Agent recebe pergunta do usuário:**

```
Usuário (via Function principal): "Liste os aniversariantes de hoje"
   ↓
Agent GPT-4o Mini: Identifica que precisa buscar dados
```

**2. Agent decide usar a tool:**

```json
{
  "tool_name": "openiaSmartBuddy",
  "parameters": {
    "search": "*",
    "filter": "aniversariante_hoje eq true and status eq 'Ativo'",
    "top": 100
  }
}
```

**3. Agent chama a Function HTTP:**

```http
POST https://func-gym-engagement.azurewebsites.net/api/openiaSmartBuddy
Content-Type: application/json
x-functions-key: ***

{
  "search": "*",
  "filter": "aniversariante_hoje eq true and status eq 'Ativo'",
  "top": 100,
  "count": true
}
```

**4. Function executa query no AI Search:**

```python
search_client.search(
    search_text="*",
    filter="aniversariante_hoje eq true and status eq 'Ativo'",
    top=100
)
```

**5. AI Search retorna documentos:**

```json
{
  "@odata.count": 3,
  "value": [
    {"nome": "Maria", "telefone": "35999999999", ...},
    {"nome": "João", "telefone": "35988888888", ...},
    {"nome": "Ana", "telefone": "35977777777", ...}
  ]
}
```

**6. Function retorna JSON para Agent:**

Agent recebe os dados e processa.

**7. Agent formata resposta final:**

```json
[
  {"nome": "Maria Silva", "telefone": "35999999999"},
  {"nome": "João Santos", "telefone": "35988888888"},
  {"nome": "Ana Costa", "telefone": "35977777777"}
]
```

### Tempo de Execução

```
Chamada HTTP → Function: ~50ms
Function → AI Search: ~100ms
AI Search processa query: ~200ms
Retorno completo: ~350ms
```

Total: **~350ms** (bem rápido! ⚡)

---

## 💬 Exemplos de Uso

### Exemplo 1: Buscar Aniversariantes

**Request:**

```bash
curl -X POST "https://func-gym-engagement.azurewebsites.net/api/openiaSmartBuddy" \
  -H "Content-Type: application/json" \
  -H "x-functions-key: YOUR_KEY" \
  -d '{
    "search": "*",
    "filter": "aniversariante_hoje eq true and status eq '\''Ativo'\''",
    "top": 100,
    "count": true
  }'
```

**Response:**

```json
{
  "@odata.count": 3,
  "value": [
    {
      "id": "1",
      "nome": "Maria Silva",
      "telefone": "35999999999",
      "aniversariante_hoje": true,
      "categoria_frequencia": "Dedicado",
      "checkins_mes": 22
    },
    {
      "id": "2",
      "nome": "João Santos",
      "telefone": "35988888888",
      "aniversariante_hoje": true,
      "categoria_frequencia": "Regular",
      "checkins_mes": 15
    },
    {
      "id": "3",
      "nome": "Ana Costa",
      "telefone": "35999887766",
      "aniversariante_hoje": true,
      "categoria_frequencia": "Irregular",
      "checkins_mes": 8
    }
  ]
}
```

---

### Exemplo 2: Buscar Baixa Frequência

**Request:**

```json
{
  "search": "*",
  "filter": "categoria_frequencia eq 'Irregular' and status eq 'Ativo'",
  "top": 50
}
```

**Response:**

```json
{
  "@odata.count": 12,
  "value": [
    {
      "nome": "Pedro Oliveira",
      "telefone": "35977665544",
      "categoria_frequencia": "Irregular",
      "checkins_mes": 6
    },
    {
      "nome": "Carla Mendes",
      "telefone": "35966554433",
      "categoria_frequencia": "Irregular",
      "checkins_mes": 4
    }
    // ... mais 10 registros
  ]
}
```

---

### Exemplo 3: Buscar Dedicados (>= 20 check-ins)

**Request:**

```json
{
  "search": "*",
  "filter": "categoria_frequencia eq 'Dedicado' and status eq 'Ativo'",
  "top": 100
}
```

**Response:**

```json
{
  "@odata.count": 8,
  "value": [
    {
      "nome": "Carlos Mendes",
      "telefone": "35955443322",
      "categoria_frequencia": "Dedicado",
      "checkins_mes": 25
    },
    {
      "nome": "Julia Fernandes",
      "telefone": "35944332211",
      "categoria_frequencia": "Dedicado",
      "checkins_mes": 22
    }
    // ... mais 6 registros
  ]
}
```

---

## 📸 Evidências

### Function Overview

![Function Overview](/docs/prints/09-function-openia/overview.png)

**O que mostra:**
- Function App name: `func-gym-search`
- Runtime: Python 3.12
- Hosting Plan: Flex Consumption
- Região: Central US
- Status: Running ✅

---

### Application Settings

![Application Settings](/docs/prints/09-function-openia/application-settings.png)

**Variáveis configuradas:**
- `SEARCH_ENDPOINT` - Endpoint do Azure AI Search
- `SEARCH_API_KEY` - API Key para autenticação
- `FUNCTIONS_WORKER_RUNTIME` - python

---

### Thread/Execution Test

![Thread Execution](/docs/prints/09-function-openia/thread-openia.png)

**O que mostra:**
- Execução da function sendo chamada pelo Agent
- Request com filtro OData
- Response com JSON de alunos
- Tempo de execução
- Status: Success ✅

---

## 📚 Próximos Passos

- **[← Voltar ao README principal](../README.md)**
- **[→ Ver Agent](AGENT.md)** (quem chama esta function)
- **[→ Ver AI Search](SEARCH.md)** (de onde vêm os dados)
- **[→ Ver Function Principal](FUNCTION-GYM-ENGAGEMENT.md)** (orquestrador)

---

**🔗 Links Úteis:**

- 📖 [Azure Functions HTTP Trigger Docs](https://learn.microsoft.com/azure/azure-functions/functions-bindings-http-webhook-trigger)
- 🔍 [Azure AI Search REST API](https://learn.microsoft.com/rest/api/searchservice/)
- 🎓 [OData Filter Syntax](https://learn.microsoft.com/azure/search/search-query-odata-filter)
- 💡 [Function Keys Management](https://learn.microsoft.com/azure/azure-functions/function-keys-how-to)
