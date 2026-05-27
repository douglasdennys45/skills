---
name: nats-best-practices
description: Aplica boas práticas oficiais do NATS 2.10+ e JetStream com Go (nats.go) ao desenhar, escrever, revisar ou refatorar publishers, subscribers, streams, consumers e workers. Foco especial em padrões de distribuição de carga onde múltiplos workers leem do mesmo subject — Core NATS Queue Groups para fan-out efêmero e JetStream Pull Consumers compartilhados (durable + queue) para entrega persistente, exactly-once, com ack/nak/term, retry com backoff, redelivery, deduplicação por Nats-Msg-Id, controle de inflight (MaxAckPending), backpressure, graceful shutdown, idempotência, observabilidade e DLQ. Cobre também convenções de nomenclatura de subjects (kebab-case hierárquico), conexão resiliente (reconnect, jitter, MaxReconnects=-1), autenticação (NKey/JWT/credentials), TLS, segregação de contextos (NATS Context), pooling de conexões, fluxo request/reply, KV/Object Store, mirrors/sources e configuração de Stream (Retention, Storage, Replicas, Discard, MaxMsgs, MaxBytes, MaxAge). Use sempre que criar/editar código Go que use nats.go ou jetstream/v2, ao modelar fluxos de mensageria, ao escalar consumers horizontalmente, ao decidir entre Core NATS vs JetStream, ao definir estratégia de retry/DLQ, ou ao revisar handlers de mensagem para garantir at-least-once + idempotência.
---

# NATS 2.10+ & JetStream Best Practices (Go)

Skill baseada na documentação oficial NATS (docs.nats.io), no client `nats.go` v1.40+ e no pacote moderno `github.com/nats-io/nats.go/jetstream` (v2 API). Cobre padrões de produção para mensageria com foco em escalabilidade horizontal de workers consumindo do mesmo subject.

## Quando Aplicar

- Ao criar/editar código Go que importe `github.com/nats-io/nats.go` ou `github.com/nats-io/nats.go/jetstream`
- Ao modelar um novo fluxo de mensageria (event, command, query)
- Ao escalar consumers horizontalmente (múltiplas réplicas do mesmo worker)
- Ao decidir entre **Core NATS** (efêmero, at-most-once) vs **JetStream** (persistente, at-least-once)
- Ao definir Stream, Consumer, Retention Policy, Storage, Replicas
- Ao implementar retry, backoff, dead-letter queue (DLQ) ou redelivery
- Ao desenhar idempotência e deduplicação (`Nats-Msg-Id`)
- Ao revisar handlers para garantir ack/nak/term corretos
- Ao configurar graceful shutdown, MaxAckPending, FetchBatch, AckWait
- Ao definir convenção de subjects, autenticação (NKey/JWT/creds), TLS

## 1. Core NATS vs JetStream — Quando Usar

| Característica          | Core NATS                          | JetStream                                  |
|-------------------------|------------------------------------|--------------------------------------------|
| Entrega                 | At-most-once (fire & forget)       | At-least-once (com ack) ou exactly-once (com dedup) |
| Persistência            | **Não** (in-memory, sem replay)    | **Sim** (file ou memory store)              |
| Replay                  | Não                                | Sim (StartSeq, StartTime, DeliverAll, etc.) |
| Ack/Nak                 | Não                                | Sim                                         |
| Múltiplos consumers     | Queue Group (subscription efêmera) | Consumer durável (1 ou N processos)         |
| Replicas                | Cluster mesh                       | RAFT por Stream (R1/R3/R5)                  |
| Use case                | Notificações, pub/sub volátil, RPC | Eventos de domínio, jobs, comandos críticos |

**Regra prática:**
- Use **Core NATS** quando perder a mensagem é aceitável (telemetria, broadcast de presença, request/reply síncrono).
- Use **JetStream** quando a mensagem **não pode** ser perdida (event sourcing, jobs, integração com Postgres, billing, auditoria).
- Em dúvida: **JetStream**.

## 2. Convenção de Subjects

Subjects são **hierárquicos**, separados por ponto (`.`). Use **kebab-case** dentro de cada token.

| Padrão                                       | Exemplo                                          |
|----------------------------------------------|--------------------------------------------------|
| `<dominio>.<entidade>.<evento>`              | `billing.invoice.created`                        |
| `<dominio>.<entidade>.<id>.<evento>`         | `orders.order.4f2a.paid`                         |
| `cmd.<dominio>.<acao>`                       | `cmd.email.send`                                 |
| `q.<dominio>.<acao>`                         | `q.search.user-by-email` (request/reply)         |
| `dlq.<stream>`                               | `dlq.orders-events`                              |

Wildcards:
- `*` casa **um** token: `orders.order.*.paid` → casa `orders.order.4f2a.paid`.
- `>` casa **um ou mais** tokens (sempre no final): `orders.>` → casa `orders.order.4f2a.paid`.

### Regras

- Subjects são **case-sensitive**. Padronize tudo minúsculo.
- Não inclua dados sensíveis no subject (vão pro broker em texto, logs, monitor `$SYS.>`).
- Stream nomeado em **kebab-case maiúsculo** ou sufixo `-events`: `ORDERS_EVENTS`, `billing-events`.
- Consumer durable nomeado por **propósito do worker**: `email-sender-worker`, `invoice-projector`.
- Evite `>` em consumer de produção sem necessidade — assina o stream inteiro.

