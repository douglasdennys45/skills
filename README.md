# Skills

[![skills.sh](https://skills.sh/b/douglasdennys45/skills)](https://skills.sh/douglasdennys45/skills)

Coleção pessoal de **Agent Skills** para o Claude Code, focadas em desenvolvimento backend Go, arquitetura, mensageria, banco de dados e processos de QA/Review.

Cada skill é um conjunto de instruções carregadas sob demanda pelo Claude para aplicar boas práticas oficiais e convenções de projeto ao escrever, revisar ou refatorar código.

## Instalação rápida

```bash
npx skills add douglasdennys45/skills
```

Ou instale uma skill específica:

```bash
npx skills add douglasdennys45/skills/go-best-practices
```

## Estrutura

```
skills/
├── architecture-best-practices/
├── fiber-v3-best-practices/
├── go-best-practices/
├── go-testing-best-practices/
├── nats-best-practices/
├── postgres-best-practices/
├── qa-best-practices/
└── task-reviewer-best-practices/
```

Cada diretório contém um arquivo `SKILL.md` com frontmatter (`name`, `description`) e o corpo da skill em Markdown.

## Skills disponíveis

| Skill | Descrição |
|---|---|
| [architecture-best-practices](skills/architecture-best-practices/SKILL.md) | Arquitetura de diretórios em projetos Next.js com monorepo (Turborepo + pnpm), Module Federation (microfrontends) e Clean Architecture por domínio. Cobre camadas, nomenclatura, regras de dependência, Event Bus tipado, shared packages, testes por camada e CI/CD. |
| [fiber-v3-best-practices](skills/fiber-v3-best-practices/SKILL.md) | Boas práticas oficiais do Fiber v3: `fiber.App`/`Config`, roteamento, `fiber.Ctx` como interface, binding unificado, validação, error handling, middlewares oficiais, hooks de ciclo de vida, graceful shutdown, testes e migração v2 → v3. |
| [go-best-practices](skills/go-best-practices/SKILL.md) | Boas práticas oficiais do Go (Effective Go, Go Blog): nomenclatura, tratamento de erros (`errors.Is/As`, `%w`), concorrência (goroutines, channels, sync), `context.Context` e design de interfaces. |
| [go-testing-best-practices](skills/go-testing-best-practices/SKILL.md) | Pirâmide completa de testes em Go: unitários com `stretchr/testify`, integração com `uber-go/mock`, aceitação orientada a comportamento e E2E com `testcontainers-go`. |
| [nats-best-practices](skills/nats-best-practices/SKILL.md) | NATS 2.10+ e JetStream com `nats.go`: Queue Groups, Pull Consumers, ack/nak/term, retry com backoff, deduplicação, controle de inflight, DLQ, convenções de subjects e conexão resiliente. |
| [postgres-best-practices](skills/postgres-best-practices/SKILL.md) | PostgreSQL 18: nomenclatura em camelCase, tipos recomendados (UUID, TIMESTAMPTZ, JSONB), índices, soft delete, migrations zero-downtime, transactions, paginação por cursor e configuração de servidor. |
| [qa-best-practices](skills/qa-best-practices/SKILL.md) | Pipeline completo de QA backend Go: análise estática, testes unitários/integração com race detector, mutation testing, E2E de API, verificação de contrato Swagger/OpenAPI e geração de relatório com evidências. |
| [task-reviewer-best-practices](skills/task-reviewer-best-practices/SKILL.md) | Revisão sistemática de tarefas Go concluídas: análise de diff, validação contra padrões idiomáticos, Clean Architecture, Fiber v3, MongoDB, RabbitMQ, uber-go/fx, slog, testes e classificação de problemas (Crítico/Major/Minor/Positivo). |

## Como usar

### Instalação local (Claude Code)

Copie ou faça symlink dos diretórios desejados para `~/.claude/skills/`:

```bash
ln -s "$PWD/skills/go-best-practices" ~/.claude/skills/go-best-practices
```

Reinicie o Claude Code — a skill aparece automaticamente na lista de skills disponíveis e é carregada sob demanda quando o contexto da conversa bate com a descrição.

### Invocação

As skills são acionadas automaticamente pelo Claude com base no `description` do frontmatter. Também é possível invocá-las explicitamente pelo nome:

```
/go-best-practices
```

## Convenções

- Toda skill segue o formato oficial do Claude Code: `SKILL.md` com frontmatter YAML (`name`, `description`) seguido do corpo em Markdown.
- Skills relacionadas se referenciam via `[[nome-da-skill]]` no corpo.
- Conteúdo em **Português (BR)**.
- Cada skill é baseada na documentação oficial da tecnologia coberta.
