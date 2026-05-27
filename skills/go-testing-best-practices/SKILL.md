---
name: go-testing-best-practices
description: Aplica melhores práticas oficiais de testes em Go cobrindo a pirâmide completa — unitários com stretchr/testify, integração com uber-go/mock (gomock), testes de aceitação orientados a comportamento e E2E com testcontainers-go. Use sempre que escrever, revisar ou refatorar arquivos *_test.go, ao discutir estratégia de testes, mocks, fixtures, containers de teste ou cobertura. Acione ao criar mocks com mockgen, ao montar suites de teste, ao subir dependências reais via testcontainers ou ao definir cenários de aceitação.
---

# Go Testing Best Practices (testify · uber-go/mock · testcontainers-go)

Skill baseada na documentação oficial:

- [pkg.go.dev/testing](https://pkg.go.dev/testing) e [go.dev/wiki/TableDrivenTests](https://go.dev/wiki/TableDrivenTests)
- [github.com/stretchr/testify](https://github.com/stretchr/testify) (`assert`, `require`, `suite`, `mock`)
- [go.uber.org/mock](https://github.com/uber-go/mock) (gomock + mockgen)
- [golang.testcontainers.org](https://golang.testcontainers.org/)

Toda recomendação aqui é a forma idiomática prescrita pelas fontes oficiais. Combine com [[go-best-practices]] para convenções gerais da linguagem.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer `*_test.go`
- Ao gerar mocks com `mockgen` ou estruturar suites
- Ao desenhar a estratégia de testes (unit, integração, aceitação, E2E)
- Ao subir dependências reais (DB, broker, HTTP) via testcontainers
- Ao discutir cobertura, fixtures, golden files ou `t.Parallel()`

## 0. A Pirâmide de Testes em Go

| Camada | Velocidade | Escopo | Ferramenta principal |
|--------|-----------|--------|----------------------|
| **Unitário** | ms | Função/struct isolada, sem I/O | `stretchr/testify` |
| **Integração** | ~100 ms | Vários componentes, dependências externas **mockadas** | `uber-go/mock` |
| **Aceitação** | ~s | Comportamento/regra de negócio do ponto de vista do usuário | `testify/suite` + linguagem ubíqua |
| **E2E** | s a min | Sistema inteiro com dependências **reais** em containers | `testcontainers-go` |

Regras inegociáveis:
- Muitos unitários → poucos integração → menos aceitação → mínimo de E2E.
- Cada camada acima é mais lenta e mais frágil. Suba o que **realmente** precisa subir.
- Toda camada usa `go test` — não há binário separado. Diferencie por **build tag** ou **convenção de pasta**, nunca por framework paralelo.

## 1. Convenções Gerais para `*_test.go`

### Layout de arquivos
- Testes vivem **no mesmo pacote** do código (`foo.go` ↔ `foo_test.go`). Use `package foo` para acesso a símbolos não exportados.
- Para testar apenas a API pública, use `package foo_test` (black-box). Permitido coexistir com o pacote interno no mesmo diretório.

### Build tags por camada
Separe execuções caras com build tags na primeira linha do arquivo:

```go
//go:build integration

package payments_test
```

Rode assim:
```bash
go test ./...                        # unitários
go test -tags=integration ./...      # integração
go test -tags=e2e ./...              # E2E (containers)
```

### Nomenclatura
- Função de teste: `func TestXxx(t *testing.T)` — `Xxx` começa em maiúscula.
- Subteste: `t.Run("descrição curta", func(t *testing.T) { ... })`.
- Benchmarks: `func BenchmarkXxx(b *testing.B)`. Fuzz: `func FuzzXxx(f *testing.F)`.
- Helpers: chame `t.Helper()` como **primeira linha** — relata o erro na linha do caller.

### `t.Cleanup` em vez de `defer`
- Use `t.Cleanup(fn)` para teardown registrado pelo helper: roda mesmo após `t.Fatal` e na ordem LIFO.
- `defer` no helper roda quando o **helper** retorna, não quando o teste termina. Para cleanup do teste, sempre `t.Cleanup`.

### Paralelismo
- `t.Parallel()` no topo do teste/subteste habilita execução concorrente.
- **Go ≥ 1.22**: variáveis de loop são por iteração — não precisa `tt := tt`.
- **Go < 1.22**: precisa `tt := tt` antes de `t.Run` em loop paralelo, senão closure captura a última iteração.
- `testify/suite` **não suporta** subtestes paralelos com state compartilhado — segregue suite por escopo.

### Falhas: `t.Errorf` vs `t.Fatalf`
- `t.Errorf` — marca como falho, **continua** o teste. Use quando faz sentido coletar mais info.
- `t.Fatalf` — marca como falho e **aborta** o teste. Use quando o resto depende do que falhou.
- Em `testify`: `assert.X` ≈ `Errorf`; `require.X` ≈ `Fatalf`.

### Mensagens de erro
Sempre inclua **got** e **want**:
```go
t.Errorf("Reverse(%q) = %q; want %q", in, got, want)
```

## 2. Testes Unitários com `stretchr/testify`

Referência oficial: https://github.com/stretchr/testify

### Pacotes da lib

| Pacote | Função |
|--------|--------|
| `assert` | Asserções que **continuam** após falha (boolean return) |
| `require` | Asserções que **param** o teste (chamam `t.FailNow`) |
| `suite` | Estrutura de suite com `SetupTest`/`TearDown` |
| `mock` | Mocks manuais (preferimos `uber-go/mock` para integração) |

### `assert` vs `require` — regra prática
- **Pré-condição** (objeto não-nil, sem erro de setup, conversão de tipo): `require`.
- **Verificação de saída** (valor, status, side effect): `assert`.

```go
func TestParseUser(t *testing.T) {
    u, err := ParseUser([]byte(`{"id":1,"name":"Ana"}`))
    require.NoError(t, err)   // sem isto, asserts abaixo causam nil deref
    require.NotNil(t, u)

    assert.Equal(t, 1, u.ID)
    assert.Equal(t, "Ana", u.Name)
}
```

### `require` **só** roda no goroutine principal do teste
`require.X` chama `t.FailNow` que chama `runtime.Goexit` — em uma goroutine secundária isso **não** para o teste, vaza a goroutine. Em código concorrente, use `assert` + canal de erros ou `t.Errorf`.

### Assertion object para múltiplas asserções
```go
a := assert.New(t)
a.Equal(123, got.ID)
a.NotNil(got.CreatedAt)
a.WithinDuration(time.Now(), got.CreatedAt, time.Second)
```

### Asserções essenciais (use estas; evite reinvenções)

| Cenário | Use |
|---------|-----|
| Igualdade | `Equal`, `EqualValues` (cast numérico), `Exactly` (mesmo tipo concreto) |
| Erro | `NoError`, `Error`, `ErrorIs`, `ErrorAs`, `ErrorContains` |
| Coleções | `Len`, `Empty`, `NotEmpty`, `Contains`, `ElementsMatch`, `Subset` |
| JSON | `JSONEq` (compara strings JSON desconsiderando ordem de chaves) |
| Tempo | `WithinDuration`, `WithinRange` |
| Pânico | `Panics`, `PanicsWithError`, `NotPanics` |
| Tipo | `IsType`, `Implements` |
| Eventual | `Eventually`, `Never` |

`ErrorIs` e `ErrorAs` percorrem a cadeia de `Unwrap()` — sempre prefira a `==` ou type assertion em testes.

### Table-driven com subtestes (idiomático)

```go
func TestReverse(t *testing.T) {
    tests := map[string]struct {
        in   string
        want string
    }{
        "vazio":           {"", ""},
        "ascii":           {"abc", "cba"},
        "emoji multibyte": {"a🎉b", "b🎉a"},
    }

    for name, tc := range tests {
        t.Run(name, func(t *testing.T) {
            t.Parallel()
            got := Reverse(tc.in)
            assert.Equal(t, tc.want, got)
        })
    }
}
```

Por que **map** em vez de slice:
- Nome do caso vira chave — auto-documenta.
- Iteração aleatória — força independência entre casos.
- `t.Run` aceita `/` no nome para filtrar: `go test -run TestReverse/emoji`.

### `testify/suite` — quando faz sentido
Use quando vários testes compartilham fixtures de **mesmo custo** (parser configurado, builder pré-preenchido). Para fixtures caras (DB, container), prefira `TestMain` ou helpers de pacote.

```go
type UserServiceSuite struct {
    suite.Suite
    svc *UserService
}

func (s *UserServiceSuite) SetupTest() {
    s.svc = NewUserService(InMemoryRepo())
}

func (s *UserServiceSuite) TestCreateUser_validInput_returnsID() {
    id, err := s.svc.Create(context.Background(), "ana@example.com")
    s.Require().NoError(err)
    s.NotEmpty(id)
}

func TestUserServiceSuite(t *testing.T) {
    suite.Run(t, new(UserServiceSuite))
}
```

Limitações oficiais:
- **Suite não suporta `t.Parallel()` entre seus testes** — eles rodam sequencialmente.
- `SetupTest` roda antes de **cada** teste; `SetupSuite` uma vez. Use o segundo para caro/imutável.

### O que **não** fazer com testify
- Não use `assert.True(t, x == y)` — perde mensagem de diagnóstico. Use `assert.Equal(t, y, x)`.
- Não use `testify/mock` para mocks novos — gere com `uber-go/mock` (typed, mais seguro).
- Não importe `assert` e `require` com alias confusos. `require` no goroutine principal, `assert` no resto — mantenha o padrão.
- Não compare `error` com `assert.Equal` — use `assert.ErrorIs`.

## 3. Testes de Integração com `uber-go/mock` (gomock)

Referência oficial: https://github.com/uber-go/mock e https://pkg.go.dev/go.uber.org/mock/gomock

`uber-go/mock` é o **fork mantido** do antigo `golang/mock` (arquivado em 2023). Use sempre `go.uber.org/mock`.

### Definição de "teste de integração" nesta skill
Verifica **a colaboração entre componentes do seu código** (use case ↔ repo ↔ adapter), com dependências externas (DB, HTTP, fila) **substituídas por mocks gerados**. Roda em ms, não precisa de container, mas exercita mais do que um único método.

E2E (camada acima) é que usa as dependências reais.

### Instalando `mockgen`

```bash
go install go.uber.org/mock/mockgen@latest
```

### Gerando mocks — duas formas, prefira directive

**Forma idiomática: `//go:generate` próximo da interface** (source mode):
```go
// repository.go
package payments

//go:generate mockgen -source=repository.go -destination=mocks/repository_mock.go -package=mocks

type Repository interface {
    FindByID(ctx context.Context, id string) (*Payment, error)
    Save(ctx context.Context, p *Payment) error
}
```

Roda com `go generate ./...`. Vantagens:
- Geração versionada junto do código.
- CI pode falhar se diff entre `go generate` e o repositório (detecta mock desatualizado).

### Flags essenciais do mockgen

| Flag | Uso |
|------|-----|
| `-source` | Caminho do arquivo Go com as interfaces |
| `-destination` | Onde escrever o mock |
| `-package` | Nome do pacote do mock (convenção: `mocks` ou `<pkg>_mock`) |
| `-typed` | **Recomendado**: gera EXPECT typed (compile-time safety nos retornos) |
| `-write_generate_directive` | Inclui `//go:generate` no topo do arquivo gerado |

Sempre use `-typed`. Sem ele, `Return(...)` aceita `any` e erros viram falhas em runtime.

### Estrutura padrão de um teste com gomock

```go
//go:build integration

package payments_test

import (
    "context"
    "errors"
    "testing"

    "github.com/stretchr/testify/require"
    "go.uber.org/mock/gomock"

    "myorg/payments"
    "myorg/payments/mocks"
)

func TestService_Charge_repoErrorBubblesUp(t *testing.T) {
    ctrl := gomock.NewController(t) // auto-Finish via t.Cleanup
    repo := mocks.NewMockRepository(ctrl)
    gw   := mocks.NewMockGateway(ctrl)

    wantErr := errors.New("db down")
    repo.EXPECT().
        FindByID(gomock.Any(), "p-1").
        Return(nil, wantErr).
        Times(1)

    svc := payments.NewService(repo, gw)

    _, err := svc.Charge(context.Background(), "p-1")

    require.ErrorIs(t, err, wantErr)
}
```

### Controller
- `gomock.NewController(t)` registra `Finish()` via `t.Cleanup` **automaticamente** quando `t` é `*testing.T`. Não chame `Finish()` manualmente.
- Um controller **por teste** — nunca compartilhe entre testes (vaza expectativas).
- Para cancelar contexto em fatal: `ctrl, ctx := gomock.WithContext(parentCtx, t)`.

### Configurando expectativas

```go
mock.EXPECT().
    Method(matcher1, matcher2).   // matchers para os args
    Return(val, nil).             // ou DoAndReturn(func(...) {...})
    Times(n)                      // ou MinTimes/MaxTimes/AnyTimes
```

| Quando o método deve... | Use |
|-------------------------|-----|
| Ser chamado exatamente N | `.Times(n)` |
| Não ser chamado | `.Times(0)` (ou simplesmente não declare a expectation) |
| Ser chamado qualquer número | `.AnyTimes()` |
| Validar argumento e retornar dinâmico | `.DoAndReturn(func(args) (...))` |
| Apenas executar efeito colateral | `.Do(func(args))` |
| Modificar arg out-parameter (ponteiro) | `.SetArg(i, value)` |

### Matchers oficiais

| Matcher | Para |
|---------|------|
| `gomock.Eq(v)` | Igualdade estrita |
| `gomock.Any()` | Qualquer valor — use **escassamente**, perde sinal |
| `gomock.Nil()` / `gomock.Not(...)` | Nulo / negação |
| `gomock.AssignableToTypeOf(zero)` | Type-check só (útil pra `context.Context`) |
| `gomock.Len(n)` | Coleção de tamanho `n` |
| `gomock.Cond(fn)` | Predicado custom — use quando precisar inspecionar struct |
| `gomock.InAnyOrder(slice)` | Coleção independente de ordem |
| `gomock.Regex(re)` | String contra regex |

**Regra crítica**: para `context.Context` use `gomock.AssignableToTypeOf(ctx)` ou `gomock.Any()` — comparar contexts com `Eq` quase sempre quebra (deadline difere).

Argumento de struct complexo? Capture com `Do`/`DoAndReturn` e asserte com `testify` em vez de matcher gigante:
```go
repo.EXPECT().
    Save(gomock.Any(), gomock.AssignableToTypeOf(&payments.Payment{})).
    DoAndReturn(func(_ context.Context, p *payments.Payment) error {
        assert.Equal(t, "USD", p.Currency)
        assert.True(t, p.Amount > 0)
        return nil
    })
```

### Ordenação de chamadas
**Padrão**: gomock **não** assume ordem. Cada `EXPECT` é um slot independente.

Para forçar ordem use `gomock.InOrder` (preferido por legibilidade):
```go
gomock.InOrder(
    repo.EXPECT().Begin(gomock.Any()),
    repo.EXPECT().Insert(gomock.Any(), gomock.Any()),
    repo.EXPECT().Commit(gomock.Any()),
)
```

Para dependência simples entre duas chamadas, `.After(prev)`. Mas `InOrder` é mais legível para 3+.

### O que **não** fazer com gomock
- Não use `gomock.Any()` para tudo — defeats o propósito do mock como contrato.
- Não chame `ctrl.Finish()` manualmente quando passou `*testing.T` ao controller.
- Não compartilhe mocks entre testes via `var` global — cada teste cria os seus.
- Não gere mocks de structs concretas — só de **interfaces**. Se precisar mockar concreto, refatore para interface no consumidor (ver [[go-best-practices]]).
- Não use o `testify/mock` em paralelo com `uber-go/mock` no mesmo projeto — escolha um e padronize.
- Não mude mock gerado à mão — sempre regenere.

### CI: detectar mock desatualizado
Adicione step:
```bash
go generate ./...
git diff --exit-code
```
Falha se algum `*_mock.go` ficou para trás da interface.

## 4. Testes de Aceitação

Testes de aceitação verificam **comportamento de negócio** descrito na linguagem do domínio — não a estrutura interna. São o ponto onde o teste valida "essa feature faz o que o usuário pediu".

Em Go não existe uma lib oficial canônica; `go test` + `testify` cobrem 90% dos casos sem trazer ferramentas novas. Use BDD-style (Given/When/Then) **na nomenclatura**, não em framework.

### Características de um bom teste de aceitação
- **Caixa preta**: usa apenas API pública do módulo/serviço.
- **Linguagem do domínio**: nomes refletem regras de negócio, não classes.
- **Independente de infra**: pode usar fakes/in-memory; só sobe container se a regra depende disso.
- **Um cenário por teste**: um único Given/When/Then. Múltiplas asserções OK, múltiplos Whens não.

### Estrutura recomendada (suite + subtests nomeados)

```go
package billing_test

type BillingAcceptanceSuite struct {
    suite.Suite
    app *billing.App   // composição completa com fakes
}

func (s *BillingAcceptanceSuite) SetupTest() {
    s.app = billing.NewApp(
        billing.WithRepo(fakes.NewInMemoryRepo()),
        billing.WithClock(fakes.FixedClock("2026-01-01T00:00:00Z")),
    )
}

func (s *BillingAcceptanceSuite) Test_FreeTrialUser_NotChargedOnFirstMonth() {
    // Given: usuário em trial gratuito
    user := s.app.CreateUser("ana@example.com", billing.PlanTrial)

    // When: ciclo de cobrança roda
    err := s.app.RunBillingCycle(context.Background())
    s.Require().NoError(err)

    // Then: nenhuma cobrança gerada
    charges := s.app.ChargesFor(user.ID)
    s.Empty(charges, "trial users must not be charged in month 1")
}

func TestBillingAcceptance(t *testing.T) {
    suite.Run(t, new(BillingAcceptanceSuite))
}
```

### Nomenclatura padrão
`Test_<Ator|Contexto>_<Ação>_<ResultadoEsperado>`

- ✅ `Test_FreeTrialUser_NotChargedOnFirstMonth`
- ✅ `Test_PaidUser_CancelsSubscription_PartialRefundIssued`
- ❌ `TestBilling1`, `TestRunBillingCycle` (descreve método, não regra)

### Fakes em vez de mocks
Mock (gomock) verifica **interações**. Fake é uma implementação alternativa que se comporta como a real (in-memory repo, clock fixo). Para aceitação prefira **fakes** — você se importa com o resultado, não com "salvou exatamente uma vez".

```go
type InMemoryRepo struct { m map[string]*Payment }
func (r *InMemoryRepo) FindByID(_ context.Context, id string) (*Payment, error) {
    p, ok := r.m[id]
    if !ok { return nil, payments.ErrNotFound }
    return p, nil
}
// ... etc
```

### Quando promover para E2E
- Se o cenário depende de comportamento real do Postgres (transação, lock, índice único), promova para E2E com testcontainers.
- Se depende de timing real de rede/fila, idem.
- Senão, fake é mais rápido, mais determinístico, e suficiente.

### Build tag
```go
//go:build acceptance
```
Roda separado: `go test -tags=acceptance ./...`. Mantém o ciclo de feedback rápido para unitários.

## 5. Testes E2E com `testcontainers-go`

Referência oficial: https://golang.testcontainers.org/

Sobe **dependências reais em containers** (Postgres, Kafka, Redis, MinIO, MySQL...) gerenciados pelo Docker, exercita o sistema ponta-a-ponta, descarta tudo no final.

### Quando E2E vale o custo
- Validar migrations contra o engine real.
- Verificar comportamento dependente de DB (transações, conflitos, full-text search).
- Smoke-test do binário compilado contra suas dependências reais.

E2E **não substitui** unitários/integração. É a cereja, não a base.

### Instalação
```bash
go get github.com/testcontainers/testcontainers-go
# Módulos específicos:
go get github.com/testcontainers/testcontainers-go/modules/postgres
```

### Padrão idiomático com módulos (recomendado)

Sempre prefira **módulos** pré-construídos (`modules/postgres`, `modules/kafka`, ...) a `GenericContainer` manual. Eles encapsulam wait strategy, env vars e helpers (`ConnectionString`).

```go
//go:build e2e

package orders_test

import (
    "context"
    "database/sql"
    "testing"
    "time"

    _ "github.com/jackc/pgx/v5/stdlib"
    "github.com/stretchr/testify/require"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/log"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
)

func startPostgres(t *testing.T) *sql.DB {
    t.Helper()
    ctx := context.Background()

    pg, err := postgres.Run(ctx,
        "postgres:16-alpine",
        postgres.WithDatabase("orders"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithLogger(log.TestLogger(t)),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).
                WithStartupTimeout(30*time.Second),
        ),
    )
    testcontainers.CleanupContainer(t, pg) // SEMPRE antes do require
    require.NoError(t, err)

    dsn, err := pg.ConnectionString(ctx, "sslmode=disable")
    require.NoError(t, err)

    db, err := sql.Open("pgx", dsn)
    require.NoError(t, err)
    t.Cleanup(func() { _ = db.Close() })

    return db
}
```

### Regra crítica: `CleanupContainer` **antes** do `require.NoError`

```go
ctr, err := postgres.Run(ctx, image, opts...)
testcontainers.CleanupContainer(t, ctr) // 1º
require.NoError(t, err)                  // 2º
```

Motivo: se a criação falhar parcialmente (container subiu mas readiness expirou), `ctr` ainda pode conter referência para limpar. Registrar cleanup **primeiro** garante que `t.Cleanup` derruba o container mesmo quando o teste aborta. `CleanupContainer` é nil-safe.

### Wait strategies — escolha pela natureza da dependência

| Estratégia | Use quando |
|-----------|-----------|
| `wait.ForLog(re)` | Serviço imprime linha de "ready" no stdout/stderr |
| `wait.ForListeningPort(port)` | Basta a porta abrir (TCP simples) |
| `wait.ForHTTP("/health")` | Serviço expõe healthcheck HTTP |
| `wait.ForSQL(...)` | DB precisa aceitar query |
| `wait.ForExec([]string{"cmd"})` | Comando exit 0 dentro do container |
| `wait.ForAll(s1, s2)` | Combinação |

Postgres específico: `wait.ForLog("ready to accept connections").WithOccurrence(2)` — o log aparece duas vezes durante init (init script + restart). Esperar 1 só dá flake.

Sempre defina `WithStartupTimeout` — sem ele o teste pendura indefinidamente se o image não baixar.

### `t.Parallel()` com containers
Funciona, mas **cada teste sobe seu próprio container** (custoso). Estratégias:

1. **Container por teste** — máximo isolamento, mais lento. OK para suíte pequena.
2. **Container por suite + schema por teste** — sobe um Postgres em `TestMain`, cada teste cria um schema/database isolado.
3. **Container compartilhado + transação por teste** — abre `BEGIN` em SetupTest, `ROLLBACK` em TearDown.

Padrão `TestMain`:
```go
var sharedDB *sql.DB

func TestMain(m *testing.M) {
    ctx := context.Background()
    pg, err := postgres.Run(ctx, "postgres:16-alpine", /* ... */)
    if err != nil { log.Fatalf("postgres up: %v", err) }
    defer func() { _ = pg.Terminate(ctx) }() // E2E em TestMain: terminate manual

    dsn, _ := pg.ConnectionString(ctx, "sslmode=disable")
    sharedDB, _ = sql.Open("pgx", dsn)

    os.Exit(m.Run())
}
```

⚠️ Em `TestMain` não há `t.Cleanup` — chame `Terminate(ctx)` em `defer`. **`os.Exit` não roda defers**, então capture o código e use:
```go
func TestMain(m *testing.M) {
    code := func() int {
        // setup com defers
        return m.Run()
    }()
    os.Exit(code)
}
```

### Logging integrado ao teste
Use `testcontainers.WithLogger(log.TestLogger(t))` como **primeira** opção — captura logs do container em `t.Log`, aparecem apenas em `-v` ou quando o teste falha. Sem isso, logs vão pro stderr global e poluem CI.

### Reuse de containers (experimental)
`testcontainers.WithReuseByName("name")` reutiliza um container já em execução. **Experimental** — API pode mudar. Útil para desenvolvimento local (não em CI), economiza tempo entre execuções.

### Network entre containers
Crie uma rede dedicada para que containers se enxerguem por hostname:
```go
net, err := network.New(ctx)
testcontainers.CleanupContainer(t, net)
require.NoError(t, err)

pg, _ := postgres.Run(ctx, "postgres:16-alpine",
    network.WithNetwork([]string{"db"}, net),
    /* ... */)

app, _ := myapp.Run(ctx, "myapp:latest",
    network.WithNetwork([]string{"app"}, net),
    /* app pode acessar pg via host "db" */
)
```

### O que **não** fazer com testcontainers
- **Não** rode E2E na mesma pipeline que unitários sem separar — use build tag (`//go:build e2e`) e job dedicado no CI.
- **Não** hardcode portas no host. Use `ctr.MappedPort(ctx, "5432/tcp")` ou `ConnectionString` do módulo.
- **Não** esqueça `WithStartupTimeout` na wait strategy. Default é generoso, mas em CI lento pode disfarçar problema real.
- **Não** chame `Terminate` manualmente quando passou `t` — `CleanupContainer(t, ctr)` já registrou via `t.Cleanup`.
- **Não** suba container em `init()` ou `var`. Ciclo de vida pertence ao teste/TestMain.
- **Não** use E2E para testar lógica que poderia ser unitário. Cada E2E adiciona segundos no CI.

## 6. Convenções Transversais

### Estrutura de pastas recomendada
```
payments/
├── service.go
├── service_test.go              // unitário, package payments
├── repository.go
├── repository_test.go           // unitário
├── service_integration_test.go  // //go:build integration
├── acceptance_test.go           // //go:build acceptance
├── e2e_test.go                  // //go:build e2e
└── mocks/
    └── repository_mock.go       // gerado por mockgen
```

### Coverage por camada
```bash
go test -race -cover ./...                              # unit
go test -race -cover -tags=integration ./...            # integration
go test -tags=e2e -timeout 5m ./...                     # E2E
```
Sempre `-race` em unitário/integração. Em E2E o `-race` ajuda mas pode mascarar timeout de container.

### `t.Helper()` em **todo** helper de teste
Sem `t.Helper()`, a linha do erro aponta para dentro do helper, não para o teste. Primeira linha do helper, sempre.

```go
func newPaymentForTest(t *testing.T, amount int) *Payment {
    t.Helper()
    p, err := NewPayment(amount, "USD")
    require.NoError(t, err)
    return p
}
```

### Golden files para output complexo
Para output grande (JSON, HTML, SQL gerado), use golden file + flag `-update`:
```go
var update = flag.Bool("update", false, "update golden files")

func TestRender(t *testing.T) {
    got := Render(input)
    golden := filepath.Join("testdata", t.Name()+".golden")
    if *update {
        require.NoError(t, os.WriteFile(golden, got, 0644))
    }
    want, err := os.ReadFile(golden)
    require.NoError(t, err)
    assert.Equal(t, string(want), string(got))
}
```
Pasta `testdata/` é ignorada pelo Go tooling (não compilada, não vista por `go list`).

### Não teste a implementação, teste o comportamento
- ❌ "verifica que `repo.Save` foi chamado 1x com `user.UpdatedAt` setado"
- ✅ "depois de `Update`, `FindByID` retorna o novo nome"

Mocks são valiosos onde I/O custaria; mas se a regra cabe em fake/in-memory, prefira — o teste sobrevive a refactors.

## 7. Checklist de Revisão

Ao escrever ou revisar testes Go, verifique:

### Geral
- [ ] `t.Helper()` na primeira linha de cada helper
- [ ] `t.Cleanup` em vez de `defer` para teardown registrado em helper
- [ ] `t.Parallel()` em unitários puros (e nos subtests também)
- [ ] Build tag separando camadas (`integration`, `acceptance`, `e2e`)
- [ ] Mensagem de erro inclui got e want
- [ ] `-race` rodado no CI

### Unitários (testify)
- [ ] `require` para pré-condições, `assert` para verificações
- [ ] `require` **só** no goroutine principal
- [ ] `ErrorIs`/`ErrorAs` em vez de comparação direta de erros
- [ ] Table-driven com map (chave = nome do caso) sempre que houver ≥ 3 casos
- [ ] Sem `assert.True(t, x == y)` — use `Equal`
- [ ] Suite só quando fixtures justificam (não suporta paralelo entre seus testes)

### Integração (gomock)
- [ ] `mockgen` com flag `-typed`
- [ ] `//go:generate` próximo à interface, regenerado em CI
- [ ] `gomock.NewController(t)` — sem `Finish()` manual
- [ ] Um controller por teste
- [ ] `gomock.AssignableToTypeOf` para `context.Context`, não `Eq`
- [ ] Struct complexa: capture com `DoAndReturn` + assert, não matcher gigante
- [ ] `InOrder` quando ordem importa; default é não-ordenado
- [ ] Mocks só de **interfaces** — refatore concreto para interface no consumidor

### Aceitação
- [ ] Nome `Test_<Ator>_<Ação>_<Resultado>` em linguagem do domínio
- [ ] Caixa preta — só API pública
- [ ] Fakes em vez de mocks quando possível
- [ ] Um Given/When/Then por teste
- [ ] Build tag `acceptance`

### E2E (testcontainers)
- [ ] Módulo pré-construído quando existe; `GenericContainer` só se necessário
- [ ] `CleanupContainer(t, ctr)` **antes** do `require.NoError`
- [ ] Wait strategy explícita com `WithStartupTimeout`
- [ ] `log.TestLogger(t)` como primeira `With...` option
- [ ] `MappedPort`/`ConnectionString` — nunca porta hardcoded
- [ ] Build tag `e2e`, job dedicado no CI
- [ ] Se compartilhar container via `TestMain`, isolamento via schema/transação

## 8. Fontes Oficiais

- https://pkg.go.dev/testing
- https://go.dev/wiki/TableDrivenTests
- https://github.com/stretchr/testify
- https://pkg.go.dev/github.com/stretchr/testify
- https://github.com/uber-go/mock
- https://pkg.go.dev/go.uber.org/mock/gomock
- https://golang.testcontainers.org/
- https://golang.testcontainers.org/features/common_functional_options/
- https://golang.testcontainers.org/modules/

Combine sempre com [[go-best-practices]] para convenções gerais da linguagem (nomenclatura, erros, contexto, interfaces).