## 3. Conexão — Setup Resiliente

```go
package natsconn

import (
    "fmt"
    "time"

    "github.com/nats-io/nats.go"
)

func Connect(cfg Config) (*nats.Conn, error) {
    opts := []nats.Option{
        nats.Name(cfg.AppName),                    // aparece no monitor — sempre setar
        nats.MaxReconnects(-1),                    // reconectar indefinidamente
        nats.ReconnectWait(2 * time.Second),
        nats.ReconnectJitter(500*time.Millisecond, 2*time.Second), // jitter evita thundering herd
        nats.PingInterval(20 * time.Second),
        nats.MaxPingsOutstanding(3),
        nats.Timeout(5 * time.Second),             // conexão inicial
        nats.DrainTimeout(30 * time.Second),       // para graceful shutdown
        nats.ErrorHandler(func(_ *nats.Conn, sub *nats.Subscription, err error) {
            // logar com slog — nunca silenciar
            slog.Error("nats async error", "sub", sub.Subject, "err", err)
        }),
        nats.DisconnectErrHandler(func(_ *nats.Conn, err error) {
            slog.Warn("nats disconnected", "err", err)
        }),
        nats.ReconnectHandler(func(nc *nats.Conn) {
            slog.Info("nats reconnected", "url", nc.ConnectedUrl())
        }),
        nats.ClosedHandler(func(nc *nats.Conn) {
            slog.Warn("nats connection closed", "last_err", nc.LastError())
        }),
    }

    if cfg.CredsFile != "" {
        opts = append(opts, nats.UserCredentials(cfg.CredsFile)) // NATS 2.x JWT
    }
    if cfg.TLSEnabled {
        opts = append(opts, nats.RootCAs(cfg.CAFile))
        opts = append(opts, nats.ClientCert(cfg.CertFile, cfg.KeyFile))
    }

    nc, err := nats.Connect(cfg.URLs, opts...) // URLs: "nats://a:4222,nats://b:4222,nats://c:4222"
    if err != nil {
        return nil, fmt.Errorf("nats connect: %w", err)
    }
    return nc, nil
}
```

### Regras de conexão

| Regra                                  | Detalhe                                                              |
|----------------------------------------|----------------------------------------------------------------------|
| **1 conexão por processo**             | `*nats.Conn` é seguro para uso concorrente. Não criar pool.          |
| **Sempre setar `Name`**                | Aparece em `$SYS.REQ.SERVER.PING.CONNZ` — essencial para debug       |
| **`MaxReconnects(-1)`**                | Reconectar sempre; nunca desistir em produção                        |
| **`ReconnectJitter`**                  | Obrigatório em cluster — evita reconexão sincronizada                |
| **Lista de URLs**                      | Passar todos os seeds do cluster (`a,b,c`)                           |
| **Handlers obrigatórios**              | `ErrorHandler`, `DisconnectErrHandler`, `ReconnectHandler`, `ClosedHandler` |
| **Drain no shutdown**                  | `nc.Drain()` (não `nc.Close()`) — processa inflight e fecha          |
| **TLS em produção**                    | NKey/JWT + mutual TLS                                                |
| **Nada de `nats.NoReconnect()`**       | Exceto em scripts CLI one-shot                                       |

## 4. JetStream — Cliente Moderno (`jetstream` v2)

A partir de `nats.go` v1.30+, use o pacote `github.com/nats-io/nats.go/jetstream` (API v2) em vez do legado `nc.JetStream()`. O v2 separa **Stream Management**, **Consumer Management** e **Messaging**, e tem ergonomia muito melhor para pull consumers.

```go
import (
    "github.com/nats-io/nats.go"
    "github.com/nats-io/nats.go/jetstream"
)

js, err := jetstream.New(nc) // ou jetstream.NewWithDomain(nc, "hub")
if err != nil {
    return fmt.Errorf("jetstream new: %w", err)
}
```

## 5. Stream — Configuração

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

