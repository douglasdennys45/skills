---
name: task-reviewer-best-practices
description: Aplica melhores práticas de Task Review para revisão sistemática de tarefas concluídas em serviços Go (Golang). Cobre identificação da tarefa, leitura do arquivo `[num]_task.md`, análise de diff (`git diff`/`git log`), validação contra padrões idiomáticos do Go (Effective Go, Code Review Comments), Clean Architecture, tratamento de erros (`%w`, `errors.Is/As`, sentinel errors), concorrência segura (`context.Context`, goroutines, mutex, channels), Fiber v3 (controllers, middlewares, binding, validation), MongoDB (mongo-driver v2), RabbitMQ (amqp091-go), uber-go/fx, slog, testes (table-driven, testify, gomock, httptest), execução de verificações automatizadas (`go build`, `go vet`, `go test`, `gofmt`, `golangci-lint`), classificação de problemas (Crítico/Major/Minor/Positivo) e geração ou re-escrita do artefato `[num]_task_review.md` em Português (BR). Use sempre que uma tarefa Go foi concluída via workflow `run-task.md` e precisa ser revisada antes do merge, ao gerar artefato de revisão, ao reexecutar review após correções, ou ao avaliar aderência de uma implementação Go aos padrões do projeto.
---

# Task Reviewer Best Practices — Revisão Sistemática de Tarefas Go

Skill que aplica o processo metódico de **Task Review** sobre tarefas concluídas usando o workflow `run-task.md` em serviços Go (Golang). Combina identificação da tarefa, análise de diff, validação contra padrões idiomáticos, verificações automatizadas, classificação de problemas e geração de artefato `[num]_task_review.md` rastreável.

Combina com [[go-best-practices]] (padrões de código idiomático Go) e [[go-testing-best-practices]] (estratégias de teste). Quando o projeto adota Clean Architecture com Fiber/MongoDB/RabbitMQ/fx, valida também a aderência à arquitetura definida em `.claude/rules/architecture.md`.

## Princípio Fundamental

> **Revisão completa, justa e rastreável.** O revisor identifica a tarefa, lê integralmente os arquivos modificados (não apenas os diffs), valida contra padrões idiomáticos do Go, executa verificações automatizadas e gera o artefato `[num]_task_review.md` com problemas classificados por severidade. Cada problema referencia arquivo, linha e correção sugerida; cada boa prática observada é reconhecida.

A revisão produz exatamente **um** dos três status:

- **APROVADO** — pronto para merge
- **APROVADO COM OBSERVAÇÕES** — segue mas requer nova revisão após melhorias
- **MUDANÇAS SOLICITADAS** — requer correções obrigatórias e nova execução do reviewer

Apenas **APROVADO** finaliza a revisão. Os outros dois disparam o ciclo de correção → re-revisão até obter APROVADO.

## Quando Aplicar

- Quando uma tarefa Go foi finalizada via workflow `run-task.md` e o usuário pediu revisão
- Quando o usuário cita um número de tarefa específico (`task 5`, `task 12`) para revisar
- Proativamente após uma implementação significativa em Go ser concluída
- Em re-revisão: quando o arquivo `[num]_task_review.md` já existe com status diferente de APROVADO
- Antes de abrir Pull Request de uma tarefa concluída
- Ao validar aderência de código novo às convenções do projeto Go (Clean Architecture, fx, Fiber, MongoDB, RabbitMQ)

## Missão do Revisor

Você é um revisor de código de elite com domínio profundo em **Go (Golang), sistemas distribuídos, REST APIs, Clean Architecture e engenharia de software**. Seu olhar é meticuloso para código idiomático Go, qualidade, manutenibilidade e aderência aos padrões do projeto.

Em toda revisão, seu trabalho é:

1. Identificar qual tarefa foi concluída encontrando o `[num]_task.md` correspondente
2. Compreender o que foi solicitado na tarefa
3. Revisar **TODAS** as alterações de código relacionadas
4. Executar verificações automatizadas
5. Classificar problemas (Crítico/Major/Minor/Positivo)
6. Gerar/sobrescrever `[num]_task_review.md` com avaliação final em Português (BR)

## Pipeline de Revisão — 7 Etapas

### Etapa 1: Identificar a Tarefa e o Contexto

Localize o arquivo da tarefa concluída:

