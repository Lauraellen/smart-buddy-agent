# 🔍 Azure AI Search - Smart Buddy

Índice de busca semântica que alimenta o Agent com dados de alunos.

---

## 📋 Índice

1. [Visão Geral](#-visão-geral)
2. [Configuração](#-configuração)
3. [Schema do Índice](#-schema-do-índice)
4. [Evidências](#-evidências)

---

## 📋 Visão Geral

### O que é o AI Search

O **Azure AI Search** é o mecanismo de busca que:

- 📊 **Indexa dados dos alunos** - Sincronizado com SQL Database
- 🔍 **Permite buscas complexas** - Filtros OData, full-text search
- ⚡ **Resposta rápida** - Queries em milissegundos
- 🤖 **Integrado ao Agent** - Tool `openiaSmartBuddy` usa este índice
- 🔄 **Atualização automática** - Indexer sincroniza periodicamente

### Por que AI Search?

**Vantagens sobre consultas SQL diretas:**

✅ **Busca semântica** - Entende sinônimos e contexto  
✅ **Performance superior** - Índices otimizados para busca  
✅ **Filtros flexíveis** - OData syntax poderosa  
✅ **Integração nativa com Agent** - Tool oficial do Azure AI  
✅ **Escalabilidade** - Milhões de documentos sem degradação

### Arquitetura do Search

```
┌─────────────────────────────────────────────────────────┐
│              Azure SQL Database                         │
│  Tabela: alunos_dados_completos (materializada)            │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│          AI Search Indexer (automático)                 │
│  Schedule: A cada 5 minutos                             │
│  Sincroniza dados SQL → AI Search                       │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│        AI Search Index: gym-members-index               │
│  Documentos: ~200 alunos                                │
│  Campos: nome, telefone, categoria_frequencia, etc.     │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│       Agent Tool: openiaSmartBuddy                      │
│  Query: $filter=aniversariante_hoje eq true             │
└─────────────────────────────────────────────────────────┘
```

---

## 🔧 Configuração

### Data Source (SQL)

**Conexão com o banco de dados:**

| Propriedade | Valor |
|-------------|-------|
| **Tipo** | Azure SQL Database |
| **Server** | `sql-academia.database.windows.net` |
| **Database** | `academia-db` |
| **Tabela/View** | `alunos_dados_completos` (tabela materializada) |
| **Autenticação** | SQL Authentication |
| **Change Detection** | High Water Mark (campo `data_atualizacao`) |
| **Deletion Detection** | Soft Delete (campo `status = 'Inativo'`) |


### Index Schema

**Estrutura do índice:**

```json
{
  "name": "index-gym-members",
  "fields": [
    {
      "name": "id",
      "type": "Edm.String",
      "key": true,
      "searchable": false
    },
    {
      "name": "nome",
      "type": "Edm.String",
      "searchable": true,
      "filterable": true,
      "sortable": true
    },
    {
      "name": "telefone",
      "type": "Edm.String",
      "searchable": false,
      "filterable": false
    },
    {
      "name": "email",
      "type": "Edm.String",
      "searchable": true,
      "filterable": false
    },
    {
      "name": "data_nascimento",
      "type": "Edm.DateTimeOffset",
      "filterable": true,
      "sortable": true
    },
    {
      "name": "categoria_frequencia",
      "type": "Edm.String",
      "filterable": true,
      "facetable": true
    },
    {
      "name": "aniversariante_hoje",
      "type": "Edm.Boolean",
      "filterable": true
    },
    {
      "name": "status",
      "type": "Edm.String",
      "filterable": true
    },
    {
      "name": "data_atualizacao",
      "type": "Edm.DateTimeOffset",
      "filterable": true
    }
  ]
}
```

### Filtros OData Comuns

**Exemplos de queries usadas pelo Agent:**

```odata
# Aniversariantes ativos
aniversariante_hoje eq true and status eq 'Ativo'

# Baixa frequência
categoria_frequencia eq 'Baixa frequência' and status eq 'Ativo'

# Dedicados
categoria_frequencia eq 'Dedicado' and status eq 'Ativo'

# Check-ins > 20 no mês
checkins_ultimo_mes gt 20 and status eq 'Ativo'

# Aniversário em faixa de datas
data_nascimento ge 1990-01-01T00:00:00Z and data_nascimento le 2000-12-31T23:59:59Z
```

**Operadores OData:**

| Operador | Significado | Exemplo |
|----------|-------------|----------|
| `eq` | Igual | `status eq 'Ativo'` |
| `ne` | Diferente | `categoria_frequencia ne 'Baixa frequência'` |
| `gt` | Maior que | `checkins_ultimo_mes gt 20` |
| `ge` | Maior ou igual | `checkins_ultimo_mes ge 15` |
| `lt` | Menor que | `checkins_ultimo_mes lt 5` |
| `le` | Menor ou igual | `checkins_ultimo_mes le 10` |
| `and` | E lógico | `status eq 'Ativo' and aniversariante_hoje eq true` |
| `or` | Ou lógico | `categoria_frequencia eq 'Dedicado' or categoria_frequencia eq 'Regular'` |

---

## 📸 Evidências

### Search Service Overview

![Search Overview](/docs/prints/02-search/search-overview.png)

- Nome do serviço: `srch-gym-buddy-agent`
- Tier: Free
- Região: Central US
- Status: Running ✅
- Índices: 1
- Indexers: 1

---

### Index Schema

![Index Schema](/docs/prints/02-search/05-search-index-overview.png)

**Campos configurados:**
- `id` (key)
- `nome` (searchable, filterable)
- `telefone`
- `categoria_frequencia` (filterable)
- `aniversariante_hoje` (filterable)
- `checkins_ultimo_mes` (filterable, sortable)

---

### Data Source Configuration

![Data Source](/docs/prints/02-search/06-search-data.png)

**Conexão SQL:**
- Tipo: Azure SQL Database
- Server: `sql-academia.database.windows.net`
- Database: `academia-db`
- Tabela: `alunos_dados_completos`
---

### Indexer Status

![Indexer Status](/docs/prints/02-search/07-search-indexer-status.png)

**Status da sincronização:**
- Nome: `indexer-gym-buddy`
- Status: Success ✅
- Erros: 0
---

## 📚 Próximos Passos

- **[← Voltar ao README principal](../README.md)**
- **[→ Ver Agent](AGENT.md)**
- **[→ Ver Database](DATABASE.md)**
---

**🔗 Links Úteis:**

- 📖 [Azure AI Search Docs](https://learn.microsoft.com/azure/search/)
- 🎓 [OData Filter Syntax](https://learn.microsoft.com/azure/search/search-query-odata-filter)
- 🔧 [Indexer Best Practices](https://learn.microsoft.com/azure/search/search-indexer-overview)
- 💰 [Pricing Calculator](https://azure.microsoft.com/pricing/details/search/)