stream, err := js.CreateOrUpdateStream(ctx, jetstream.StreamConfig{
    Name:        "ORDERS_EVENTS",
    Description: "Eventos do domínio Orders",
    Subjects:    []string{"orders.>"},                  // capta todos os eventos do domínio
    Storage:     jetstream.FileStorage,                 // FileStorage em prod; MemoryStorage só para cache
    Retention:   jetstream.LimitsPolicy,                // LimitsPolicy é o default e o mais flexível
    Discard:     jetstream.DiscardOld,                  // ao bater limite, descarta antigos (FIFO)
    MaxMsgs:     -1,                                    // sem limite por contagem
    MaxBytes:    20 * 1024 * 1024 * 1024,               // 20 GB
    MaxAge:      30 * 24 * time.Hour,                   // 30 dias
    MaxMsgSize:  1 * 1024 * 1024,                       // 1 MB por mensagem
    Replicas:    3,                                     // R3 em prod (mínimo para HA)
    Duplicates:  2 * time.Minute,                       // janela de dedup por Nats-Msg-Id
    NoAck:       false,
})
if err != nil {
    return fmt.Errorf("create stream: %w", err)
}
```

### Retention Policies

| Policy             | Comportamento                                                            | Quando usar                                  |
|--------------------|--------------------------------------------------------------------------|----------------------------------------------|
| `LimitsPolicy`     | Mantém até bater limite (msgs, bytes, age). Default.                     | 90% dos casos — eventos, audit                |
| `WorkQueuePolicy`  | Apaga a mensagem depois que **qualquer** consumer fizer ack              | Job queue com 1 consumer por mensagem         |
| `InterestPolicy`   | Mantém enquanto **algum** consumer ainda está interessado                | Pub/sub durável sem replay para novos joiners |

### Storage

| Storage         | Durabilidade | Performance | Quando usar                          |
|-----------------|--------------|-------------|--------------------------------------|
| `FileStorage`   | Persiste     | Alta        | **Sempre em produção**               |
| `MemoryStorage` | Volátil      | Máxima      | Cache, dev, testes                   |

### Replicas

| Replicas | HA            | Performance         | Quando usar                          |
|----------|---------------|---------------------|--------------------------------------|
| 1        | Sem HA        | Máxima              | Dev, single-node                     |
| 3        | Sobrevive 1 falha | Bom              | Padrão produção                      |
| 5        | Sobrevive 2 falhas | Menor (mais RAFT) | Críticos (billing, ledger)         |

**Replicas só faz sentido com cluster JetStream com >= 3 servidores.**

## 6. Publisher — Padrão Robusto

```go
type Publisher struct {
    js jetstream.JetStream
}

func (p *Publisher) PublishEvent(ctx context.Context, evt domain.Event) error {
    payload, err := json.Marshal(evt)
    if err != nil {
        return fmt.Errorf("marshal event: %w", err)
    }

    subject := fmt.Sprintf("orders.order.%s.%s", evt.AggregateID, evt.Type)

    // Sync publish — bloqueia até ACK do server.
    // Use AsyncPublish + AsyncPublishComplete para throughput, mas
    // espere o ack ANTES de retornar do handler que chamou.
    ack, err := p.js.Publish(ctx, subject, payload,
        jetstream.WithMsgID(evt.ID),                    // OBRIGATÓRIO para dedup
        jetstream.WithExpectStream("ORDERS_EVENTS"),    // defensivo: rejeita se subject mudou de stream
    )
    if err != nil {
        return fmt.Errorf("publish: %w", err)
    }
    slog.Debug("event published", "stream", ack.Stream, "seq", ack.Sequence, "id", evt.ID)
    return nil
}
```

### Regras de publish

| Regra                                          | Por quê                                                              |
|------------------------------------------------|----------------------------------------------------------------------|
| **Sempre `WithMsgID(id)`** com ID estável      | Dedup automática na janela `Duplicates` do stream                    |
| **Sync publish em código transacional**        | Garante que a mensagem chegou ao stream antes de seguir              |
| **Async publish só para bulk/throughput**      | E sempre aguardar `PublishAsyncComplete()` antes do shutdown         |
| **Publicar DEPOIS do commit do DB**            | Nunca dentro de tx Postgres. Vide [[postgres-best-practices]]        |
| **Payload em JSON (ou protobuf)**              | JSON para flexibilidade; protobuf para volume alto                   |
| **Headers para metadados**                     | `traceparent`, `tenant-id`, `event-type`, `schema-version`           |
| **Subject estável**                            | Mudar subject = quebrar consumers. Versionar via header              |

### Headers — convenção mínima

```go
msg := &nats.Msg{
    Subject: subject,
    Data:    payload,
    Header:  nats.Header{},
}
msg.Header.Set("Nats-Msg-Id", evt.ID)              // dedup
msg.Header.Set("event-type", evt.Type)              // routing/filter
msg.Header.Set("schema-version", "v1")              // evolução
msg.Header.Set("traceparent", traceparent)          // W3C trace context
msg.Header.Set("tenant-id", evt.TenantID)
ack, err := p.js.PublishMsg(ctx, msg)
```

## 7. Múltiplos Workers no Mesmo Subject — O Padrão Central

### 7.1 Core NATS — Queue Subscription (efêmero, at-most-once)

Quando vários processos do mesmo serviço se inscrevem com o **mesmo queue group**, o NATS faz **load balancing**: cada mensagem vai para **exatamente um** worker do grupo.

```go
// Worker 1, 2, 3... cada réplica do binário roda este código.
// Todas no mesmo queue group => NATS distribui round-robin entre eles.
sub, err := nc.QueueSubscribe("orders.order.*.paid", "order-paid-workers", func(m *nats.Msg) {
    handle(m) // processado por UM worker apenas
})
if err != nil {
    return fmt.Errorf("queue subscribe: %w", err)
}
```

**Características:**
- **Sem persistência** — mensagem perdida se nenhum worker estiver up.
- **Sem replay, sem ack** — fire & forget.
- **Escala horizontalmente** apenas adicionando réplicas no mesmo queue group.

**Use somente quando** perder mensagens é aceitável (telemetria, broadcast, RPC).

### 7.2 JetStream — Pull Consumer Durável Compartilhado (recomendado)

**Este é o padrão recomendado para múltiplos workers** com persistência, ack e replay.

> **Como funciona:** crie **um único consumer durável** no stream. Todas as réplicas do worker fazem `Consume()` desse **mesmo consumer**. O servidor distribui mensagens entre as réplicas, respeitando `MaxAckPending`, `AckWait` e redelivery. É efetivamente um "queue group durável".

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

stream, err := js.Stream(ctx, "ORDERS_EVENTS")
if err != nil {
    return fmt.Errorf("get stream: %w", err)
}

// Cria/atualiza um consumer durável compartilhado por TODAS as réplicas do worker.
cons, err := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Name:          "order-paid-projector",                  // nome estável — mesmo em todas as réplicas
    Durable:       "order-paid-projector",                  // == Name para durabilidade
    Description:   "Projeta orders.*.paid no read model",
    FilterSubject: "orders.order.*.paid",                   // filtra do stream
    AckPolicy:     jetstream.AckExplicitPolicy,             // OBRIGATÓRIO ack manual
    AckWait:       30 * time.Second,                        // tempo até redelivery se sem ack
    MaxDeliver:    5,                                       // tentativas antes de descartar/DLQ
    BackOff: []time.Duration{                               // backoff entre redeliveries
        1 * time.Second,
        5 * time.Second,
        15 * time.Second,
        60 * time.Second,
    },
    MaxAckPending: 256,                                     // inflight por consumer (TODAS as réplicas somadas)
    DeliverPolicy: jetstream.DeliverAllPolicy,              // novo consumer pega tudo desde o início
    ReplayPolicy:  jetstream.ReplayInstantPolicy,
    Replicas:      3,                                       // HA do estado do consumer
})
if err != nil {
    return fmt.Errorf("create consumer: %w", err)
}
```