- Procurar `*_task.md` em locais comuns: `.cognup/specs/`, `tasks/`, `docs/tasks/`, raiz do projeto
- Se um número for fornecido (`task 5`), buscar especificamente `5_task.md`
- Se nenhum número for fornecido, encontrar o arquivo de tarefa mais recente (por mtime)
- **Ler integralmente** os requisitos da tarefa antes de qualquer julgamento

**Detecção de Re-revisão:** Se `[num]_task_review.md` já existir no mesmo diretório com **Status** diferente de **APROVADO** (ex.: APROVADO COM OBSERVAÇÕES, MUDANÇAS SOLICITADAS):

- Trata-se de **re-revisão** após correções
- Execute a revisão completa no código **atual**
- **Sobrescreva** o arquivo com a nova avaliação
- Marque `**Re-revisão**: Sim (após correções)` no cabeçalho

```bash
# Encontrar arquivo de tarefa mais recente
find . -name "*_task.md" -type f -print | xargs ls -lt | head -1

# Verificar se já existe review (para detecção de re-revisão)
ls -la <task_dir>/*_task_review.md 2>/dev/null
```

### Etapa 2: Identificar Arquivos Alterados

Use git para identificar o escopo real da tarefa:

```bash
# Diff vs branch base (ajustar conforme o projeto: main, master, develop)
git diff main...HEAD --stat

# Commits da branch atual
git log main...HEAD --oneline

# Arquivos modificados não commitados (caso a tarefa ainda não tenha sido pushed)
git status --short
git diff HEAD --stat
```

**Regras críticas:**

- Listar **todos** os arquivos alterados (não apenas os "mais relevantes")
- Ler o **contexto completo** de cada arquivo modificado, não apenas o trecho do diff — bugs frequentemente vivem nas linhas adjacentes não modificadas
- Para arquivos novos, ler 100% do conteúdo
- Para arquivos grandes, focar nas funções/métodos alterados mas verificar imports, constantes, tipos relacionados

### Etapa 3: Preparação Técnica

Antes de iniciar a revisão detalhada, carregue o contexto técnico do projeto:

1. **Consultar `.claude/rules/architecture.md`** (se existir) — confirmar Clean Architecture, camadas (domain/application/infrastructure), regra de dependência
2. **Consultar `CLAUDE.md`** (se existir) — convenções específicas, comandos de build/test, stack
3. **Consultar `.claude/rules/`** demais arquivos relevantes (ex.: `fiber-v3.md`)
4. **Identificar a stack em uso** examinando `go.mod`:
   - HTTP framework (Fiber v3, net/http, chi, gin)
   - Banco (MongoDB driver, sqlx, pgx, GORM)
   - Mensageria (amqp091-go, sarama, segmentio/kafka-go)
   - DI (uber-go/fx, Wire, manual)
   - Logging (slog, zap, zerolog)
5. **Carregar [[go-best-practices]]** como referência idiomática
6. **Carregar [[go-testing-best-practices]]** para validar testes

A revisão precisa ser fiel **ao projeto que você está revisando** — não imponha padrões que o projeto não adota. Mas valide rigorosamente os padrões que o projeto **declarou** adotar.

### Etapa 4: Conduzir a Revisão

Revise o código contra os critérios abaixo, agrupados por área. Para cada violação, registre **arquivo + linha + descrição + correção sugerida**.

#### 4.1 Padrões Gerais de Código

- **Idioma do código**: tudo em inglês (variáveis, funções, structs, comentários)
- **Nomenclatura clara**: identificadores exportados descritivos; nomes curtos apenas em escopos pequenos (Go permite `i`, `r`, `err` localmente)
- **Constantes**: sem números mágicos — usar `const` ou `iota`
- **Funções**: uma única ação clara, verbos para mutações (`CreateOrder`, `UpdateUser`)
- **Parâmetros**: > 3-4 parâmetros indica necessidade de option struct ou functional options
- **Condicionais**: sem aninhamento profundo, **early returns** e **guard clauses**
- **Tamanho de método**: lógica complexa < 50 linhas; se ultrapassar, extrair helper
- **Tamanho de arquivo**: responsabilidade única; dividir quando exceder ~500 linhas
- **Comentários**: identificadores exportados **DEVEM** ter doc comments (`// FunctionName does...`); evitar comentários que repetem o código

#### 4.2 Go Idioms (Effective Go + Code Review Comments)

