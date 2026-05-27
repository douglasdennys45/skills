---
name: postgres-best-practices
description: Aplica boas práticas oficiais do PostgreSQL 18 ao modelar, escrever, revisar ou refatorar schema, queries, migrations e código de acesso a dados. Cobre convenções de nomenclatura em camelCase com aspas duplas, tipos recomendados (UUID, TIMESTAMPTZ, JSONB, BIGINT em centavos), primary keys, foreign keys, constraints nomeadas, índices (B-tree, GIN, BRIN, partial, compostos), soft delete, tabelas de junção N:N, migrations zero-downtime (golang-migrate), transactions (RunInTx, níveis de isolamento, eventos pós-commit), paginação por cursor, queries idiomáticas (SELECT explícito, INSERT RETURNING, UPSERT, SELECT FOR UPDATE/SKIP LOCKED), pool de conexões em Go e configuração de servidor (shared_buffers, work_mem, WAL, logging). Use sempre que criar/editar migrations SQL, modelar tabelas, escrever queries, implementar repositórios Go que falam com Postgres, definir índices, orquestrar transactions em use cases ou ajustar configuração do banco. Acione ao trabalhar com arquivos .sql, repositórios que usam database/sql/pgx, migrations em migrations/<app>/*, ou ao discutir performance/locks/isolation no PostgreSQL.
---

# PostgreSQL 18 Best Practices

Skill baseada nas práticas oficiais do PostgreSQL 18 e nos padrões internos do projeto para modelagem, queries, migrations, transactions e configuração. Todas as recomendações aqui devem ser seguidas ao criar, revisar ou refatorar qualquer artefato que envolva o banco.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer arquivo `.sql` (DDL ou DML)
- Ao escrever ou revisar migrations em `migrations/<app>/*`
- Ao implementar repositórios Go que falam com PostgreSQL (`database/sql`, `pgx`)
- Ao modelar novas tabelas, índices, constraints ou tipos
- Ao orquestrar transactions em use cases (multi-repo, multi-write)
- Ao definir paginação, locking, isolamento ou estratégia de soft delete
- Ao ajustar configuração do servidor (`postgresql.conf`, `docker-compose.yml`)
- Ao discutir performance, locks, race conditions ou planos de execução (`EXPLAIN ANALYZE`)

## 1. Convenções de Nomenclatura

| Elemento     | Convenção                       | Exemplo                                      |
|--------------|---------------------------------|----------------------------------------------|
| Tabela       | camelCase, **plural**           | `users`, `orderItems`, `auditLogs`           |
| Coluna       | camelCase                       | `createdAt`, `userId`, `firstName`           |
| Primary Key  | `id`                            | `id UUID PRIMARY KEY`                        |
| Foreign Key  | `<entidade>Id`                  | `userId`, `orderId`                          |
| Constraint   | `<tabela>_<descricao>`          | `users_emailUnique`, `orders_statusCheck`    |
| Index        | `idx_<tabela>_<colunas>`        | `idx_users_email`, `idx_orders_userId_createdAt` |
| Trigger      | `trg_<tabela>_<acao>`           | `trg_users_updatedAt`                        |
| Function     | camelCase descritivo            | `setUpdatedAt()`, `validateStatus()`         |
| Enum Type    | camelCase singular              | `orderStatus`, `userRole`                    |
| Sequence     | `<tabela>_<coluna>_seq`         | `users_id_seq`                               |

### Regras inegociáveis

- PostgreSQL preserva case **apenas** quando o nome está entre aspas duplas. Como usamos camelCase, **todas** as referências a tabelas e colunas devem usar aspas duplas no SQL.
- Tabelas sempre no **plural** — representam coleções de entidades.
- Foreign key sempre nomeada como `<entidadeSingular>Id`.
- Constraints sempre nomeadas **explicitamente** — nunca deixar o PostgreSQL gerar nomes automáticos.

### Aspas duplas — padrão obrigatório

```sql
-- DDL
CREATE TABLE "orderItems" (
    "id"        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "orderId"   UUID NOT NULL,
    "productId" UUID NOT NULL,
    "quantity"  INT  NOT NULL
);

-- DML
SELECT "id", "orderId", "quantity"
FROM "orderItems"
WHERE "orderId" = $1;
```

```go
// Go repository
const query = `
    INSERT INTO "users" ("name", "email", "createdAt", "updatedAt")
    VALUES ($1, $2, $3, $4)
    RETURNING "id"`
```

## 2. Padrão de Criação de Tabelas

### Template base

```sql
CREATE TABLE "users" (
    -- identidade
    "id"        UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- dados de domínio
    "name"      TEXT NOT NULL,
    "email"     TEXT NOT NULL,
    "role"      TEXT NOT NULL DEFAULT 'viewer',

    -- timestamps
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- constraints nomeadas
    CONSTRAINT "users_emailUnique" UNIQUE ("email"),
    CONSTRAINT "users_roleCheck"   CHECK ("role" IN ('admin', 'editor', 'viewer'))
);

CREATE TRIGGER "trg_users_updatedAt"
    BEFORE UPDATE ON "users"
    FOR EACH ROW
    EXECUTE FUNCTION "setUpdatedAt"();

CREATE INDEX "idx_users_email"     ON "users" ("email");
CREATE INDEX "idx_users_createdAt" ON "users" ("createdAt" DESC);
```

### Função global `setUpdatedAt`

Criar **uma única vez** na migration inicial:

```sql
-- migrations/<app>/000001_setup.up.sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE OR REPLACE FUNCTION "setUpdatedAt"()
RETURNS TRIGGER AS $$
BEGIN
    NEW."updatedAt" = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

Toda tabela com `"updatedAt"` deve registrar o trigger `"trg_<tabela>_updatedAt"`.

## 3. Tipos de Dados

### Recomendados

| Caso de Uso              | Tipo Recomendado            | Justificativa                                                                 |
|--------------------------|-----------------------------|------------------------------------------------------------------------------|
| Identificadores (PK/FK)  | `UUID`                      | Sem sequência previsível, facilita sharding, `gen_random_uuid()` no banco    |
| Texto curto              | `TEXT`                      | Sem limite artificial; validação no app                                       |
| Texto com limite rígido  | `VARCHAR(N)`                | Quando o limite faz parte da regra de negócio (ex: `VARCHAR(2)` para país)    |
| Inteiros                 | `INT` / `BIGINT`            | `INT` para contadores, `BIGINT` para IDs sequenciais ou valores grandes       |
| Dinheiro                 | `BIGINT` (centavos)         | Armazenar em centavos. Nunca `NUMERIC`/`DECIMAL` para dinheiro em APIs        |
| Booleanos                | `BOOLEAN`                   | Sempre `BOOLEAN`, nunca `INT` 0/1                                             |
| Data/hora                | `TIMESTAMPTZ`               | **Sempre** com timezone                                                       |
| Apenas data              | `DATE`                      | Para datas sem hora (ex: nascimento)                                          |
| JSON estruturado         | `JSONB`                     | Indexável; nunca `JSON`                                                       |
| Enumerações              | `TEXT` + `CHECK`            | Adicionar valor sem `ALTER TYPE` (lock exclusivo)                             |
| Arrays                   | `TEXT[]` / `UUID[]`         | Para listas pequenas; relações N:N usam tabela de junção                      |
| IP addresses             | `INET`                      | Tipo nativo                                                                   |
| Intervalos de tempo      | `INTERVAL`                  | Para durações                                                                 |

### Tipos a evitar

| Tipo                  | Problema                                              | Usar em vez                          |
|-----------------------|-------------------------------------------------------|--------------------------------------|
| `SERIAL` / `BIGSERIAL`| IDs previsíveis, problemas em sharding                 | `UUID DEFAULT gen_random_uuid()`     |
| `TIMESTAMP` (sem TZ)  | Ambiguidade entre regiões                              | `TIMESTAMPTZ`                        |
| `CHAR(N)`             | Padding com espaços                                    | `TEXT` ou `VARCHAR(N)`               |
| `MONEY`               | Locale-dependent, impreciso                            | `BIGINT` (centavos)                  |
| `JSON`                | Não indexável, reprocessa a cada leitura               | `JSONB`                              |
| `ENUM` type           | `ALTER TYPE` exige lock exclusivo                      | `TEXT` + `CHECK`                     |
| `FLOAT` / `DOUBLE`    | Imprecisão de ponto flutuante                          | `BIGINT` (centavos) ou `NUMERIC`     |

## 4. Primary Keys

```sql
"id" UUID PRIMARY KEY DEFAULT gen_random_uuid()
```

- Sempre `UUID` como primary key.
- `gen_random_uuid()` gera UUID v4 no banco (requer `pgcrypto`; nativo no PG 13+).
- Nunca usar `SERIAL` — evita previsibilidade e facilita replicação/sharding.
- Nunca gerar UUID no app — o **banco** é a fonte de verdade.
- Usar `INSERT ... RETURNING "id"` para obter o UUID gerado.

## 5. Foreign Keys

```sql
CREATE TABLE "orders" (
    "id"        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "userId"    UUID NOT NULL,
    "status"    TEXT NOT NULL DEFAULT 'pending',
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT "orders_userId_fk"
        FOREIGN KEY ("userId")
        REFERENCES "users" ("id")
        ON DELETE RESTRICT
        ON UPDATE CASCADE
);

CREATE INDEX "idx_orders_userId" ON "orders" ("userId");
```

- Toda FK **nomeada**: `CONSTRAINT "<tabela>_<coluna>_fk"`.
- **Sempre** criar índice na coluna FK — PostgreSQL não cria automaticamente.
- `ON DELETE`:
  - `RESTRICT` (padrão): impede exclusão do pai se há filhos. Mais seguro.
  - `CASCADE`: exclui filhos. Usar apenas quando faz sentido (tenant → recursos).
  - `SET NULL`: para relações opcionais.
- `ON UPDATE CASCADE`: defensivo, raro com UUID.

## 6. Constraints

```sql
-- UNIQUE simples
CONSTRAINT "users_emailUnique" UNIQUE ("email")

-- UNIQUE composto
CONSTRAINT "users_tenantId_email_unique" UNIQUE ("tenantId", "email")

-- CHECK enumeração (preferível a ENUM type)
CONSTRAINT "orders_statusCheck"
    CHECK ("status" IN ('pending', 'processing', 'completed', 'cancelled'))

-- CHECK range
CONSTRAINT "orders_amountPositive" CHECK ("amountCents" > 0)

-- CHECK formato
CONSTRAINT "users_emailFormat"
    CHECK ("email" ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
```

- Todas as constraints **nomeadas explicitamente**.
- Preferir `TEXT` + `CHECK` em vez de `CREATE TYPE ... AS ENUM`.
- `NOT NULL` como padrão. Usar `NULL` apenas quando a ausência tem significado de negócio.

## 7. Índices

### Quando criar

| Situação                          | Índice                                              |
|-----------------------------------|-----------------------------------------------------|
| Coluna em `WHERE` frequente       | `CREATE INDEX`                                      |
| Coluna em `ORDER BY`              | `CREATE INDEX ... DESC/ASC`                         |
| Coluna em `JOIN` (FK)             | `CREATE INDEX` (**obrigatório** em toda FK)         |
| Busca por prefixo em texto        | `CREATE INDEX ... USING gin (col gin_trgm_ops)`     |
| Filtro em subconjunto             | `CREATE INDEX ... WHERE` (partial)                  |
| Coluna JSONB consultada           | `CREATE INDEX ... USING gin (col)`                  |

### Tipos

```sql
-- B-tree (padrão, 90% dos casos)
CREATE INDEX "idx_users_email" ON "users" ("email");

-- Ordenação para paginação
CREATE INDEX "idx_users_createdAt" ON "users" ("createdAt" DESC);

-- Composto
CREATE INDEX "idx_orders_userId_status" ON "orders" ("userId", "status");

-- Partial
CREATE INDEX "idx_orders_pending" ON "orders" ("createdAt")
    WHERE "status" = 'pending';

-- Unique index
CREATE UNIQUE INDEX "idx_users_email_unique" ON "users" ("email");

-- GIN para JSONB
CREATE INDEX "idx_users_metadata" ON "users" USING gin ("metadata");

-- GIN para busca textual
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX "idx_users_name_trgm" ON "users" USING gin ("name" gin_trgm_ops);

-- BRIN para tabelas grandes ordenadas por insert (logs)
CREATE INDEX "idx_auditLogs_createdAt" ON "auditLogs" USING brin ("createdAt");
```

### Índices compostos — ordem das colunas

A ordem **importa**: índice em `(a, b, c)` cobre `WHERE a` e `WHERE a AND b`, mas **não** cobre `WHERE b`.

**Regra da seletividade:** colunas mais seletivas (mais valores distintos) primeiro.

```sql
-- Query: WHERE "tenantId" = $1 AND "status" = $2 ORDER BY "createdAt" DESC
CREATE INDEX "idx_orders_tenantId_status_createdAt"
    ON "orders" ("tenantId", "status", "createdAt" DESC);
```

### Regras

- **Sempre** indexar colunas de FK.
- **Nunca** criar índice em coluna de baixa cardinalidade isolada (ex: `BOOLEAN`). Usar partial.
- **Sempre** validar com `EXPLAIN ANALYZE`.
- **Nunca** criar índices especulativos — apenas para queries existentes.
- Em tabelas grandes em produção, usar `CREATE INDEX CONCURRENTLY` para evitar lock.

## 8. Soft Delete

```sql
CREATE TABLE "users" (
    "id"        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    "name"      TEXT NOT NULL,
    "email"     TEXT NOT NULL,
    "createdAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt" TIMESTAMPTZ NOT NULL DEFAULT now(),
    "deletedAt" TIMESTAMPTZ  -- NULL = ativo
);

-- Unique parcial: email único apenas entre ativos
CREATE UNIQUE INDEX "idx_users_email_active"
    ON "users" ("email")
    WHERE "deletedAt" IS NULL;

-- Index parcial para queries que filtram ativos
CREATE INDEX "idx_users_active"
    ON "users" ("createdAt" DESC)
    WHERE "deletedAt" IS NULL;
```

- `"deletedAt" TIMESTAMPTZ` sem `NOT NULL` — `NULL` = ativo.
- Unique constraints devem ser **partial** com `WHERE "deletedAt" IS NULL`.
- Toda query de leitura deve incluir `WHERE "deletedAt" IS NULL`.
- Repositório expõe método `SoftDelete` que faz `UPDATE ... SET "deletedAt" = now()`.

## 9. Tabelas de Junção (N:N)

```sql
CREATE TABLE "userBusinessUnits" (
    "userId"         UUID NOT NULL,
    "businessUnitId" UUID NOT NULL,
    "assignedAt"     TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT "userBusinessUnits_pk" PRIMARY KEY ("userId", "businessUnitId"),

    CONSTRAINT "userBusinessUnits_userId_fk"
        FOREIGN KEY ("userId") REFERENCES "users" ("id") ON DELETE CASCADE,

    CONSTRAINT "userBusinessUnits_businessUnitId_fk"
        FOREIGN KEY ("businessUnitId") REFERENCES "businessUnits" ("id") ON DELETE CASCADE
);

-- Índice reverso (PG só cria índice para o primeiro campo da PK)
CREATE INDEX "idx_userBusinessUnits_businessUnitId"
    ON "userBusinessUnits" ("businessUnitId");
```

- Nome: `<entidadeA><entidadeB>` em camelCase plural.
- PK composta com as duas FKs.
- `ON DELETE CASCADE` em ambas.
- Sempre criar **índice no segundo campo** da PK.

## 10. Migrations

### Estrutura

```
migrations/
└── <app>/
    ├── 000001_setup.up.sql            # extensões + funções globais
    ├── 000001_setup.down.sql
    ├── 000002_create_users.up.sql
    ├── 000002_create_users.down.sql
    └── ...
```

### Regras

- Numeração sequencial com 6 dígitos: `000001`, `000002`.
- Sempre criar par `up.sql` + `down.sql`.
- `down.sql` reverte **exatamente** o que o `up.sql` fez.
- Cada migration faz **uma coisa** (criar tabela, adicionar coluna, criar índice).
- **Nunca** alterar migration já aplicada em produção — criar nova.
- Testar `up` e `down` localmente antes de commit.
- Usar `golang-migrate` para execução.

### Migrations Zero-Downtime

| Operação                              | Segura? | Como Fazer                                                                |
|---------------------------------------|---------|---------------------------------------------------------------------------|
| Criar tabela                          | Sim     | `CREATE TABLE` normal                                                     |
| Adicionar coluna nullable             | Sim     | `ALTER TABLE ADD COLUMN ... NULL`                                         |
| Adicionar coluna NOT NULL com default | Sim     | `ALTER TABLE ADD COLUMN ... NOT NULL DEFAULT ...` (PG 11+ não reescreve)  |
| Remover coluna                        | Sim     | `ALTER TABLE DROP COLUMN` (após confirmar que nenhum código referencia)   |
| Criar índice                          | **Perigoso** | `CREATE INDEX CONCURRENTLY` (não bloqueia writes)                    |
| Renomear coluna                       | **Perigoso** | Nova coluna → copiar → dropar antiga, em migrations separadas         |
| Alterar tipo de coluna                | **Perigoso** | Criar nova, migrar dados, renomear                                    |
| Adicionar constraint                  | **Perigoso** | `ADD CONSTRAINT ... NOT VALID` + `VALIDATE CONSTRAINT` em migration separada |

## 11. Transactions

### Quando usar

| Cenário                                       | Transaction? | Por quê                                              |
|-----------------------------------------------|--------------|------------------------------------------------------|
| INSERT único                                  | **Não**      | Autocommit já garante atomicidade                    |
| SELECT único                                  | **Não**      | Leitura simples não precisa                          |
| INSERT + INSERT relacionados                  | **Sim**      | Tudo ou nada                                         |
| UPDATE condicional (read-then-write)          | **Sim**      | Evitar race condition                                |
| Transferência de saldo                        | **Sim**      | Débito e crédito atômicos                            |
| DELETE em cascata manual                      | **Sim**      | Consistência ao remover relacionados                 |
| Operação de negócio com múltiplas entidades   | **Sim**      | Order + estoque + log                                |
| Bulk insert                                   | **Sim**      | Sem registros órfãos                                 |
| Leitura consistente de múltiplas tabelas      | **Sim** (`READ ONLY`) | Snapshot isolation                          |

### Helper `pkg/postgres/tx.go`

```go
package postgres

import (
    "context"
    "database/sql"
    "fmt"
)

// RunInTx executa fn dentro de uma transaction.
// COMMIT se fn retorna nil, ROLLBACK caso contrário.
func RunInTx(ctx context.Context, db *sql.DB, fn func(tx *sql.Tx) error) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }

    if err := fn(tx); err != nil {
        if rbErr := tx.Rollback(); rbErr != nil {
            return fmt.Errorf("rollback failed: %v (original: %w)", rbErr, err)
        }
        return err
    }

    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit: %w", err)
    }

    return nil
}

func RunInTxWithOptions(ctx context.Context, db *sql.DB, opts *sql.TxOptions, fn func(tx *sql.Tx) error) error {
    tx, err := db.BeginTx(ctx, opts)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }

    if err := fn(tx); err != nil {
        if rbErr := tx.Rollback(); rbErr != nil {
            return fmt.Errorf("rollback failed: %v (original: %w)", rbErr, err)
        }
        return err
    }

    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit: %w", err)
    }

    return nil
}
```

### Uso no repositório

```go
func (r *OrderRepository) CreateWithItems(ctx context.Context, order *entity.Order, items []entity.OrderItem) error {
    return postgres.RunInTx(ctx, r.db, func(tx *sql.Tx) error {
        const orderQuery = `
            INSERT INTO "orders" ("userId", "status", "totalCents", "createdAt", "updatedAt")
            VALUES ($1, $2, $3, $4, $5)
            RETURNING "id"`

        err := tx.QueryRowContext(ctx, orderQuery,
            order.UserID, order.Status, order.TotalCents, order.CreatedAt, order.UpdatedAt,
        ).Scan(&order.ID)
        if err != nil {
            return fmt.Errorf("insert order: %w", err)
        }

        const itemQuery = `
            INSERT INTO "orderItems" ("orderId", "productId", "quantity", "unitPriceCents")
            VALUES ($1, $2, $3, $4)
            RETURNING "id"`

        for i := range items {
            items[i].OrderID = order.ID
            err := tx.QueryRowContext(ctx, itemQuery,
                items[i].OrderID, items[i].ProductID, items[i].Quantity, items[i].UnitPriceCents,
            ).Scan(&items[i].ID)
            if err != nil {
                return fmt.Errorf("insert order item: %w", err)
            }
        }
        return nil
    })
}
```

### Orquestração no use case com `Transactor`

Quando a transaction envolve **múltiplos repositórios**, o use case orquestra via interface `Transactor`:

```go
// domain/repository/transactor.go
type Transactor interface {
    RunInTx(ctx context.Context, fn func(tx *sql.Tx) error) error
}
```

```go
// infrastructure/repository/transactor.go
type PostgresTransactor struct{ db *sql.DB }

func NewPostgresTransactor(db *sql.DB) domainrepo.Transactor {
    return &PostgresTransactor{db: db}
}

func (t *PostgresTransactor) RunInTx(ctx context.Context, fn func(tx *sql.Tx) error) error {
    return postgres.RunInTx(ctx, t.db, fn)
}
```

```go
// application/usecase/billing/transfer-usecase.go
func (u *TransferUseCase) Perform(ctx context.Context, in TransferInput) error {
    return u.transactor.RunInTx(ctx, func(tx *sql.Tx) error {
        if err := u.accountRepo.DebitWithTx(ctx, tx, in.FromAccountID, in.Amount); err != nil {
            return fmt.Errorf("debit: %w", err)
        }
        if err := u.accountRepo.CreditWithTx(ctx, tx, in.ToAccountID, in.Amount); err != nil {
            return fmt.Errorf("credit: %w", err)
        }
        return nil
    })
}
```

Repositórios que participam de tx externa expõem variantes `WithTx`:

```go
func (r *AccountRepository) DebitWithTx(ctx context.Context, tx *sql.Tx, id string, amount int64) error {
    const query = `
        UPDATE "accounts"
        SET "balanceCents" = "balanceCents" - $1, "updatedAt" = now()
        WHERE "id" = $2 AND "balanceCents" >= $1`

    res, err := tx.ExecContext(ctx, query, amount, id)
    if err != nil {
        return fmt.Errorf("debit account: %w", err)
    }
    rows, _ := res.RowsAffected()
    if rows == 0 {
        return entity.ErrInsufficientBalance
    }
    return nil
}
```

### Níveis de isolamento

| Nível                       | Quando usar                              | Config                                                        |
|-----------------------------|------------------------------------------|---------------------------------------------------------------|
| `READ COMMITTED` (padrão)   | 90% dos casos                            | `nil`                                                         |
| `REPEATABLE READ`           | Leitura consistente entre tabelas        | `&sql.TxOptions{Isolation: sql.LevelRepeatableRead}`          |
| `SERIALIZABLE`              | Operações críticas de saldo              | `&sql.TxOptions{Isolation: sql.LevelSerializable}`            |
| `READ ONLY`                 | Relatórios                               | `&sql.TxOptions{ReadOnly: true}`                              |

### Regras de transactions

| Regra                                    | Detalhe                                                                          |
|------------------------------------------|----------------------------------------------------------------------------------|
| **Transactions curtas**                  | Milissegundos. Quanto mais longa, mais locks retidos                             |
| **Nenhum I/O externo dentro do tx**      | Nada de HTTP, NATS, RabbitMQ dentro do tx — sempre **após** o commit             |
| **Evento DEPOIS do commit**              | `repo.Create` dentro do tx, `event.Publish` fora                                 |
| **Retry em serialization failure**       | `SQLSTATE 40001` deve ser retentado pelo app                                     |
| **Sem tx para operações únicas**         | INSERT, SELECT, UPDATE únicos não precisam                                       |
| **Sem tx aninhados**                     | PostgreSQL não suporta — usar `SAVEPOINT` se necessário                          |
| **Context com timeout**                  | Sempre propagar `context.Context` com deadline                                   |
| **Commit explícito**                     | Nunca confiar em close automático                                                |

### Anti-pattern: evento dentro da transaction

```go
// ERRADO — evento publicado antes do commit
return postgres.RunInTx(ctx, u.db, func(tx *sql.Tx) error {
    u.repo.CreateWithTx(ctx, tx, entity)
    u.event.PublishCreated(ctx, entity) // ERRADO
    return nil
})

// CORRETO — evento após o commit
if err := u.repo.Create(ctx, entity); err != nil {
    return err
}
_ = u.event.PublishCreated(ctx, entity)
return nil
```

## 12. Paginação

### Por cursor (recomendada)

```sql
-- Primeira página
SELECT "id", "name", "email", "createdAt"
FROM "users"
WHERE "deletedAt" IS NULL
ORDER BY "id" ASC
LIMIT $1;

-- Próximas (cursor = último ID)
SELECT "id", "name", "email", "createdAt"
FROM "users"
WHERE "deletedAt" IS NULL AND "id" > $1
ORDER BY "id" ASC
LIMIT $2;
```

- Performance constante (usa índice).
- Sem duplicados/pulos quando dados mudam.
- Não usar quando o usuário precisa pular para uma página específica.

### Por offset (apenas quando necessário)

```sql
SELECT "id", "name", "email"
FROM "users"
WHERE "deletedAt" IS NULL
ORDER BY "createdAt" DESC
LIMIT $1 OFFSET $2;
```

- Performance degrada com offsets grandes.
- Resultados inconsistentes se dados mudam.
- Usar **apenas** em dashboards administrativos com poucas páginas.

## 13. Queries — Boas Práticas

### SELECT explícito (nunca `SELECT *`)

```sql
SELECT "id", "name", "email", "createdAt"
FROM "users"
WHERE "id" = $1;
```

### INSERT com RETURNING

```sql
INSERT INTO "users" ("name", "email", "createdAt", "updatedAt")
VALUES ($1, $2, $3, $4)
RETURNING "id", "createdAt";
```

### UPSERT (idempotência)

```sql
-- Ignorar duplicata
INSERT INTO "accounts" ("userId", "name", "email", "createdAt", "updatedAt")
VALUES ($1, $2, $3, $4, $5)
ON CONFLICT ("userId") DO NOTHING;

-- Inserir ou atualizar
INSERT INTO "accounts" ("userId", "name", "email", "createdAt", "updatedAt")
VALUES ($1, $2, $3, $4, $5)
ON CONFLICT ("userId") DO UPDATE SET
    "name"      = EXCLUDED."name",
    "email"     = EXCLUDED."email",
    "updatedAt" = EXCLUDED."updatedAt";
```

### Bulk insert

```sql
INSERT INTO "orderItems" ("orderId", "productId", "quantity", "unitPriceCents")
VALUES
    ($1, $2, $3, $4),
    ($5, $6, $7, $8),
    ($9, $10, $11, $12)
RETURNING "id";
```

### Locking

```sql
-- Lock para read-then-write
SELECT "id", "balanceCents"
FROM "accounts"
WHERE "id" = $1
FOR UPDATE;

-- Filas de trabalho (concorrência segura)
SELECT "id", "payload"
FROM "jobs"
WHERE "status" = 'pending'
ORDER BY "createdAt" ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

## 14. Pool de Conexões (Go)

```go
// pkg/postgres/conn.go
db.SetMaxOpenConns(cfg.MaxOpenConns)    // 10-25 (2-5× núcleos do DB)
db.SetMaxIdleConns(cfg.MaxIdleConns)    // = MaxOpenConns
db.SetConnMaxLifetime(30 * time.Minute) // rotaciona (DNS, PgBouncer)
db.SetConnMaxIdleTime(5 * time.Minute)  // libera ociosas
```

| Parâmetro          | Recomendação        | Justificativa                                            |
|--------------------|---------------------|----------------------------------------------------------|
| `MaxOpenConns`     | 10-25               | `max_connections` do PG ÷ instâncias do app              |
| `MaxIdleConns`     | = `MaxOpenConns`    | Manter conns quentes                                     |
| `ConnMaxLifetime`  | 30 min              | Rotacionar para respeitar DNS/PgBouncer                  |
| `ConnMaxIdleTime`  | 5 min               | Liberar conns em baixo tráfego                           |

## 15. Configuração do Servidor (PostgreSQL 18)

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:18
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
    command:
      - "postgres"
      # Memória
      - "-c"
      - "shared_buffers=256MB"            # ~25% da RAM
      - "-c"
      - "effective_cache_size=512MB"      # 50-75% da RAM
      - "-c"
      - "work_mem=16MB"                   # por operação de sort/hash
      - "-c"
      - "maintenance_work_mem=128MB"      # VACUUM, CREATE INDEX
      # WAL
      - "-c"
      - "wal_buffers=16MB"
      - "-c"
      - "min_wal_size=512MB"
      - "-c"
      - "max_wal_size=2GB"
      # Planner
      - "-c"
      - "random_page_cost=1.1"            # SSD
      - "-c"
      - "effective_io_concurrency=200"    # SSD
      # Logging
      - "-c"
      - "log_min_duration_statement=100"  # loga queries > 100ms
      - "-c"
      - "log_statement=none"
      - "-c"
      - "log_lock_waits=on"
      # Conexões
      - "-c"
      - "max_connections=100"
      # Checkpoints
      - "-c"
      - "checkpoint_completion_target=0.9"

volumes:
  pgdata:
```

## Checklist — Nova Tabela

- [ ] Nome em camelCase plural com aspas duplas (`"orderItems"`)
- [ ] `"id" UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- [ ] Colunas em camelCase com aspas duplas
- [ ] `TIMESTAMPTZ` para datas (nunca `TIMESTAMP`)
- [ ] `"createdAt" TIMESTAMPTZ NOT NULL DEFAULT now()`
- [ ] `"updatedAt" TIMESTAMPTZ NOT NULL DEFAULT now()`
- [ ] Trigger `"trg_<tabela>_updatedAt"` registrado
- [ ] Constraints nomeadas (`CONSTRAINT "<tabela>_<descricao>"`)
- [ ] FKs com índice na coluna FK
- [ ] FKs com `ON DELETE` explícito
- [ ] `NOT NULL` em todas as colunas obrigatórias
- [ ] `CHECK` para enumerações (em vez de `ENUM` type)
- [ ] Índices para colunas em `WHERE` e `ORDER BY`
- [ ] `INSERT ... RETURNING "id"` no repositório
- [ ] `sql.ErrNoRows` tratado como `nil, nil` no `FindBy*`
- [ ] Migration `up.sql` e `down.sql` criadas e testadas

## Checklist — Nova Transaction

- [ ] Múltiplas escritas ou read-then-write? Se sim, usar transaction
- [ ] `RunInTx` ou `RunInTxWithOptions` do `pkg/postgres`
- [ ] Nenhuma chamada externa (HTTP, NATS, RabbitMQ) dentro do tx
- [ ] Evento publicado **depois** do commit
- [ ] Context com timeout propagado
- [ ] Nível de isolamento adequado (`READ COMMITTED` na maioria)
- [ ] Retry para `serialization_failure` (40001) se usando `SERIALIZABLE`
- [ ] Repositórios participantes expõem variantes `WithTx`

## Checklist — Nova Migration

- [ ] Numeração sequencial com 6 dígitos
- [ ] Par `up.sql` + `down.sql` criados
- [ ] `down.sql` reverte exatamente o `up.sql`
- [ ] Uma única responsabilidade por migration
- [ ] `CREATE INDEX CONCURRENTLY` para tabelas grandes
- [ ] Constraints adicionadas via `NOT VALID` + `VALIDATE` separados
- [ ] Testada localmente (up e down) antes do commit
- [ ] Não altera migration já aplicada em produção

## Exemplo Completo — Migration de Tabela

```sql
-- migrations/billing/000002_create_orders.up.sql

CREATE TABLE "orders" (
    "id"          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    "userId"      UUID        NOT NULL,
    "status"      TEXT        NOT NULL DEFAULT 'pending',
    "totalCents"  BIGINT      NOT NULL,
    "currency"    VARCHAR(3)  NOT NULL DEFAULT 'BRL',
    "notes"       TEXT,
    "metadata"    JSONB,
    "createdAt"   TIMESTAMPTZ NOT NULL DEFAULT now(),
    "updatedAt"   TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT "orders_userId_fk"
        FOREIGN KEY ("userId")
        REFERENCES "users" ("id")
        ON DELETE RESTRICT
        ON UPDATE CASCADE,

    CONSTRAINT "orders_statusCheck"
        CHECK ("status" IN ('pending', 'processing', 'completed', 'cancelled', 'refunded')),

    CONSTRAINT "orders_totalPositive"
        CHECK ("totalCents" > 0),

    CONSTRAINT "orders_currencyCheck"
        CHECK ("currency" IN ('BRL', 'USD', 'EUR'))
);

CREATE TRIGGER "trg_orders_updatedAt"
    BEFORE UPDATE ON "orders"
    FOR EACH ROW
    EXECUTE FUNCTION "setUpdatedAt"();

CREATE INDEX "idx_orders_userId"          ON "orders" ("userId");
CREATE INDEX "idx_orders_status"          ON "orders" ("status") WHERE "status" IN ('pending', 'processing');
CREATE INDEX "idx_orders_createdAt"       ON "orders" ("createdAt" DESC);
CREATE INDEX "idx_orders_userId_status"   ON "orders" ("userId", "status");

COMMENT ON TABLE  "orders"               IS 'Pedidos de compra dos usuários';
COMMENT ON COLUMN "orders"."totalCents"  IS 'Valor total em centavos';
COMMENT ON COLUMN "orders"."metadata"    IS 'Dados adicionais semi-estruturados do pedido';
```

```sql
-- migrations/billing/000002_create_orders.down.sql

DROP TRIGGER IF EXISTS "trg_orders_updatedAt" ON "orders";
DROP TABLE   IF EXISTS "orders";
```

## Fontes

- https://www.postgresql.org/docs/18/ — documentação oficial PG 18
- https://www.postgresql.org/docs/18/ddl.html — DDL, constraints, herança
- https://www.postgresql.org/docs/18/datatype.html — tipos de dados
- https://www.postgresql.org/docs/18/indexes.html — índices (B-tree, GIN, BRIN, partial)
- https://www.postgresql.org/docs/18/transaction-iso.html — níveis de isolamento
- https://www.postgresql.org/docs/18/explicit-locking.html — locks e `FOR UPDATE`/`SKIP LOCKED`
- https://www.postgresql.org/docs/18/runtime-config.html — configuração do servidor
- https://github.com/golang-migrate/migrate — ferramenta de migrations