### 7.3 Worker — Loop de Consumo

```go
type Worker struct {
    cons    jetstream.Consumer
    handler Handler
    log     *slog.Logger
}

func (w *Worker) Run(ctx context.Context) error {
    // Consume entrega mensagens em callback de forma contínua, com controle de inflight.
    consumeCtx, err := w.cons.Consume(
        func(msg jetstream.Msg) {
            w.handle(ctx, msg)
        },
        jetstream.PullMaxMessages(64),                      // batch interno do client
        jetstream.PullExpiry(30*time.Second),
        jetstream.ConsumeErrHandler(func(_ jetstream.ConsumeContext, err error) {
            w.log.Error("consume error", "err", err)
        }),
    )
    if err != nil {
        return fmt.Errorf("consume: %w", err)
    }
    defer consumeCtx.Stop()

    <-ctx.Done()                                            // bloqueia até shutdown
    consumeCtx.Drain()                                      // processa inflight antes de sair
    return ctx.Err()
}

func (w *Worker) handle(ctx context.Context, msg jetstream.Msg) {
    meta, _ := msg.Metadata()
    log := w.log.With(
        "stream", meta.Stream,
        "consumer", meta.Consumer,
        "seq", meta.Sequence.Stream,
        "delivered", meta.NumDelivered,
        "msg_id", msg.Headers().Get("Nats-Msg-Id"),
    )

    // Timeout por mensagem — sempre menor que AckWait
    hctx, cancel := context.WithTimeout(ctx, 20*time.Second)
    defer cancel()

    err := w.handler.Handle(hctx, msg)
    switch {
    case err == nil:
        if ackErr := msg.Ack(); ackErr != nil {
            log.Error("ack failed", "err", ackErr) // será redelivered por AckWait
        }
    case errors.Is(err, domain.ErrPermanent):
        // erro terminal — não retentar; envia pro DLQ via Term()
        log.Warn("permanent error, terminating", "err", err)
        _ = msg.Term()
    default:
        // erro transitório — Nak com delay calculado
        delay := backoffFor(meta.NumDelivered)
        log.Warn("transient error, naking", "err", err, "delay", delay)
        _ = msg.NakWithDelay(delay)
    }
}
```

### 7.4 Escala Horizontal — Como Adicionar Mais Workers

**Não crie um consumer por réplica.** Crie **UM consumer durável** e suba **N réplicas** do mesmo binário apontando para esse consumer.

```
                        ┌─────────────────────────────┐
                        │  Stream: ORDERS_EVENTS      │
                        │  Subjects: orders.>         │
                        └─────────────┬───────────────┘
                                      │
                            ┌─────────▼──────────┐
                            │ Consumer Durável:  │
                            │ order-paid-projector│
                            │ MaxAckPending: 256 │
                            └─┬────────┬───────┬─┘
                              │        │       │
                  ┌───────────▼┐  ┌────▼───┐ ┌─▼──────────┐
                  │ Worker #1  │  │Worker#2│ │ Worker #3  │
                  └────────────┘  └────────┘ └────────────┘
```