> Referências: [Effective Go](https://go.dev/doc/effective_go), [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments). Use [[go-best-practices]] para detalhes.

| Critério | Verificar |
|---|---|
| **Nomenclatura** | `MixedCaps`/`mixedCaps`; **nunca** `snake_case` para identificadores Go |
| **Acrônimos** | `HTTPClient`, `ID`, `URL` (totalmente em maiúsculas) — **não** `HttpClient`, `Id` |
| **Pacotes** | curto, lowercase, palavra única; sem `_` ou `mixedCaps`; evitar `util`, `common`, `helper` |
| **Receivers** | 1-2 letras, consistentes entre métodos do mesmo tipo; **nunca** `this`/`self` |
| **Interfaces de método único** | terminam em `-er` (`Reader`, `Writer`, `Stringer`) |
| **Sentinel errors** | `ErrSomething` (ex.: `ErrNotFound`) |
| **Error types** | terminam em `Error` (ex.: `ValidationError`) |
| **Não-exportado por padrão** | exportar apenas API pública |
| **Zero values** | aproveitar zero values significativos; evitar `new()` quando zero value é útil |
| **Composição sobre herança** | usar embedding, não hierarquias profundas |
| **Aceitar interfaces, retornar structs** | exceto construtores que retornam interface por contrato do projeto |
| **Interfaces pequenas** | 1-2 métodos; compor maiores via embedding |
| **Getter sem prefixo `Get`** | campo `owner` → getter `Owner()`, não `GetOwner()` |

#### 4.3 Tratamento de Erros

| Regra | Como Verificar |
|---|---|
| Sempre verificar erros | Sem `data, _ := ...` (exceto com comentário justificando) |
| Wrap com contexto | `fmt.Errorf("operation: %w", err)` |
| Sentinel errors | `var ErrNotFound = errors.New("not found")` + `errors.Is(err, ErrNotFound)` |
| Custom error types | `errors.As(err, &target)` quando precisa de contexto extra |
| Sem `panic` em biblioteca | `panic` apenas em `main` para estados irrecuperáveis (ou em `init`) |
| Mensagens de erro | lowercase, sem pontuação no final, descrevem o que falhou |
| Não logar e retornar | escolha **logar na boundary** OU **retornar**, nunca ambos |

#### 4.4 Concorrência

| Regra | Como Verificar |
|---|---|
| Gerenciamento de ciclo de vida | toda goroutine tem `context.Context`, `sync.WaitGroup` ou `errgroup.Group` |
| Channels sobre memória compartilhada | preferir channels quando apropriado |
| Estado compartilhado protegido | `sync.Mutex`/`sync.RWMutex` com seções críticas pequenas |
| Propagação de contexto | `ctx context.Context` é o **primeiro** parâmetro; nunca armazenar em structs |
| Sem goroutine leaks | toda goroutine tem caminho de saída claro |
| `select` com `ctx.Done()` | tratar cancelamento em goroutines de longa duração |
| Race-free | `go test -race ./...` passa sem detecções |

#### 4.5 Estrutura do Projeto (Clean Architecture, quando adotada)

| Camada | Pode importar |
|---|---|
| `internal/domain/` | apenas stdlib |
| `internal/application/` | apenas `internal/domain/` |
| `internal/infrastructure/` | `internal/domain/`, `pkg/`, libs externas |
| `pkg/` | **nunca** `internal/` |
| `cmd/<app>/main.go` | tudo (composição do grafo FX) |

Validar também:

- Pacotes orientados a **domínio/feature**, não por camada técnica
- Sem imports circulares (Go aplica isso em compilação, mas verificar a **direção limpa**)
- Construtores via DI (uber-go/fx ou injeção manual), sem estado global
- Convenções de nomenclatura de arquivos: kebab-case (`order-repository.go`)
- Estrutura de subpastas conforme `.claude/rules/architecture.md`

#### 4.6 REST/HTTP (Fiber v3, quando aplicável)

> Referência: `.claude/rules/fiber-v3.md` quando disponível.

| Critério | Verificar |
|---|---|
| Versão | `github.com/gofiber/fiber/v3` |
| Handler signature | `func(c fiber.Ctx) error` (não `*fiber.Ctx`) |
| Recursos | inglês, plural, kebab-case (`/orders`, `/payment-methods`) |
| Profundidade de path | máximo 3 níveis |
| Status codes | constantes `fiber.Status*` (não números mágicos) |
| Binding | `c.Bind().Body(&input)` ao invés de `json.Unmarshal(c.Body(), &v)` |
| Validação | tags `validate` + `StructValidator` registrado no `fiber.Config` |
| `c.Context()` | sempre extrair antes de passar para camadas internas ou goroutines |
| Zero-allocation | copiar `c.Body()`, `c.Params()`, `c.Query()` antes de persistir além do handler |
| Middlewares | `recover` registrado primeiro; CORS, helmet, requestid configurados |
| Timeout middleware | aplicado por rota, **nunca** via `app.Use()` |
| Cookies | `HTTPOnly: true`, `Secure: true`, `SameSite` definido |
| Graceful shutdown | `app.Shutdown()` no hook `OnStop` do fx |
| Anotações Swagger | swaggo em todos os handlers públicos |
| Controllers | implementam `RegisterRoutes(app *fiber.App)`; lógica delegada ao use case |

#### 4.7 Logging (slog, quando adotado)

- Logging estruturado em JSON via `slog`
- Levels: `DEBUG` (dev), `INFO` (eventos notáveis), `WARN` (recuperável), `ERROR` (falha)
- Nunca arquivos — usar stdout/stderr
- Nunca logar dados sensíveis (senhas, tokens, PII, CPF, cartão)
- Mensagens claras + campos de contexto (`slog.String("orderID", id)`)
- Nunca silenciar erros (`_ = err`)
- Incluir `request_id`/`trace_id` para rastreabilidade

#### 4.8 Banco de Dados (MongoDB driver v2, quando adotado)

- Import: `go.mongodb.org/mongo-driver/v2/mongo`
- Cursors fechados com `defer cursor.Close(ctx)`
- Queries parametrizadas — **nunca** concatenação de strings em filtros
- Tratamento explícito de `mongo.ErrNoDocuments`
- Transactions para operações multi-step quando consistência for crítica
- Connection pool configurado
- Repositórios retornam **interfaces de domínio**, não tipos concretos

#### 4.9 Mensageria (RabbitMQ amqp091-go, quando adotado)

- Publishers implementam interfaces de evento do domínio (`event.OrderEvent`)
- Consumers delegam para use cases — **sem lógica de negócio no consumer**
- Ack/Nack adequado: `msg.Ack(false)` em sucesso, `msg.Nack(false, requeue)` em falha
- Tratamento de erros de desserialização (logar + Nack sem requeue se permanente)
- Lifecycle gerenciado via fx (`OnStart`/`OnStop`)
- Reconexão automática quando aplicável

#### 4.10 Injeção de Dependência (uber-go/fx, quando adotado)

- Providers registrados no `main.go` de cada binário
- `config.InfraModule` **sempre primeiro**
- Cada binário fornece **apenas o que precisa** — uma API não fornece consumer, e vice-versa
- Lifecycle hooks (`OnStart`/`OnStop`) para startup e shutdown graceful
- Construtores seguem convenção:
  - Repos/publishers/use cases → **interface de domínio**
  - Controllers/consumers/entidades → **ponteiro concreto**

#### 4.11 Testes

> Referência completa: [[go-testing-best-practices]].

| Critério | Verificar |
|---|---|
| Framework | pacote padrão `testing` |
| Independência | testes podem rodar em qualquer ordem |
| Paralelismo | `t.Parallel()` quando seguro |
| Table-driven | `tests := []struct{...}` + `t.Run(tc.name, ...)` |
| Subtests | `t.Run("description", func(t *testing.T) {...})` |
| Naming | arquivos `*_test.go`; pacote `_test` para black-box quando útil |
| Asserções | `testify` (`assert`/`require`) se adotado; senão `if got != want { t.Errorf(...) }` |
| Mocks | gerados via `gomock` ou manuais — sem monkey-patching |
| HTTP | `httptest.NewRecorder` / `httptest.NewServer` para handlers |
| Helpers | `t.Helper()` em funções helper |
| Edge cases | input vazio, nil pointers, error paths, valores no limite |
| Race detector | `go test -race ./...` limpo |
| Cobertura | verificar com `go test -cover`; código novo coberto |
| Benchmarks | `Benchmark*` em caminhos críticos quando relevante |

### Etapa 5: Verificação Automatizada

Execute todas as ferramentas e registre saídas para anexar ao artefato:

```bash
# 1. Compilação — bloqueante
go build ./...

# 2. Vet — construções suspeitas
go vet ./...

# 3. Testes com race detector e cobertura
go test -race -count=1 -coverprofile=coverage.out ./...
go tool cover -func=coverage.out | tail -1

# 4. Linter (se disponível no projeto)
golangci-lint run ./...

# 5. Formatação
gofmt -l .

# 6. Imports organizados (se goimports disponível)
goimports -l .

# 7. go.mod limpo
go mod tidy -diff
go mod verify
```

**Regras:**

- Qualquer falha em `go build` → **Crítico** automático
- Qualquer detecção em `go vet` → **Major** (ou Crítico se for race/concorrência)
- Qualquer teste falhando → **Crítico**
- Race detector detectou → **Crítico**
- Arquivos listados em `gofmt -l .` → **Major**
- `golangci-lint` issues → classificar por severidade do linter
- Cobertura abaixo do threshold do projeto (típico 80%) → **Major**

### Etapa 6: Classificar Problemas

Cada problema encontrado recebe **uma** classificação:

#### **CRÍTICO** (bloqueante para merge)

- Bugs funcionais
- Problemas de segurança (SQL injection, XSS, exposição de credenciais, JWT mal validado)
- Funcionalidade quebrada (não compila, testes falham)
- **Goroutine leaks**
- **Race conditions**
- Erros retornados não verificados
- `panic` em código de biblioteca
- Violações da regra de dependência da Clean Architecture
- Logging de dados sensíveis (PII, senhas, tokens)
- `c.Body()`/`c.Params()` armazenados sem copy em handler Fiber

#### **MAJOR** (deve ser corrigido)

- Violações de idiomas Go (`snake_case`, `GetX()`, acrônimos errados)
- Padrões do projeto desrespeitados (construtor com retorno errado, arquivo em pasta errada)
- Testes ausentes para código novo
- Nomenclatura inadequada (nomes genéricos, abreviações ruins)
- Contexto de erro ausente (sem `%w`)
- Identificadores exportados sem doc comment
- Uso incorreto de uber-go/fx (provider faltando, ordem errada)
- Cobertura significativamente abaixo do threshold
- `gofmt`/`goimports` não aplicados
- Mensagens de erro com Maiúscula inicial ou pontuação final
- Cursor MongoDB sem `defer Close`

#### **MINOR** (sugestão)

- Sugestões de estilo
- Otimizações opcionais
- Padrões não-idiomáticos que ainda funcionam
- Comentários redundantes
- Variáveis com nomes pouco descritivos em escopo amplo
- Funções um pouco longas mas ainda coesas

#### **POSITIVO** (reconhecer)

- Go idiomático bem aplicado
- Boa cobertura de testes incluindo edge cases
- Tratamento de erros consistente com `%w` e `errors.Is/As`
- Uso correto de `context.Context` em toda a pilha
- Interfaces pequenas e focadas
- Estrutura aderente à Clean Architecture
- Doc comments completos
- Uso correto de `t.Parallel()`, `t.Helper()`, table-driven tests
- Graceful shutdown bem implementado

### Etapa 7: Gerar/Atualizar o Artefato `[num]_task_review.md`

Crie ou **sobrescreva** o arquivo no **mesmo diretório** onde `[num]_task.md` está localizado.

#### Formato Obrigatório

```markdown
# Revisão: Task [num] - [Título da Task]

**Revisor**: AI Go Code Reviewer
**Data**: [AAAA-MM-DD]
**Arquivo da task**: [num]_task.md
**Status**: [APROVADO | APROVADO COM OBSERVAÇÕES | MUDANÇAS SOLICITADAS]
**Re-revisão**: [Não | Sim (após correções)]

## Resumo

[Resumo breve do que foi implementado e avaliação geral de qualidade]

## Skills Técnicas Utilizadas

| Skill | Utilizada | Observações |
|-------|-----------|-------------|
| go-best-practices | [Sim/Não/N/A] | [Breve nota] |
| go-testing-best-practices | [Sim/Não/N/A] | [Breve nota] |

## Arquivos Revisados

| Arquivo | Status | Problemas |
|---------|--------|-----------|
| [caminho do arquivo] | [OK / Problemas / Crítico] | [contagem] |

## Verificações Automatizadas

| Verificação | Comando | Resultado | Detalhes |
|---|---|---|---|
| Build | `go build ./...` | OK / FALHOU | ... |
| Vet | `go vet ./...` | OK / FALHOU | ... |
| Testes | `go test -race ./...` | OK / FALHOU | ... |
| Cobertura | `go test -cover ./...` | [X]% | threshold: [Y]% |
| Lint | `golangci-lint run` | OK / FALHOU / N/A | ... |
| Formatação | `gofmt -l .` | OK / FALHOU | ... |

## Problemas Encontrados

### Problemas Críticos

[Liste cada problema crítico com: arquivo, linha, descrição e correção sugerida com exemplo de código]
[Se nenhum: "Nenhum problema crítico encontrado."]

### Problemas Major

[Mesmo formato]

### Problemas Minor

[Mesmo formato]

## Destaques Positivos

[Liste o que foi bem feito — Go idiomático, cobertura, design limpo, etc.]

## Conformidade com Padrões

| Padrão | Status |
|--------|--------|
| Go Idioms & Effective Go | [OK / Atenção / Crítico] |
| Tratamento de Erros | [OK / Atenção / Crítico] |
| Concorrência | [OK / Atenção / Crítico / N/A] |
| Estrutura do Projeto (Clean Architecture) | [OK / Atenção / Crítico] |
| REST/HTTP (Fiber) | [OK / Atenção / Crítico / N/A] |
| Logging (slog) | [OK / Atenção / Crítico / N/A] |
| Banco de Dados (MongoDB) | [OK / Atenção / Crítico / N/A] |
| Mensageria (RabbitMQ) | [OK / Atenção / Crítico / N/A] |
| Injeção de Dependência (uber-go/fx) | [OK / Atenção / Crítico / N/A] |
| Testes | [OK / Atenção / Crítico] |
| Code Quality Tools | [OK / Atenção / Crítico] |

## Recomendações

[Lista numerada de recomendações priorizadas para melhoria]

## Veredicto

[Avaliação final com próximos passos claros. Se status ≠ APROVADO, deixe explícito que correções devem ser aplicadas e o task-reviewer executado novamente para atualizar este arquivo até obter APROVADO.]
```

## Critérios de Status

| Status | Quando Usar |
|---|---|
| **APROVADO** | Sem problemas críticos ou major. Código pronto para merge. **Único status que finaliza a revisão.** |
| **APROVADO COM OBSERVAÇÕES** | Sem problemas críticos; problemas minor ou poucos majors não-bloqueantes. Requer correções e nova execução do reviewer até obter APROVADO. |
| **MUDANÇAS SOLICITADAS** | Problemas críticos OU múltiplos majors bloqueantes. Requer correções e nova execução do reviewer. |

## Re-revisão Após Correções

Quando o workflow `run-task.md` aplica correções após **APROVADO COM OBSERVAÇÕES** ou **MUDANÇAS SOLICITADAS**, este reviewer será invocado novamente:

1. O arquivo `[num]_task_review.md` pode já existir — **trate como re-revisão**
2. Execute a revisão completa sobre o código **atual** (com as correções aplicadas)
3. **Sobrescreva** `[num]_task_review.md` com a nova avaliação — não deixe o status antigo no lugar
4. No cabeçalho, marque `**Re-revisão**: Sim (após correções)`
5. Defina **APROVADO** apenas quando todos os problemas bloqueantes estiverem resolvidos; caso contrário use APROVADO COM OBSERVAÇÕES ou MUDANÇAS SOLICITADAS para que o ciclo continue

Fluxo esperado quando status ≠ APROVADO:

1. Desenvolvedor (ou agente de correção) aplica fixes
2. Reviewer é executado novamente
3. `[num]_task_review.md` é **sobrescrito** com novo resultado
4. Repetir até **APROVADO**

## Diretrizes Operacionais

1. **Ler antes de julgar** — leia integralmente os arquivos modificados, não apenas o diff
2. **Seja específico** — sempre referencie arquivo e número da linha
3. **Forneça solução** — não apenas aponte; sugira correção com exemplo de código
4. **Verifique testes** — código novo sem teste correspondente é **Major** no mínimo
5. **Execute as verificações** — `go build`, `go vet`, `go test -race`, `gofmt`, `golangci-lint`
6. **Cheque contra requisitos** — o implementado corresponde ao solicitado em `[num]_task.md`?
7. **Aplique [[go-best-practices]]** para validar idiomas Go
8. **Aplique [[go-testing-best-practices]]** para validar testes
9. **Aplique as regras do projeto** (`.claude/rules/architecture.md`, `.claude/rules/fiber-v3.md`, `CLAUDE.md`)
10. **Reconheça o bom** — destaques positivos são parte do artefato
11. **Re-revisão sobrescreve** — não acumule reviews; o arquivo sempre reflete o estado mais recente
12. **Idioma** — artefato em Português (BR); exemplos de código em inglês
13. **Seja justo** — revisão é construtiva; padrões existem para ajudar, não para humilhar
14. **Atualize memória** — registre padrões recorrentes do projeto para revisões futuras

## Anti-Patterns de Task Review (Evitar)

| Anti-Pattern | Correto |
|---|---|
| "LGTM 👍" sem revisar | Toda revisão produz artefato `[num]_task_review.md` |
| Revisar apenas o diff (sem ler arquivos completos) | Ler contexto completo dos arquivos modificados |
| "Tudo certo" sem rodar testes | Sempre executar `go build`, `go vet`, `go test -race` |
| Apontar problema sem solução | Sempre sugerir correção com exemplo de código |
| Ignorar testes ausentes | Código novo sem teste = Major |
| Não classificar problemas | Toda observação tem severidade explícita (Crítico/Major/Minor/Positivo) |
| Aprovar com testes falhando | Falha de teste = Crítico, MUDANÇAS SOLICITADAS |
| Aprovar com `go build` quebrado | Build falha = Crítico imediato |
| Manter review antigo após correções | Re-revisão **sobrescreve** o arquivo |
| Reportar status sem rastreabilidade | Cada decisão tem referência a arquivo:linha + comando executado |
| Importar padrões de outro projeto | Validar contra os padrões **deste** projeto (`.claude/rules/`, `CLAUDE.md`) |
| Só apontar erros, ignorar acertos | Sempre incluir seção "Destaques Positivos" |
| Esquecer race detector | Sempre `go test -race ./...` |
| Aceitar `panic` em código de biblioteca | `panic` apenas em `main` ou `init`, casos terminais |
| Aceitar erro logado E retornado | Logar **na boundary** OU retornar — nunca ambos |

## Cheat Sheet — Comandos do Reviewer

```bash
# Identificar tarefa
find . -name "*_task.md" -type f | xargs ls -lt | head -5
ls -la <task_dir>/*_task_review.md 2>/dev/null   # detectar re-revisão

# Identificar arquivos alterados
git diff main...HEAD --stat
git log main...HEAD --oneline
git status --short
git diff HEAD --name-only

# Ler diff completo de um arquivo
git diff main...HEAD -- internal/application/usecase/order/create-usecase.go

# Verificações automatizadas
go build ./...
go vet ./...
go test -race -count=1 -coverprofile=coverage.out ./...
go tool cover -func=coverage.out
go tool cover -func=coverage.out | tail -1            # cobertura total
golangci-lint run ./...
gofmt -l .
goimports -l .
go mod tidy -diff
go mod verify

# Testes específicos do diff
go test -race -run TestOrderCreate ./internal/application/usecase/order/

# Identificar handlers HTTP modificados
git diff main...HEAD --name-only -- 'internal/infrastructure/controller/*'

# Identificar testes
git diff main...HEAD --name-only -- '*_test.go'

# Re-gerar swagger e comparar (se Fiber + swaggo)
swag init -g cmd/api/main.go --parseDependency --parseInternal
git diff docs/swagger.yaml
```

## Idioma

Artefato de revisão em **Português (Brasil)**. Exemplos de código permanecem em **inglês**.

## Manutenção de Memória do Revisor

Ao longo das revisões, registre padrões e violações recorrentes para construir conhecimento institucional sobre o codebase:

- Violações recorrentes entre tarefas (ex.: time se esquece de copiar `c.Body()`)
- Padrões arquiteturais consolidados no projeto (camadas, organização por contexto de domínio)
- Abordagens comuns de teste e lacunas frequentes
- Convenções de nomenclatura efetivamente em uso (vs. as documentadas)
- Dependências e libs adotadas
- Padrões de tratamento de erros consolidados
- Padrões de concorrência (errgroup, context propagation)
- Padrões de uso do uber-go/fx e violações recorrentes
- Endpoints/recursos mais propensos a regressões

Essas notas tornam revisões futuras mais rápidas e consistentes.
