---
name: go-best-practices
description: Aplica boas práticas oficiais do Go (go.dev/doc, Effective Go, Go Blog) ao escrever, revisar ou refatorar código Go. Use ao criar/editar arquivos .go, ao revisar código Go, ao discutir convenções de nomenclatura, tratamento de erros (errors.Is/As, %w), concorrência (goroutines, channels, sync), uso de context.Context, ou design de interfaces. Acione sempre que o trabalho envolver Go/Golang.
---

# Go Best Practices (Documentação Oficial)

Skill baseada exclusivamente na documentação oficial em https://go.dev/doc/ — Effective Go, Go Blog (Pipelines, Context, Error Handling) e Go 1.13 Errors. Todas as recomendações aqui devem ser seguidas ao escrever, revisar ou refatorar código Go.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer arquivo `.go`
- Ao discutir design, nomenclatura ou arquitetura de aplicações Go
- Ao tratar de concorrência, context, erros ou interfaces em Go
- Ao gerar exemplos de código Go em respostas

## 1. Convenções de Nomenclatura

Referência: [Effective Go — Names](https://go.dev/doc/effective_go#names)

### Pacotes
- Nome em **minúsculas**, **uma palavra só**, **curto** e **conciso**. Sem underscore. Sem `mixedCaps`.
- O nome do pacote é o nome base do diretório: `src/encoding/base64` → import `"encoding/base64"` → nome `base64`.
- Evite nomes genéricos como `util`, `common`, `helpers`. Prefira nomes que descrevam o que o pacote faz (`bufio`, `strconv`, `httputil`).
- Clientes usam `pacote.Identificador` — não repita o nome do pacote no identificador: use `bufio.Reader`, **não** `bufio.BufReader`.

### Getters e Setters
- **Omita o prefixo `Get`**. Para um campo `owner` (unexported), o getter é `Owner()`, não `GetOwner()`.
- Setter mantém o prefixo: `SetOwner(...)`.

```go
owner := obj.Owner()
if owner != user {
    obj.SetOwner(user)
}
```

### Interfaces
- Interface de **um método** é nomeada pelo método + sufixo `-er`: `Reader`, `Writer`, `Formatter`, `CloseNotifier`.
- Use **assinaturas canônicas** quando seu método tiver o mesmo significado de um conhecido: `Read`, `Write`, `Close`, `Flush`, `String`. Não invente nomes diferentes para o mesmo conceito.

### MixedCaps
- Use `MixedCaps` (exported) ou `mixedCaps` (unexported). **Nunca** use `snake_case` para identificadores multi-palavra.
- `ServeHTTP`, não `Serve_HTTP`. `userID`, não `user_id`.

### Acrônimos
- Acrônimos mantêm o caso consistente: `URL`, `HTTP`, `ID`. Use `ServeHTTP` e `userID`, não `ServeHttp` ou `userId`.

## 2. Tratamento de Erros

Referência: [Error handling and Go](https://go.dev/blog/error-handling-and-go) + [Go 1.13 Errors](https://go.dev/blog/go1.13-errors)

### A interface `error`
```go
type error interface {
    Error() string
}
```

### Sempre verifique erros explicitamente
- Go não tem exceptions. Verifique o erro onde ele ocorre.
- Padrão idiomático: error é o **último valor de retorno**.

```go
f, err := os.Open(name)
if err != nil {
    return err
}
// fluxo de sucesso continua sem else
```

- Quando o `if` termina em `return`/`break`/`continue`/`goto`, **omita o `else`** — deixe o caminho feliz fluir verticalmente.

### Criando erros

| Função | Quando usar |
|--------|-------------|
| `errors.New("texto")` | Mensagem fixa, sem formatação |
| `fmt.Errorf("ctx: %v", err)` | Adicionar contexto **sem** expor o erro original |
| `fmt.Errorf("ctx: %w", err)` | Adicionar contexto **e** preservar a cadeia (wrap) |

### Sentinel errors
```go
var ErrNotFound = errors.New("item: not found")
```
- Defina como `var` no nível do pacote.
- **Documente** que funções podem retornar um erro que envelopa esse sentinel.
- Retorne sempre **wrapped** (`fmt.Errorf("%q: %w", name, ErrNotFound)`), nunca o sentinel direto — assim você força os callers a usarem `errors.Is`.

### Comparação e extração (Go 1.13+)

```go
// Comparar contra um sentinel — percorre toda a cadeia
if errors.Is(err, ErrNotFound) {
    // ...
}

// Extrair tipo concreto — percorre toda a cadeia
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    log.Printf("op=%s path=%s", pathErr.Op, pathErr.Path)
}
```

- **Nunca** compare erros com `==` (exceto `err != nil`). Use `errors.Is`.
- **Nunca** faça `err.(*MyErr)` direto. Use `errors.As`.

### Tipos de erro customizados
```go
type SyntaxError struct {
    Msg    string
    Offset int64
}

func (e *SyntaxError) Error() string { return e.Msg }
```

Implemente `Unwrap() error` quando o tipo embrulha outro erro:
```go
type QueryError struct {
    Query string
    Err   error
}

func (e *QueryError) Error() string { return e.Query + ": " + e.Err.Error() }
func (e *QueryError) Unwrap() error { return e.Err }
```

### Quando envelopar (`%w`) vs não envelopar (`%v`)

- **Envelope (`%w`)** quando o erro de origem é parte legítima da API pública e callers podem precisar inspecioná-lo.
- **Não envelope (`%v`)** quando o erro é detalhe de implementação. Embrulhar com `%w` torna o erro parte do contrato público — não embrulhe algo que você não está disposto a manter para sempre.

### Mensagens de erro
- **Comecem com minúscula**, sem ponto final (serão concatenadas).
- Identifiquem a origem: `"image: unknown format"`, não apenas `"unknown format"`.

## 3. Concorrência: Goroutines, Channels e `sync`

Referência: [Go Concurrency Patterns: Pipelines](https://go.dev/blog/pipelines)

### Goroutines
- Goroutines são baratas, mas **não infinitas**. Toda goroutine deve ter um caminho claro de término.
- **Goroutine leak** é bug: se você lança uma goroutine, garanta que ela termina (via `done`, `context`, fechamento de channel, etc.).

### Channels

**Unbuffered:**
```go
c := make(chan int)        // bloqueia até sender e receiver estarem prontos (sync)
```

**Buffered:**
```go
c := make(chan int, 2)     // só use quando souber o número exato de sends
```

Regras inegociáveis:
- **Quem envia fecha o channel**, nunca o receiver.
- **Nunca envie em channel fechado** (panic).
- **Múltiplos senders** precisam de coordenação (`sync.WaitGroup`) antes de qualquer close.
- Receber de channel fechado retorna o **zero value imediatamente** — útil para broadcast de cancelamento.

### Padrão Pipeline

Cada stage:
1. Recebe valores de channels de entrada.
2. Processa.
3. Envia para channels de saída.
4. **Fecha o channel de saída** quando termina (via `defer close(out)`).

```go
func sq(done <-chan struct{}, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-done:
                return
            }
        }
    }()
    return out
}
```

### Fan-out / Fan-in

Fan-in usa `sync.WaitGroup` para fechar o channel agregado **apenas após todos os senders terminarem**:

```go
func merge(done <-chan struct{}, cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    output := func(c <-chan int) {
        defer wg.Done()
        for n := range c {
            select {
            case out <- n:
            case <-done:
                return
            }
        }
    }
    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```

### Cancelamento explícito
- Sempre passe um `done <-chan struct{}` (ou, preferencialmente, `context.Context`) através do pipeline.
- Use `defer close(done)` no caller para garantir limpeza em todos os caminhos de saída.

### Bounded parallelism
- Limite goroutines concorrentes para controlar recursos:
```go
const workers = 20
var wg sync.WaitGroup
wg.Add(workers)
for i := 0; i < workers; i++ {
    go func() {
        defer wg.Done()
        worker(done, in, out)
    }()
}
go func() { wg.Wait(); close(out) }()
```

### Sincronização
- Channels para **comunicação**. `sync.Mutex`/`sync.RWMutex` para proteger **estado compartilhado**.
- Lema oficial: *"Don't communicate by sharing memory; share memory by communicating."* — mas use mutex quando ele for de fato a ferramenta certa (cache, contador, struct compartilhada).
- Sempre rode testes com `-race`.

## 4. `context.Context`

Referência: [Go Concurrency Patterns: Context](https://go.dev/blog/context)

### A interface
```go
type Context interface {
    Done() <-chan struct{}
    Err() error
    Deadline() (time.Time, bool)
    Value(key any) any
}
```

### Regras absolutas

1. **Context é o primeiro parâmetro**, chamado `ctx`:
   ```go
   func DoWork(ctx context.Context, arg Arg) (Result, error)
   ```
2. **Nunca passe `nil` context.** Em dúvida, use `context.TODO()`.
3. **Nunca armazene context em struct.** Passe-o explicitamente nas chamadas.
4. **Context values são para dados request-scoped** (request ID, user, trace ID) — **não** para parâmetros opcionais de função.
5. **Sempre `defer cancel()`** após `WithCancel`/`WithTimeout`/`WithDeadline`, mesmo que pareça redundante.

### Raízes
- `context.Background()`: raiz em `main`, `init`, testes, e topo de requests de entrada.
- `context.TODO()`: placeholder quando ainda não está claro qual context usar.

### Contexts derivados

```go
ctx, cancel := context.WithCancel(parent)
defer cancel()

ctx, cancel := context.WithTimeout(parent, 2*time.Second)
defer cancel()

ctx, cancel := context.WithDeadline(parent, deadline)
defer cancel()

ctx := context.WithValue(parent, userIPKey, ip)
```

### Propagação de cancelamento
```go
select {
case <-ctx.Done():
    return ctx.Err()
case res := <-resultCh:
    return res, nil
}
```

### Chaves de `WithValue`
- **Sempre use tipo unexported** para evitar colisões entre pacotes:
```go
package userip

type key int
const userIPKey key = 0

func NewContext(ctx context.Context, ip net.IP) context.Context {
    return context.WithValue(ctx, userIPKey, ip)
}

func FromContext(ctx context.Context) (net.IP, bool) {
    ip, ok := ctx.Value(userIPKey).(net.IP)
    return ip, ok
}
```
- **Nunca** use `string` ou tipo built-in como chave.
- Exponha helpers `NewContext` / `FromContext` ao invés de expor a chave.

### Context é thread-safe
- Um único `Context` pode ser passado para múltiplas goroutines e cancelado de uma vez para sinalizar todas.

### O que **não** vai em context
- Parâmetros obrigatórios da função (devem ser argumentos explícitos).
- Configurações estáticas da aplicação.
- Sessions de DB, conexões, loggers — passe-os explicitamente ou via struct do serviço.

## 5. Interfaces

Referência: [Effective Go — Interfaces](https://go.dev/doc/effective_go#interfaces_and_types)

### Pequenas e focadas
- O ideal são interfaces de **1 ou 2 métodos**. `io.Reader`, `io.Writer`, `fmt.Stringer` são modelo.
- Quanto menor a interface, mais reutilizável e fácil de implementar/mockar.

### Satisfação implícita
- Tipos **não declaram** que implementam uma interface — basta ter os métodos.
- Permita design "duck typing" verificado em compile time.

### Composição por embedding
```go
type ReadWriter interface {
    Reader
    Writer
}
```

### "Accept interfaces, return structs"
- **Parâmetros**: aceite a interface mais ampla/genérica possível (flexibilidade).
- **Retornos**: retorne o **tipo concreto** — dá mais informação ao caller e permite que ele decida qual abstração usar.
- Exceção: quando o tipo existe **apenas** para implementar a interface (ex.: `crypto/cipher.NewCTR` retorna `Stream`), aí retorne a interface.

### Defina interfaces no consumidor, não no produtor
- A interface pertence ao pacote que **usa**, não ao que **implementa**. Isso evita acoplamento desnecessário e dependências invertidas.

### Type assertions e type switches
```go
// "comma, ok" — sempre prefira a versão segura
if s, ok := v.(string); ok {
    use(s)
}

// type switch para múltiplos tipos
switch x := v.(type) {
case string:
    return x
case fmt.Stringer:
    return x.String()
default:
    return fmt.Sprintf("%v", x)
}
```
- Para checar erro: use `errors.As`, não type assertion direto.

### Interface vazia (`any`)
- Use `any` (alias de `interface{}` desde Go 1.18) **apenas** quando o tipo é genuinamente desconhecido (parsers, serializers).
- Em código de aplicação, `any` quase sempre é sintoma de design ruim — prefira generics ou interfaces específicas.

### Verificação em compile time
Para garantir que um tipo implementa uma interface:
```go
var _ io.Reader = (*MyReader)(nil)
```
Coloque essa linha próxima à definição do tipo — falha na compilação se o tipo deixar de implementar.

## Checklist de Revisão

Ao escrever ou revisar código Go, verifique:

- [ ] Nomes de pacotes em minúscula, curtos, sem underscore
- [ ] Getters sem prefixo `Get`
- [ ] Interfaces de um método com sufixo `-er`
- [ ] `MixedCaps`, nunca `snake_case`
- [ ] Acrônimos consistentes (`URL`, `ID`, `HTTP`)
- [ ] Erros verificados explicitamente, caminho feliz alinhado à esquerda
- [ ] `errors.Is` / `errors.As` ao invés de `==` ou type assertion
- [ ] `%w` apenas quando o erro envelopado faz parte da API pública
- [ ] Mensagens de erro em minúscula, com prefixo do pacote/operação
- [ ] Goroutines com caminho de término claro (sem leaks)
- [ ] Channels fechados pelo sender, nunca pelo receiver
- [ ] `defer close(out)` em stages de pipeline
- [ ] `context.Context` como primeiro parâmetro, chamado `ctx`
- [ ] `defer cancel()` após `WithCancel`/`WithTimeout`
- [ ] Chaves de `WithValue` com tipo unexported
- [ ] Context **não** armazenado em struct
- [ ] Interfaces pequenas, definidas no consumidor
- [ ] "Accept interfaces, return structs"
- [ ] Testes rodados com `-race`

## Fontes Oficiais

Toda recomendação acima vem destas fontes — consulte-as quando houver dúvida:

- https://go.dev/doc/effective_go
- https://go.dev/blog/error-handling-and-go
- https://go.dev/blog/go1.13-errors
- https://go.dev/blog/pipelines
- https://go.dev/blog/context
- https://go.dev/doc/ (índice geral)