| Decisão                                | Padrão Recomendado                                             |
|----------------------------------------|----------------------------------------------------------------|
| Quantos consumers?                     | **1 por propósito de processamento**, não por réplica          |
| `Durable` igual em todas as réplicas?  | **Sim** — é o que faz N réplicas dividirem trabalho            |
| Como aumentar paralelismo?             | (a) Subir mais réplicas; (b) aumentar `MaxAckPending`          |
| Como reduzir lag?                      | Aumentar réplicas + `PullMaxMessages` + paralelismo interno    |
| Como ter dois processamentos paralelos? | **Dois consumers** com nomes diferentes, mesmo FilterSubject  |

### 7.5 Workers Independentes para o Mesmo Subject (fan-out de domínio)

Quando dois domínios precisam reagir ao mesmo evento (ex: `orders.order.*.paid` precisa atualizar projeção **E** enviar email), crie **dois consumers durables distintos** apontando para o mesmo subject. Cada um é uma "trilha" independente, com seu próprio ack/retry.

```go
projectorCons, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Name: "order-paid-projector", Durable: "order-paid-projector",
    FilterSubject: "orders.order.*.paid",
    AckPolicy:     jetstream.AckExplicitPolicy,
})

emailCons, _ := stream.CreateOrUpdateConsumer(ctx, jetstream.ConsumerConfig{
    Name: "order-paid-emailer", Durable: "order-paid-emailer",
    FilterSubject: "orders.order.*.paid",
    AckPolicy:     jetstream.AckExplicitPolicy,
})
```

Cada consumer mantém seu próprio offset, redelivery e DLQ. Falha no email **não** afeta a projeção.

## 8. Ack, Nak, Term, InProgress — Tabela de Decisão

| Método                         | Significado                                            | Quando usar                                    |
|--------------------------------|--------------------------------------------------------|------------------------------------------------|
| `msg.Ack()`                    | Sucesso — não redeliver                                | Handler concluiu com sucesso                   |
| `msg.Nak()`                    | Falha — redeliver **imediatamente** (sujeito a backoff) | Erro transitório sem delay específico         |
| `msg.NakWithDelay(d)`          | Falha — redeliver após `d`                             | Erro transitório com backoff calculado         |
| `msg.Term()`                   | Erro terminal — **não** redeliver, **não** conta para MaxDeliver | Payload inválido, regra de negócio violada |
| `msg.InProgress()`             | "Ainda processando" — reseta AckWait                   | Handler longo (heartbeat para evitar redelivery) |
| `msg.DoubleAck(ctx)`           | Ack com confirmação do server                          | Quando perder o ack é catastrófico             |

### Anti-patterns

```go
// ERRADO — esquecer o ack
func handle(msg jetstream.Msg) {
    process(msg.Data())
    // sem ack → AckWait expira → redelivery infinita até MaxDeliver
}

// ERRADO — ack antes de processar
func handle(msg jetstream.Msg) {
    msg.Ack()           // ack já saiu...
    if err := process(msg.Data()); err != nil {
        // ...mas falhou. Mensagem perdida sem redelivery.
    }
}

// ERRADO — Nak sem distinguir erro permanente
func handle(msg jetstream.Msg) {
    if err := process(msg.Data()); err != nil {
        msg.Nak()       // payload corrompido vai redeliver 5x antes de desistir
    }
}

// CORRETO
func handle(msg jetstream.Msg) {
    err := process(msg.Data())
    switch {
    case err == nil:
        msg.Ack()
    case errors.Is(err, ErrPermanent):
        msg.Term()      // direto pro DLQ via MaxDeliver=1 ou MaxDeliver excedido
    default:
        msg.NakWithDelay(backoff)
    }
}
```

## 9. Idempotência — Obrigatório em At-Least-Once

JetStream entrega **pelo menos uma vez**. Toda redelivery, todo failover, todo restart pode causar duplicata. O handler **deve** ser idempotente.

### Estratégias

| Estratégia                              | Quando usar                                          |
|-----------------------------------------|------------------------------------------------------|
| **Dedup via `Nats-Msg-Id` + `Duplicates`** | Janela curta (minutos). Resolve duplicatas do publisher |
| **Tabela de processados** (`processed_events`) | Janela longa. UPSERT por `event_id`                 |
| **Operação naturalmente idempotente**   | `UPDATE SET status='paid' WHERE id=$1`               |
| **UPSERT com chave de negócio**         | `INSERT ... ON CONFLICT DO NOTHING`                  |
| **Versionamento otimista**              | `WHERE version = $expected_version`                  |

### Padrão: tabela de processados

