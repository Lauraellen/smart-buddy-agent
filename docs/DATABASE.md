# 📊 Estrutura do Banco de Dados - Smart Buddy

Documentação completa do schema, queries e otimizações do banco de dados.

---

## 📋 Índice

1. [Visão Geral](#-visão-geral)
2. [Schema Completo](#-schema-completo)
3. [Tabela Materializada](#-tabela-materializada)
4. [Evidências](#-evidências)

---

## 🎯 Visão Geral

### Estrutura

- **2 tabelas principais:** `alunos`, `checkins`
- **1 tabela materializada:** `alunos_dados_completos`
- **Relacionamento:** 1:N (aluno → checkins)

### Propósito

- `alunos`: Dados cadastrais
- `checkins`: Log de frequência
- `alunos_dados_completos`: Dados agregados para AI Search

---

## 📐 Schema Completo

### Tabela: alunos

```sql
CREATE TABLE alunos (
    id INT PRIMARY KEY IDENTITY(1,1),
    nome NVARCHAR(100) NOT NULL,
    telefone VARCHAR(15) NOT NULL,
    email NVARCHAR(100),
    data_nascimento DATE NOT NULL,
    objetivo NVARCHAR(50),
    plano NVARCHAR(20),
    ativo BIT DEFAULT 1,
    data_cadastro DATETIME DEFAULT GETDATE(),
    
    -- Índices
    INDEX idx_ativo (ativo),
    INDEX idx_data_nascimento (data_nascimento)
);
```

**Campos:**
- `id`: Chave primária auto-increment
- `nome`: Nome completo do aluno
- `telefone`: Formato: 11999999999 (DDD + número)
- `email`: Email (opcional)
- `data_nascimento`: Usado para calcular aniversariantes
- `objetivo`: Ex: "Emagrecimento", "Hipertrofia"
- `plano`: Ex: "Mensal", "Trimestral"
- `ativo`: Se está ativo na academia
- `data_cadastro`: Timestamp de criação

---

### Tabela: checkins

```sql
CREATE TABLE checkins (
    id INT PRIMARY KEY IDENTITY(1,1),
    aluno_id INT NOT NULL,
    data_checkin DATETIME NOT NULL DEFAULT GETDATE(),
    
    -- Foreign key
    CONSTRAINT FK_checkins_alunos 
        FOREIGN KEY (aluno_id) 
        REFERENCES alunos(id)
        ON DELETE CASCADE,
    
    -- Índices
    INDEX idx_aluno_data (aluno_id, data_checkin DESC),
    INDEX idx_data_checkin (data_checkin)
);
```

**Campos:**
- `id`: Chave primária
- `aluno_id`: Referência ao aluno
- `data_checkin`: Data/hora do check-in

**Índices:**
- `idx_aluno_data`: Otimiza buscas por aluno + data
- `idx_data_checkin`: Otimiza filtros por período

---

## 🔄 Tabela Materializada

### alunos_dados_completos

**Por que materializada?**
- ✅ Performance: Cálculos feitos uma vez
- ✅ AI Search: Sincronização mais rápida
- ✅ Queries: Sem JOINs complexos em tempo real

**Atualização:**
- Manualmente (após inserts/updates)
- Via Job agendado (diário)
- Via Trigger (futuro)

```sql
CREATE TABLE alunos_dados_completos (
    id INT PRIMARY KEY,
    nome NVARCHAR(100),
    telefone NVARCHAR(20),
    email NVARCHAR(100),
    data_nascimento DATE,
    objetivo NVARCHAR(50),
    plano NVARCHAR(20),
    ativo BIT,
    
    -- Campos calculados
    idade INT,
    aniversario_mes_dia NVARCHAR(5),  -- Ex: "11-19"
    aniversariante_hoje BIT,
    
    checkins_ultimo_mes INT,
    checkins_total INT,
    categoria_frequencia NVARCHAR(20), -- 'Dedicado', 'Regular', 'Irregular'
    
    ultimo_checkin DATETIME,
    dias_desde_ultimo_checkin INT,
    
    data_atualizacao DATETIME DEFAULT GETDATE()
);
```

---

### Script de Atualização

```sql
-- Limpar dados antigos
TRUNCATE TABLE alunos_dados_completos;

-- Popular com dados atualizados
INSERT INTO alunos_dados_completos (
    id, nome, telefone, email, data_nascimento, 
    objetivo, plano, ativo,
    idade, aniversario_mes_dia, aniversariante_hoje,
    checkins_ultimo_mes, checkins_total, categoria_frequencia,
    ultimo_checkin, dias_desde_ultimo_checkin,
    data_atualizacao
)
SELECT 
    a.id,
    a.nome,
    a.telefone,
    a.email,
    a.data_nascimento,
    a.objetivo,
    a.plano,
    a.ativo,
    
    -- Idade
    DATEDIFF(YEAR, a.data_nascimento, GETDATE()) as idade,
    
    -- Aniversário
    FORMAT(a.data_nascimento, 'MM-dd') as aniversario_mes_dia,
    CAST(CASE 
        WHEN FORMAT(a.data_nascimento, 'MM-dd') = FORMAT(GETDATE(), 'MM-dd') 
        THEN 1 ELSE 0 
    END AS BIT) as aniversariante_hoje,
    
    -- Check-ins últimos 30 dias
    (SELECT COUNT(*) 
     FROM checkins c 
     WHERE c.aluno_id = a.id 
       AND c.data_checkin >= DATEADD(DAY, -30, GETDATE())
    ) as checkins_ultimo_mes,
    
    -- Total de check-ins
    (SELECT COUNT(*) 
     FROM checkins c 
     WHERE c.aluno_id = a.id
    ) as checkins_total,
    
    -- Categoria de frequência
    CASE 
        WHEN (SELECT COUNT(*) FROM checkins c 
              WHERE c.aluno_id = a.id 
                AND c.data_checkin >= DATEADD(DAY, -30, GETDATE())) >= 20 
        THEN 'Dedicado'
        
        WHEN (SELECT COUNT(*) FROM checkins c 
              WHERE c.aluno_id = a.id 
                AND c.data_checkin >= DATEADD(DAY, -30, GETDATE())) >= 10 
        THEN 'Regular'
        
        ELSE 'Irregular'
    END as categoria_frequencia,
    
    -- Último check-in
    (SELECT MAX(c.data_checkin) 
     FROM checkins c 
     WHERE c.aluno_id = a.id
    ) as ultimo_checkin,
    
    -- Dias desde último check-in
    DATEDIFF(DAY, 
        (SELECT MAX(c.data_checkin) FROM checkins c WHERE c.aluno_id = a.id),
        GETDATE()
    ) as dias_desde_ultimo_checkin,
    
    -- Data de atualização
    GETDATE() as data_atualizacao

FROM alunos a;
```

**Executar diariamente:**
```sql
-- Via SQL Server Agent Job
-- Schedule: Diário às 08:30 (antes da Function rodar às 09:00)
```

---

## 📸 Evidências

### Database Structure

![SQL Structure](/docs/prints/03-database/09-sql-structure.png)

**O que mostra:**
- Tabelas: `alunos`, `checkins`, `alunos_dados_completos`
- Relacionamentos (Foreign Keys)
- Índices configurados
- Object Explorer do Azure Data Studio/SSMS

---

### Sample Data

![SQL Sample Data](/docs/prints/03-database/10-sql-sample-data.png)

**Query executada:**
```sql
SELECT TOP 10 
    nome, 
    telefone, 
    categoria_frequencia, 
    checkins_ultimo_mes,
    aniversariante_hoje
FROM alunos_dados_completos
WHERE ativo = 1;
```

**O que mostra:**
- Dados reais de alunos
- Categorias de frequência calculadas
- Check-ins do último mês
- Flag de aniversariante

---

## 📚 Próximos Passos

- [Voltar à Arquitetura →](ARCHITECTURE.md)
- [Voltar ao Setup →](SETUP.md)

---

**⬅️ [Voltar ao README principal](../README.md)**