```sql
CREATE TABLE "processedEvents" (
    "eventId"     TEXT        PRIMARY KEY,
    "consumer"    TEXT        NOT NULL,
    "processedAt" TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

```go
func (h *Handler) Handle(ctx context.Context, msg jetstream.Msg) error {
    eventID := msg.Headers().Get("Nats-Msg-Id")
    if eventID == "" {
        return fmt.Errorf("%w: missing Nats-Msg-Id", domain.ErrPermanent)
    }

    return h.tx.RunInTx(ctx, func(tx *sql.Tx) error {
        // Tenta marcar como processado; se já existe, é duplicata → ack sem reprocessar.
        const claim = `
            INSERT INTO "processedEvents" ("eventId", "consumer")
            VALUES ($1, $2)
            ON CONFLICT ("eventId") DO NOTHING
            RETURNING "eventId"`
        var got string
        err := tx.QueryRowContext(ctx, claim, eventID, h.consumerName).Scan(&got)
        if errors.Is(err, sql.ErrNoRows) {
            return nil // duplicata — apenas ack
        }
        if err != nil {
            return fmt.Errorf("claim event: %w", err)
        }
        return h.applyBusinessLogic(ctx, tx, msg)
    })
}
```

## 10. DLQ (Dead Letter Queue)

JetStream não tem DLQ embutida, mas você implementa via:

### Opção A — Stream avisa quando MaxDeliver é excedido

Configure o stream para emitir advisory em `$JS.EVENT.ADVISORY.CONSUMER.MAX_DELIVERIES.<STREAM>.<CONSUMER>`. Um worker dedicado pode capturar e republicar em `dlq.<consumer>`.

### Opção B — Publicar para DLQ explicitamente no `Term()`

```go
func (w *Worker) handle(ctx context.Context, msg jetstream.Msg) {
    meta, _ := msg.Metadata()

    err := w.handler.Handle(ctx, msg)
    if err == nil {
        _ = msg.Ack()
        return
    }

    // Se for permanente OU excedeu o limite → DLQ
    if errors.Is(err, domain.ErrPermanent) || meta.NumDelivered >= w.maxDeliver {
        if pubErr := w.publishDLQ(ctx, msg, err); pubErr != nil {
            // não consegue mandar pro DLQ — Nak para tentar de novo
            _ = msg.NakWithDelay(30 * time.Second)
            return
        }
        _ = msg.Term() // só term DEPOIS que DLQ confirmou
        return
    }
    _ = msg.NakWithDelay(backoffFor(meta.NumDelivered))
}

func (w *Worker) publishDLQ(ctx context.Context, original jetstream.Msg, cause error) error {
    headers := original.Headers().Clone()
    headers.Set("dlq-reason", cause.Error())
    headers.Set("dlq-original-subject", original.Subject())
    headers.Set("dlq-consumer", w.consumerName)
    headers.Set("dlq-at", time.Now().UTC().Format(time.RFC3339Nano))

    dlqMsg := &nats.Msg{
        Subject: fmt.Sprintf("dlq.%s", w.consumerName),
        Data:    original.Data(),
        Header:  headers,
    }
    _, err := w.js.PublishMsg(ctx, dlqMsg, jetstream.WithMsgID(original.Headers().Get("Nats-Msg-Id")))
    return err
}
```

Crie um stream separado `DLQ` capturando `dlq.>` com retenção longa (30-90 dias) para inspeção e replay manual.

## 11. Graceful Shutdown

```go
func main() {
    ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer cancel()

    nc, _ := natsconn.Connect(cfg)
    js, _ := jetstream.New(nc)
    worker := buildWorker(js)

    errCh := make(chan error, 1)
    go func() { errCh <- worker.Run(ctx) }()

    select {
    case <-ctx.Done():
        slog.Info("shutdown requested")
    case err := <-errCh:
        slog.Error("worker exited", "err", err)
    }

    // 1) parar de aceitar novas mensagens, mas processar inflight
    shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer shutdownCancel()
    _ = worker.Shutdown(shutdownCtx)

    // 2) drain da conexão NATS (flush + close)
    if err := nc.Drain(); err != nil {
        slog.Error("nats drain", "err", err)
    }
}
```

**Regras:**
- Use `nc.Drain()`, nunca `nc.Close()`, em shutdown gracioso.
- O `Consume()` deve receber `consumeCtx.Drain()` antes do `Stop()`.
- `AckWait` deve ser maior que o tempo de drain configurado.

## 12. Observabilidade

### Métricas mínimas por worker

| Métrica                          | Tipo       | Labels                                |
|----------------------------------|------------|---------------------------------------|
| `nats_messages_received_total`   | Counter    | `consumer`, `subject`                 |
| `nats_messages_acked_total`      | Counter    | `consumer`                            |
| `nats_messages_naked_total`      | Counter    | `consumer`, `reason`                  |
| `nats_messages_termed_total`     | Counter    | `consumer`, `reason`                  |
| `nats_handler_duration_seconds`  | Histogram  | `consumer`, `outcome`                 |
| `nats_consumer_pending`          | Gauge      | `stream`, `consumer` (via `cons.Info()`) |
| `nats_consumer_ack_pending`      | Gauge      | `stream`, `consumer`                  |
| `nats_consumer_num_redelivered`  | Gauge      | `stream`, `consumer`                  |

### Tracing (W3C)

Propague `traceparent` via header. Cada handler abre um span filho:

```go
ctx := otel.GetTextMapPropagator().Extract(ctx, propagation.HeaderCarrier(msg.Headers()))
ctx, span := tracer.Start(ctx, "nats.consume "+msg.Subject())
defer span.End()
```

### Logs estruturados

Sempre logar com `slog` incluindo: `stream`, `consumer`, `seq`, `delivered`, `msg_id`, `subject`, `error`.

## 13. Autenticação e Segurança

| Mecanismo            | Quando usar                                | Configuração no client                          |
|----------------------|--------------------------------------------|-------------------------------------------------|
| **JWT + NKey (creds)** | **Padrão produção** (multi-tenant)        | `nats.UserCredentials("path/to/user.creds")`    |
| **NKey puro**        | Account interno sem JWT                    | `nats.Nkey(nkey, sigHandler)`                   |
| **User/Password**    | Dev local                                  | `nats.UserInfo(user, pass)`                     |
| **Token**            | Scripts simples                            | `nats.Token(token)`                             |
| **TLS**              | **Sempre em produção** (mTLS preferível)   | `nats.RootCAs(ca)` + `nats.ClientCert(c, k)`    |

**Regras de segurança:**
- **Nunca** commitar `.creds`, NKey ou senha. Usar secret manager.
- **TLS obrigatório** em qualquer ambiente fora do localhost.
- Cada serviço com **sua própria credencial** — facilita revogação e auditoria.
- Use **accounts** NATS para isolar tenants/ambientes.
- Permissões granulares por subject (`publish: ["orders.>"]`, `subscribe: ["cmd.email.>"]`).

## 14. Request/Reply (Core NATS)

```go
// Servidor (responder)
sub, _ := nc.QueueSubscribe("q.search.user-by-email", "search-workers", func(m *nats.Msg) {
    user, err := repo.FindByEmail(m.Header.Get("email"))
    if err != nil {
        // padrão de erro: header "error"
        respondError(m, err)
        return
    }
    payload, _ := json.Marshal(user)
    _ = m.Respond(payload)
})

// Cliente (requester)
ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
defer cancel()
reply, err := nc.RequestWithContext(ctx, "q.search.user-by-email", payload)
```

**Regras:**
- Sempre usar `RequestWithContext` (timeout obrigatório).
- Queue group no respondente para load balance.
- Para alta concorrência, considere `RequestMsgWithContext` + headers para erros.
- Para fluxos críticos com replay/audit, **prefira JetStream** com subject `cmd.<dominio>.<acao>` + resposta via outro subject.

## 15. KV Store e Object Store

JetStream oferece duas abstrações úteis:

### KV — chave/valor versionado

```go
kv, _ := js.CreateKeyValue(ctx, jetstream.KeyValueConfig{
    Bucket:  "feature-flags",
    History: 5,
    TTL:     0, // sem TTL
})

_, _ = kv.Put(ctx, "billing.new-checkout", []byte("true"))
entry, _ := kv.Get(ctx, "billing.new-checkout")
```

Use para: feature flags, configuração dinâmica, leader election leve, cache distribuído.

### Object Store — blobs grandes

```go
os, _ := js.CreateObjectStore(ctx, jetstream.ObjectStoreConfig{Bucket: "invoices-pdf"})
_, _ = os.PutBytes(ctx, "inv-123.pdf", pdfBytes)
```

Use para: PDFs, imagens, exports — qualquer payload > 1 MB que não cabe bem em mensagem.

## 16. Anti-patterns a Evitar

| Anti-pattern                                                         | Por que evitar                                                      |
|----------------------------------------------------------------------|---------------------------------------------------------------------|
| Criar conexão NATS por requisição                                    | Conexão é cara, é thread-safe — use 1 por processo                  |
| Criar consumer por réplica do worker                                 | Quebra distribuição. Use 1 consumer durable compartilhado           |
| `MaxAckPending` muito alto (>10k)                                    | Consome RAM, atrasa redelivery, mascara lentidão                    |
| `MaxAckPending = 1` sem necessidade                                  | Serializa o consumer todo — péssimo throughput                      |
| Esquecer `WithMsgID` no publish                                      | Sem dedup nativa — handler precisa fazer tudo                       |
| Handler sem timeout                                                  | Trava o worker; AckWait expira; redelivery em loop                  |
| Ack antes de processar                                               | Perda silenciosa de mensagem em falha                               |
| Nak em erro permanente                                               | Estoura MaxDeliver — atrasa entrega do DLQ                          |
| Publicar dentro de transaction Postgres                              | Vide [[postgres-best-practices]] — eventos sempre **após** commit  |
| `Term()` sem mandar para DLQ                                         | Mensagem some sem rastro                                            |
| Subject com dado sensível (CPF, email)                               | Aparece em logs, monitor, advisory                                  |
| `nc.Close()` em shutdown gracioso                                    | Use `nc.Drain()` para processar inflight                            |
| `MaxReconnects` finito em produção                                   | Worker morre numa janela de rede                                    |
| Wildcard `>` em consumer crítico                                     | Captura mensagens inesperadas; subject novo entra silenciosamente   |
| `MemoryStorage` em produção                                          | Perde tudo em restart do server                                     |
| Replicas=1 em stream crítico                                         | Sem HA — perda total se o servidor cair                             |
| Stream com `WorkQueuePolicy` + múltiplos consumers diferentes        | Inválido — `WorkQueuePolicy` exige 1 consumer por mensagem          |

## 17. Configuração de Servidor (Docker)

```yaml
# docker-compose.yml — cluster JetStream R3 para dev/staging
services:
  nats-1:
    image: nats:2.10-alpine
    command:
      - "-js"                                    # habilita JetStream
      - "-sd"
      - "/data"                                  # storage dir
      - "-m"
      - "8222"                                   # monitor HTTP
      - "--cluster_name"
      - "app-cluster"
      - "--cluster"
      - "nats://0.0.0.0:6222"
      - "--routes"
      - "nats://nats-2:6222,nats://nats-3:6222"
      - "--server_name"
      - "nats-1"
    ports: ["4222:4222", "8222:8222"]
    volumes: ["nats1-data:/data"]

  nats-2:
    image: nats:2.10-alpine
    command:
      - "-js"
      - "-sd"
      - "/data"
      - "--cluster_name"
      - "app-cluster"
      - "--cluster"
      - "nats://0.0.0.0:6222"
      - "--routes"
      - "nats://nats-1:6222,nats://nats-3:6222"
      - "--server_name"
      - "nats-2"
    volumes: ["nats2-data:/data"]

  nats-3:
    image: nats:2.10-alpine
    command:
      - "-js"
      - "-sd"
      - "/data"
      - "--cluster_name"
      - "app-cluster"
      - "--cluster"
      - "nats://0.0.0.0:6222"
      - "--routes"
      - "nats://nats-1:6222,nats://nats-2:6222"
      - "--server_name"
      - "nats-3"
    volumes: ["nats3-data:/data"]

volumes:
  nats1-data: {}
  nats2-data: {}
  nats3-data: {}
```

## Checklist — Novo Stream

- [ ] Nome em UPPER_SNAKE ou kebab-case com sufixo `-events`
- [ ] `Subjects` cobre apenas o domínio (sem `>` raiz)
- [ ] `Storage: FileStorage` em produção
- [ ] `Replicas: 3` em produção
- [ ] `Retention` adequada (`LimitsPolicy` na maioria)
- [ ] `MaxAge`, `MaxBytes` definidos — nunca deixar default infinito
- [ ] `Duplicates` configurado (1-5 min) para dedup
- [ ] `MaxMsgSize` definido para evitar payloads gigantes
- [ ] Documentado no `Description` com finalidade do stream

## Checklist — Novo Consumer Durável

- [ ] `Name` == `Durable`, kebab-case, descreve o propósito do worker
- [ ] `FilterSubject` específico (não `>`)
- [ ] `AckPolicy: AckExplicitPolicy`
- [ ] `AckWait` >= maior `handler timeout` esperado
- [ ] `MaxDeliver` finito (3-10), nunca infinito sem DLQ
- [ ] `BackOff` definido — sem reentregar em loop apertado
- [ ] `MaxAckPending` calibrado ao throughput e à RAM
- [ ] `Replicas: 3` em produção (HA do estado do consumer)
- [ ] Description explica qual worker consome

## Checklist — Novo Worker

- [ ] 1 conexão NATS por processo, com Drain no shutdown
- [ ] Handlers com `context.Context` e timeout próprio
- [ ] Ack/Nak/Term explícitos em todos os caminhos
- [ ] Erros permanentes detectados (`errors.Is(err, ErrPermanent)`) → `Term` ou DLQ
- [ ] Idempotência garantida (dedup, UPSERT, ou tabela de processados)
- [ ] Métricas expostas: received, acked, naked, termed, duration, pending
- [ ] Tracing W3C propagado via `traceparent` header
- [ ] Logs com `stream`, `consumer`, `seq`, `delivered`, `msg_id`
- [ ] DLQ implementado (subject `dlq.<consumer>`)
- [ ] Graceful shutdown com `consumeCtx.Drain()` + `nc.Drain()`
- [ ] Testes de integração com `nats-server` real ou testcontainers

## Checklist — Múltiplos Workers no Mesmo Subject

- [ ] **Um único** consumer durável criado (não um por réplica)
- [ ] Todas as réplicas usam o **mesmo** `Durable` no `Consume()`
- [ ] `MaxAckPending` dimensionado para o **total** entre todas as réplicas
- [ ] Idempotência **obrigatória** — qualquer réplica pode pegar a redelivery
- [ ] Métrica de lag (`num_pending`) monitorada para decidir escalar réplicas
- [ ] Se precisar de processamentos paralelos: **dois consumers** com nomes distintos
- [ ] Se for Core NATS efêmero: `QueueSubscribe` com queue group estável

## Fontes

- https://docs.nats.io/nats-concepts/jetstream — conceitos JetStream
- https://docs.nats.io/nats-concepts/jetstream/consumers — Consumer config
- https://docs.nats.io/nats-concepts/jetstream/streams — Stream config
- https://docs.nats.io/using-nats/developer/connecting — conexão e reconexão
- https://docs.nats.io/using-nats/developer/sending — publish patterns
- https://docs.nats.io/using-nats/developer/receiving/queues — queue groups (Core)
- https://github.com/nats-io/nats.go — cliente oficial Go
- https://pkg.go.dev/github.com/nats-io/nats.go/jetstream — pacote jetstream v2
- https://docs.nats.io/running-a-nats-service/configuration/securing_nats — auth/TLS
- https://docs.nats.io/running-a-nats-service/configuration/clustering/jetstream_clustering — JetStream clustering
