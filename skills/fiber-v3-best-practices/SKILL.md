---
name: fiber-v3-best-practices
description: Aplica melhores práticas oficiais do Fiber v3 (docs.gofiber.io/next) ao escrever, revisar ou refatorar APIs HTTP em Go. Cobre inicialização do `fiber.App` e `Config`, roteamento (parâmetros, constraints, grupos, `RouteChain`, domain routing, HEAD auto-registrado), `fiber.Ctx` como interface (handler `func(c fiber.Ctx) error`), zero-allocation gotchas (cópia de strings/bytes, `Immutable`), binding unificado (`Bind().Body/URI/Query/Header/Cookie/Form/All`) com tags `json/uri/query/form/header/cookie`, validação com `StructValidator` + `go-playground/validator`, error handling centralizado (`ErrorHandler`, `fiber.Error`, `fiber.NewError`), middlewares oficiais (Recover, Logger, RequestID, CORS, CSRF, Session, Timeout, Compress, BasicAuth, EncryptCookie, SSE, Cache, Proxy), acesso a dados de middleware via context keys tipados (`requestid.FromContext`, `csrf.TokenFromContext`), hooks de ciclo de vida (`OnRoute`, `OnListen`, `OnPreShutdown`, `OnPostShutdown`, `OnPreStartupMessage`), `fiber.Service` para dependências, `State`/`SharedState`, templates (`Views`, `Render`, `ViewBind`, `ReloadViews`), graceful shutdown via `app.Shutdown`/`ShutdownWithContext`, testes com `app.Test()`, e migração v2 → v3 (`Mount`/`Static`/`ListenTLS` removidos, `BodyParser`/`ParamsParser`/`QueryParser` removidos, `Ctx` agora interface, struct tag `params` → `uri`, `Redirect()` chainable, redirect default 303). Use sempre que criar/editar arquivos `.go` que importam `github.com/gofiber/fiber/v3`, ao desenhar rotas, controllers, middlewares ou error handlers em Fiber, ao migrar projetos de v2 para v3, ou ao revisar PRs que tocam código Fiber.
---

# Fiber v3 Best Practices (Documentação Oficial)

Skill baseada exclusivamente na documentação oficial em https://docs.gofiber.io/next/ — Guides, API Reference, Middlewares e "What's New in v3". Todas as recomendações aqui devem ser seguidas ao escrever, revisar ou refatorar código que use `github.com/gofiber/fiber/v3`.

Esta skill **complementa** [[go-best-practices]] (Effective Go, error handling, context, concorrência) e [[go-testing-best-practices]] (testify, gomock, testcontainers). Sempre carregue as duas em conjunto ao trabalhar com Fiber.

## Quando Aplicar

- Ao criar, editar ou revisar qualquer arquivo `.go` que importe `github.com/gofiber/fiber/v3`
- Ao desenhar rotas, handlers, middlewares ou error handlers
- Ao implementar binding/validação de request, ou tratamento centralizado de erros
- Ao migrar código de Fiber v2 para v3
- Ao revisar PRs que tocam camada HTTP de um serviço Go

## Pré-requisitos

- **Go 1.25+** é obrigatório (Fiber v3 elevou o mínimo)
- Módulo: `github.com/gofiber/fiber/v3`
- Utils foram extraídos: importe `github.com/gofiber/utils/v2` se precisar

## 1. Inicialização da App e `Config`

Referência: [API — App](https://docs.gofiber.io/next/api/app), [API — Fiber Config](https://docs.gofiber.io/next/api/fiber)

### Padrão idiomático

```go
package main

import (
    "log/slog"

    "github.com/gofiber/fiber/v3"
    "github.com/gofiber/fiber/v3/middleware/logger"
    "github.com/gofiber/fiber/v3/middleware/recover"
    "github.com/gofiber/fiber/v3/middleware/requestid"
)

func newApp(v StructValidator, errHandler fiber.ErrorHandler) *fiber.App {
    app := fiber.New(fiber.Config{
        AppName:         "my-service",
        StructValidator: v,
        ErrorHandler:    errHandler,
        ReadTimeout:     10 * time.Second,
        WriteTimeout:    10 * time.Second,
        IdleTimeout:     60 * time.Second,
        BodyLimit:       4 * 1024 * 1024, // 4 MiB
        TrustProxy:      true,
        TrustProxyConfig: fiber.TrustProxyConfig{
            Proxies: []string{"10.0.0.0/8"},
        },
    })

    // Ordem importa: Recover deve ser o primeiro middleware
    app.Use(recover.New())
    app.Use(requestid.New())
    app.Use(logger.New())

    return app
}
```

### Regras absolutas

- **Sempre configure `ErrorHandler`** — o default só devolve `text/plain`. Em APIs JSON, customize (ver §6).
- **Sempre configure `StructValidator`** se for usar `Bind()` (ver §5).
- **Configure timeouts** (`ReadTimeout`, `WriteTimeout`, `IdleTimeout`) — default é zero (sem limite).
- **Configure `BodyLimit`** explicitamente em endpoints que aceitam upload.
- **`TrustProxy: true` + `TrustProxyConfig`** quando atrás de load balancer / CDN.
- **Não use** `app.Static(...)` (removido em v3): use o middleware `static`.
- **Não use** `app.Mount(...)` (removido): use `app.Use("/api", subApp)`.

### Listener (v3 unificou TLS)

Métodos `ListenTLS`, `ListenMutualTLS`, `ListenMutualTLSWithCertificate` foram **removidos**. Use `Listen` com `ListenConfig`:

```go
err := app.Listen(":3000", fiber.ListenConfig{
    DisableStartupMessage: true,  // movido do fiber.Config em v3
    EnablePrefork:         false, // movido do fiber.Config em v3
    EnablePrintRoutes:     false, // movido do fiber.Config em v3
    ListenerNetwork:       "tcp4",
    TLSMinVersion:         tls.VersionTLS13,
    CertFile:              "cert.pem",
    CertKeyFile:           "key.pem",
    AutoCertManager:       autocertMgr, // suporte ACME / Let's Encrypt nativo
})
```

## 2. Roteamento

Referência: [Guide — Routing](https://docs.gofiber.io/next/guide/routing), [API — App](https://docs.gofiber.io/next/api/app)

### Métodos HTTP

```go
app.Get("/users/:id", handler)
app.Post("/users", handler)
app.Put("/users/:id", handler)
app.Patch("/users/:id", handler)
app.Delete("/users/:id", handler)
app.All("/health", handler)                // todos os métodos
app.Add([]string{"GET", "POST"}, "/x", h)  // v3: methods como slice
```

> **v3:** `Add` mudou de variádico para slice — `Add(method ...string, ...)` ⇒ `Add([]string{...}, path, handler)`.

### Parâmetros

| Sintaxe | Significado |
|---------|-------------|
| `:name` | Parâmetro obrigatório |
| `:name?` | Parâmetro opcional |
| `+` | Wildcard greedy (≥ 1 char) |
| `*` | Wildcard greedy (≥ 0 char) |

```go
app.Get("/user/:name/books/:title", handler)
app.Get("/files/*", handler)            // c.Params("*")
app.Get("/flights/:from-:to", handler)  // /flights/LAX-SFO
```

### Route Constraints (validação no roteador)

Constraints rodam **antes** do handler; se falharem, retornam **404**. Use para garantir que tipos básicos cheguem corretos no handler.

```go
app.Get("/users/:id<int>", handler)
app.Get("/posts/:id<guid>", handler)
app.Get("/items/:age<min(18);max(120)>", handler)
app.Get(`/feed/:date<regex(\d{4}-\d{2}-\d{2})>`, handler)
```

Built-ins: `int`, `bool`, `float`, `guid`, `minLen(n)`, `maxLen(n)`, `len(n)`, `min(n)`, `max(n)`, `range(min,max)`, `alpha`, `datetime(fmt)`, `regex(pat)`.

**Custom constraint** (v3, novo):

```go
type ULIDConstraint struct{}

func (*ULIDConstraint) Name() string { return "ulid" }
func (*ULIDConstraint) Execute(param string, _ ...string) bool {
    _, err := ulid.Parse(param)
    return err == nil
}

app.RegisterCustomConstraint(&ULIDConstraint{})
app.Get("/orders/:id<ulid>", handler)
```

### Ordem de registro importa

> *Routes are matched in registration order (first match wins).*

Declare **rotas específicas antes** de rotas com parâmetros/wildcards:

```go
app.Get("/users/me", currentUser)    // específica primeiro
app.Get("/users/:id", userByID)      // genérica depois
```

### Grupos

```go
api := app.Group("/api")
v1  := api.Group("/v1", authMiddleware) // middleware no grupo
v1.Get("/users", listUsers)              // → /api/v1/users
v1.Post("/users", createUser)
```

### RouteChain (Express-style, novo em v3)

Para empilhar múltiplos métodos no mesmo path sem repetir o path:

```go
app.RouteChain("/events").
    All(audit).
    Get(list).
    Post(create)
```

### Domain Routing (novo em v3)

Custo zero em rotas que não usam domain:

```go
api  := app.Domain("api.example.com")
admin := app.Domain("admin.example.com")

api.Get("/users", listUsers)
admin.Get("/dashboard", dashboard)
```

### Named Routes + URL building

```go
app.Get("/users/:id", show).Name("user.show")

url, err := app.GetRoute("user.show").URL(fiber.Map{"id": 42})
// url = "/users/42"
```

### HEAD automático

V3 registra `HEAD` automaticamente para toda rota `GET` (resposta sem body). Desabilite globalmente com `Config.DisableHeadAutoRegister: true` quando for ruído.

## 3. Handlers, `fiber.Ctx` e Zero-Allocation

Referência: [API — Ctx](https://docs.gofiber.io/next/api/ctx), [What's New](https://docs.gofiber.io/next/whats_new)

### `Ctx` agora é interface

> **v3 breaking change:** handlers usam `func(c fiber.Ctx) error`, **sem ponteiro** (`*fiber.Ctx` é v2).

```go
func listUsers(c fiber.Ctx) error {
    return c.JSON(fiber.Map{"ok": true})
}
```

### Zero-allocation: **NÃO retenha** valores do `Ctx`

> *Values from `fiber.Ctx` are not immutable by default and will be reused across requests.*

Tudo que `Ctx` retorna (string, []byte, header, body) pode ser sobrescrito pelo próximo request. **Nunca** armazene esses valores em goroutines, channels, struct fields, caches ou closures que sobrevivam ao handler.

**Errado** — corrompe o cache no próximo request:
```go
func handler(c fiber.Ctx) error {
    name := c.Query("name")
    go func() {
        cache.Set("last", name) // ❌ name pode mudar quando outra request chegar
    }()
    return nil
}
```

**Certo** — copie antes de passar adiante:
```go
import "github.com/gofiber/utils/v2"

func handler(c fiber.Ctx) error {
    name := utils.CopyString(c.Query("name"))
    go func() {
        cache.Set("last", name) // ✅ cópia segura
    }()
    return nil
}
```

Para bytes use `utils.CopyBytes` ou `append([]byte(nil), b...)`.

### Helpers genéricos (novo em v3)

```go
import "github.com/gofiber/fiber/v3"

id     := fiber.Params[int](c, "id", 0)
page   := fiber.Query[int](c, "page", 1)
ua     := fiber.GetReqHeader[string](c, "User-Agent", "")
userID := fiber.Locals[string](c, "userID")
```

São type-safe, com default value e sem cast manual. Prefira-os a `c.Params("id")` + `strconv.Atoi` em rotas simples.

> **Removidos em v3:** `c.ParamsInt`, `c.QueryInt`, `c.QueryFloat`, `c.QueryBool` — substituídos pelos helpers genéricos acima.

### `Immutable` quando precisar reter

Se realmente precisa de imutabilidade, custe o que custar:

```go
app := fiber.New(fiber.Config{Immutable: true})
```

Use por exceção, não por regra — anula o benefício de performance do Fiber.

### `Ctx` implementa `context.Context`

> **Novo em v3:** `fiber.Ctx` já é `context.Context`. Passe-o direto para repositórios/serviços.

```go
func handler(c fiber.Ctx) error {
    user, err := repo.Find(c, id) // c serve como context.Context
    // ...
}
```

> **Removido em v3:** `c.UserContext()`/`c.SetUserContext()`. Não existem mais.
> **Renomeado:** `c.Context()` (fasthttp) virou `c.RequestCtx()`.

## 4. Binding (Bind API unificada)

Referência: [API — Bind](https://docs.gofiber.io/next/api/bind), [What's New](https://docs.gofiber.io/next/whats_new)

### Métodos

| Método | Origem |
|--------|--------|
| `c.Bind().Body(&out)` | Body (auto-detecta JSON/XML/Form/MsgPack/CBOR pelo `Content-Type`) |
| `c.Bind().JSON(&out)` / `.XML(&out)` / `.Form(&out)` / `.MsgPack(&out)` / `.CBOR(&out)` | Body forçando formato |
| `c.Bind().URI(&out)` | Path params |
| `c.Bind().Query(&out)` | Query string |
| `c.Bind().Header(&out)` | Headers |
| `c.Bind().Cookie(&out)` | Cookies |
| `c.Bind().All(&out)` | Encadeia URI → Body → Query → Headers → Cookies |

> **Removidos em v3:** `BodyParser`, `ParamsParser`, `QueryParser`, `ReqHeaderParser`, `CookieParser` — todos virados `Bind().X(...)`.

### Tags

| Origem | Tag | Exemplo |
|--------|-----|---------|
| Path param | `uri` | `uri:"id"` |
| Query | `query` | `query:"page"` |
| Form | `form` | `form:"email"` |
| Header | `header` | `header:"X-Trace"` |
| Cookie | `cookie` | `cookie:"session"` |
| JSON | `json` | `json:"name"` |
| XML | `xml` | `xml:"name"` |
| MsgPack | `msgpack` | `msgpack:"name"` |

> **v3 breaking change:** struct tag para path params mudou de `params:"id"` (v2) para `uri:"id"` (v3). Faça o rename no momento da migração.

### Exemplo canônico

```go
type CreateUserRequest struct {
    OrgID    int    `uri:"orgId" validate:"required,gt=0"`
    Name     string `json:"name" validate:"required,min=3"`
    Email    string `json:"email" validate:"required,email"`
    TraceID  string `header:"X-Trace-Id"`
    Page     int    `query:"page" default:"1" validate:"gte=1"`
}

func createUser(c fiber.Ctx) error {
    req := new(CreateUserRequest)
    if err := c.Bind().All(req); err != nil {
        return err // delega ao ErrorHandler central
    }
    // req validado se StructValidator estiver configurado
    return c.Status(fiber.StatusCreated).JSON(req)
}
```

### Defaults

Defaults com tag `default:`. Para slices, separe com `|`:

```go
type Filter struct {
    Status []string `query:"status" default:"active|pending"`
    Limit  int      `query:"limit"  default:"20"`
}
```

### Auto-handling

Por padrão o binder retorna **400 Bad Request** automaticamente quando a parse falha. Desligue se quiser tratar manualmente:

```go
c.Bind().WithoutAutoHandling().Body(&req)
```

### Custom binders (novos MIME types)

```go
app.RegisterCustomBinder(&YAMLBinder{})
```

### Distinguir origem do erro

```go
var be *fiber.BindError
if errors.As(err, &be) {
    switch be.Source {
    case fiber.BindSourceURI:
        return fiber.ErrNotFound          // recurso inexistente
    default:
        return fiber.ErrBadRequest        // input inválido
    }
}
```

## 5. Validação

Referência: [Guide — Validation](https://docs.gofiber.io/next/guide/validation)

Fiber não traz validador embutido — você fornece um `StructValidator` no `Config`. Padrão: `go-playground/validator`.

```go
import "github.com/go-playground/validator/v10"

type structValidator struct {
    v *validator.Validate
}

func (s *structValidator) Validate(out any) error {
    return s.v.Struct(out)
}

app := fiber.New(fiber.Config{
    StructValidator: &structValidator{v: validator.New(validator.WithRequiredStructEnabled())},
})
```

### Regras

- Validação **só roda em structs**. *"Binding into maps and other non-struct types skips validation."*
- O validador é chamado **dentro** de `Bind().X(...)` — não precisa chamar `Validate` manualmente.
- Em erro, faça type-assert para `validator.ValidationErrors` no `ErrorHandler` central para gerar resposta estruturada (ver §6).

### Tags comuns

```go
type Req struct {
    Email    string  `json:"email"    validate:"required,email"`
    Age      int     `json:"age"      validate:"gte=18,lte=120"`
    Password string  `json:"password" validate:"required,min=8"`
    URL      string  `json:"url"      validate:"omitempty,url"`
    Role     string  `json:"role"     validate:"oneof=admin user guest"`
}
```

### Custom validators

Encapsule o `*validator.Validate` e registre funções:

```go
v := validator.New()
v.RegisterValidation("strong_password", strongPassword)
```

## 6. Error Handling Centralizado

Referência: [Guide — Error Handling](https://docs.gofiber.io/next/guide/error-handling)

### Filosofia: **retorne erro, não responda**

Handlers e middlewares devem **retornar** o erro. O `ErrorHandler` central decide formato, status e logging.

```go
func handler(c fiber.Ctx) error {
    user, err := svc.Get(c, id)
    if err != nil {
        return err // não chame c.Status(500).Send(...) aqui
    }
    return c.JSON(user)
}
```

### `fiber.Error` e `fiber.NewError`

```go
return fiber.ErrNotFound                                     // 404 "Not Found"
return fiber.NewError(fiber.StatusConflict, "email taken")   // 409 com msg
return fiber.NewError(fiber.StatusUnprocessableEntity)       // 422 + msg padrão
```

Constantes prontas: `fiber.ErrBadRequest`, `ErrUnauthorized`, `ErrForbidden`, `ErrNotFound`, `ErrConflict`, `ErrUnprocessableEntity`, `ErrTooManyRequests`, `ErrInternalServerError`, `ErrServiceUnavailable`, etc.

### `ErrorHandler` customizado (JSON API)

```go
type ErrorResponse struct {
    Code    int               `json:"code"`
    Message string            `json:"message"`
    Errors  map[string]string `json:"errors,omitempty"`
    TraceID string            `json:"traceId,omitempty"`
}

func errorHandler(c fiber.Ctx, err error) error {
    code := fiber.StatusInternalServerError
    msg  := "internal server error"

    var fe *fiber.Error
    var ve validator.ValidationErrors
    var be *fiber.BindError

    switch {
    case errors.As(err, &fe):
        code, msg = fe.Code, fe.Message

    case errors.As(err, &ve):
        details := map[string]string{}
        for _, fe := range ve {
            details[fe.Field()] = fe.Tag()
        }
        return c.Status(fiber.StatusUnprocessableEntity).JSON(ErrorResponse{
            Code: fiber.StatusUnprocessableEntity,
            Message: "validation failed",
            Errors: details,
            TraceID: requestid.FromContext(c),
        })

    case errors.As(err, &be):
        code = fiber.StatusBadRequest
        msg = be.Error()

    case errors.Is(err, domain.ErrNotFound):
        code, msg = fiber.StatusNotFound, "resource not found"

    default:
        slog.ErrorContext(c, "unhandled error", "err", err) // só log no default
    }

    return c.Status(code).JSON(ErrorResponse{
        Code: code,
        Message: msg,
        TraceID: requestid.FromContext(c),
    })
}
```

### Recovery de panic é separado

> Fiber **não** captura `panic` automaticamente. Você precisa do middleware `recover` (ver §7).

## 7. Middlewares (oficiais)

Referência: [Middleware index](https://docs.gofiber.io/next/middleware/)

### Ordem recomendada

```go
app.Use(recover.New())         // 1. captura panic primeiro
app.Use(requestid.New())       // 2. injeta trace ID
app.Use(logger.New())          // 3. loga já com trace ID
app.Use(cors.New())            // 4. CORS antes de auth
app.Use(authMiddleware)        // 5. auth depois de CORS (preflight ok)
app.Use(timeout.New(timeout.Config{Timeout: 5 * time.Second}))
```

### `recover`

```go
import recoverer "github.com/gofiber/fiber/v3/middleware/recover"

app.Use(recoverer.New(recoverer.Config{
    EnableStackTrace: true,
    PanicHandler: func(c fiber.Ctx, r any) error {
        slog.ErrorContext(c, "panic", "value", r)
        return fiber.ErrInternalServerError // não vaza detalhe
    },
}))
```

### `logger`

Formato com tags `${...}`:

```go
app.Use(logger.New(logger.Config{
    Format:     logger.JSONFormat,                       // novo em v3
    TimeFormat: time.RFC3339,
    TimeZone:   "UTC",
    Stream:     os.Stdout,                               // v3: era "Output"
    CustomTags: map[string]logger.LogFunc{
        "tenant": func(o logger.Output, c fiber.Ctx, d *logger.Data, _ string) (int, error) {
            return o.WriteString(fiber.Locals[string](c, "tenant"))
        },
    },
}))
```

Tags úteis: `${time}`, `${ip}`, `${method}`, `${path}`, `${status}`, `${latency}`, `${error}`, `${requestid}` (auto-registrada por requestid), `${reqHeader:X-Foo}`.

> **v3 breaking change:** campo `Output` virou `Stream`. Para integrar com Zerolog/Zap/Logrus use `logger.LoggerToWriter(...)`.

### `requestid`

```go
import "github.com/gofiber/fiber/v3/middleware/requestid"

app.Use(requestid.New())

func handler(c fiber.Ctx) error {
    rid := requestid.FromContext(c) // type-safe, novo em v3
    // ...
}
```

> **v3 breaking change:** dados de middleware não vão mais em `c.Locals("requestid")`. Use as funções `*.FromContext(c)` exportadas pelos pacotes (`csrf.TokenFromContext`, `session.FromContext`, `basicauth.UsernameFromContext`, etc.).

### `cors`

```go
app.Use(cors.New(cors.Config{
    AllowOrigins:     []string{"https://app.example.com"}, // v3: slice, não CSV
    AllowMethods:     []string{fiber.MethodGet, fiber.MethodPost},
    AllowHeaders:     []string{"Authorization", "Content-Type"},
    AllowCredentials: true,
    MaxAge:           int((12 * time.Hour).Seconds()),
}))
```

> **v3:** todos os campos CSV viraram `[]string`. Migre `"a,b,c"` para `[]string{"a","b","c"}`.

### `timeout`

```go
app.Use(timeout.New(timeout.Config{
    Timeout: 5 * time.Second,
    OnTimeout: func(c fiber.Ctx) error {
        return fiber.NewError(fiber.StatusGatewayTimeout, "request timed out")
    },
}))
```

V3 trouxe `Abandon`: retorno imediato ao timeout sem esperar o handler terminar.

### Outros essenciais

- **`compress`** — agora suporta Zstd, respeita `BodyLimit` na descompressão
- **`cache`** — chave estrutural (method+path+query+headers), `Cache-Control` por default, suporta `CacheInvalidator`
- **`session`** — `IdleTimeout` + `AbsoluteTimeout`, **chamar `s.Release()`** manualmente, `SecureToken` por default
- **`csrf`** — usa `IdleTimeout`, valida `Sec-Fetch-Site`, token redacted nos logs
- **`basicauth`** — exige senhas hasheadas, `Authorizer` recebe contexto
- **`encryptcookie`** — cookie name vira AAD na criptografia
- **`sse`** — novo, Server-Sent Events
- **`proxy`** — `TlsConfig` virou `TLSConfig`, balancer variádico

### Pular middleware condicionalmente

Toda config oficial expõe `Next func(c fiber.Ctx) bool`:

```go
app.Use(logger.New(logger.Config{
    Next: func(c fiber.Ctx) bool {
        return c.Path() == "/health" // não loga health checks
    },
}))
```

### Adaptadores

Fiber v3 aceita 17 assinaturas de handler, incluindo `http.Handler`, `http.HandlerFunc` e `fasthttp.RequestHandler` diretamente. Útil para reaproveitar middleware de `net/http` sem wrapper.

## 8. Hooks de Ciclo de Vida

Referência: [API — Hooks](https://docs.gofiber.io/next/api/hooks)

```go
app.Hooks().OnListen(func(d fiber.ListenData) error {
    slog.Info("listening", "host", d.Host, "port", d.Port, "tls", d.TLS)
    return nil
})

app.Hooks().OnPreShutdown(func() error {
    return queue.Drain(ctx) // drena fila antes de fechar
})

app.Hooks().OnPostShutdown(func(err error) error {
    if err != nil {
        slog.Error("shutdown failed", "err", err)
    }
    return nil
})
```

> **v3:** `OnShutdown` deprecado → use `OnPreShutdown` (antes) e `OnPostShutdown` (depois). Hooks são protegidos por mutex e seguros para registro concorrente.

Outros úteis: `OnRoute`, `OnName`, `OnGroup`, `OnGroupName`, `OnFork`, `OnMount`, `OnPreStartupMessage`, `OnPostStartupMessage`.

## 9. Graceful Shutdown

```go
go func() {
    if err := app.Listen(":3000"); err != nil {
        slog.Error("listen", "err", err)
    }
}()

stop := make(chan os.Signal, 1)
signal.Notify(stop, syscall.SIGINT, syscall.SIGTERM)
<-stop

ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

if err := app.ShutdownWithContext(ctx); err != nil {
    slog.Error("shutdown", "err", err)
}
```

- `app.Shutdown()` fecha sem timeout — prefira `ShutdownWithContext` em produção.
- Combine com `OnPreShutdown` para drenar dependências.

## 10. State e SharedState (novo em v3)

```go
app.State().Set("buildSHA", buildSHA)
sha, _ := app.State().Get("buildSHA")
```

Para prefork/multi-processo use **`SharedState`** (storage-backed): valores compartilhados entre forks.

> **Evite** State como "globals disfarçados". Use para metadados imutáveis (build info, feature flags raras). Dependências (DB, queue, cache) injete por construtor.

## 11. `fiber.Service` (novo em v3)

Interface para registrar dependências cujo ciclo de vida o Fiber gerencia (start/stop):

```go
type DB struct{ conn *sql.DB }

func (db *DB) String() string                 { return "postgres" }
func (db *DB) State(_ context.Context) (string, error) { return "ready", nil }
func (db *DB) Start(ctx context.Context) error { /* ping */ return nil }
func (db *DB) Stop(ctx context.Context)  error { return db.conn.Close() }
func (db *DB) Terminate(ctx context.Context) error { return db.Stop(ctx) }

app := fiber.New(fiber.Config{Services: []fiber.Service{db}})
```

## 12. Templates

```go
import "github.com/gofiber/template/html/v2"

engine := html.New("./views", ".html")
app := fiber.New(fiber.Config{
    Views:             engine,
    ViewsLayout:       "layouts/main",
    PassLocalsToViews: true,
})

app.Get("/", func(c fiber.Ctx) error {
    return c.Render("index", fiber.Map{"Title": "Hello"})
})
```

- **`c.Bind()`** = bind de **request**. Para bind de **template**, use **`c.ViewBind(fiber.Map{...})`** (renomeado em v3).
- **`app.ReloadViews()`** para recarregar templates em dev (com guards de segurança).

## 13. Testes

Referência: [API — App `Test`](https://docs.gofiber.io/next/api/app)

`app.Test(req, timeout...)` executa o handler in-memory, sem subir socket:

```go
func TestListUsers(t *testing.T) {
    app := newApp(...)
    app.Get("/users", listUsers)

    req := httptest.NewRequest(http.MethodGet, "/users?page=2", nil)
    resp, err := app.Test(req, -1) // -1 desativa timeout
    require.NoError(t, err)
    require.Equal(t, http.StatusOK, resp.StatusCode)

    var body []User
    require.NoError(t, json.NewDecoder(resp.Body).Decode(&body))
}
```

- Timeout default é 1s — passe `-1` para desativar quando estiver depurando com breakpoints.
- Para testes de aceitação completos (multi-serviço), combine com [[go-testing-best-practices]] e testcontainers.

## 14. Migração v2 → v3 (resumo executável)

| v2 | v3 |
|----|----|
| `func(c *fiber.Ctx) error` | `func(c fiber.Ctx) error` |
| `c.BodyParser(&x)` | `c.Bind().Body(&x)` |
| `c.QueryParser(&x)` | `c.Bind().Query(&x)` |
| `c.ParamsParser(&x)` | `c.Bind().URI(&x)` |
| `c.CookieParser(&x)` | `c.Bind().Cookie(&x)` |
| `c.QueryInt("page")` | `fiber.Query[int](c, "page", 0)` |
| `c.ParamsInt("id")` | `fiber.Params[int](c, "id", 0)` |
| `params:"id"` | `uri:"id"` |
| `c.UserContext()` | `c` (já é `context.Context`) |
| `c.Context()` (fasthttp) | `c.RequestCtx()` |
| `c.Redirect(url, 302)` | `c.Redirect().Status(302).To(url)` |
| Redirect default `302` | Redirect default `303` |
| `c.Locals("requestid")` | `requestid.FromContext(c)` |
| `app.Static(...)` | middleware `static` |
| `app.Mount("/x", sub)` | `app.Use("/x", sub)` |
| `app.ListenTLS(...)` | `app.Listen(addr, fiber.ListenConfig{...})` |
| `Config.DisableStartupMessage` | `ListenConfig.DisableStartupMessage` |
| `Config.Prefork` | `ListenConfig.EnablePrefork` |
| `EnabledTrustedProxyCheck` | `TrustProxy` + `TrustProxyConfig` |
| `BindBody / BindQuery` | `c.Bind().Body / .Query` |
| Logger `Output:` | Logger `Stream:` |
| `Bind()` (template) | `ViewBind()` |
| `utils.CopyString` (interno) | `github.com/gofiber/utils/v2` |

CLI oficial automatiza grande parte:
```bash
go install github.com/gofiber/cli/fiber@latest
fiber migrate --to v3
```

## Checklist de Revisão (Fiber v3)

Ao revisar ou aprovar código Fiber, verifique:

- [ ] Go `1.25+` e import `github.com/gofiber/fiber/v3`
- [ ] Handlers usam `func(c fiber.Ctx) error` (sem `*`)
- [ ] **Nenhum** valor retornado por `c.Query/Params/Body/Get` é retido fora do handler sem `utils.CopyString`/`CopyBytes`
- [ ] `fiber.Config` define `ErrorHandler`, `StructValidator`, timeouts e `BodyLimit`
- [ ] Listener configurado via `app.Listen(addr, fiber.ListenConfig{...})` (não `ListenTLS`)
- [ ] Rotas específicas antes das genéricas; constraints (`<int>`, `<guid>`, ...) onde fizer sentido
- [ ] Struct tag para path param é `uri:` (não `params:`)
- [ ] Binding usa `c.Bind().X(...)`; sem `BodyParser/QueryParser/ParamsParser`
- [ ] Validações em structs com tags `validate:"..."`; erros tratados no `ErrorHandler`
- [ ] Erros são **retornados**, nunca respondidos manualmente com `c.Status(500).Send(...)`
- [ ] `recover.New()` registrado **antes** de qualquer outro middleware
- [ ] Middleware reads usam `*.FromContext(c)` (não `c.Locals(string)`)
- [ ] CORS / Logger / outros configs em `[]string`, não CSV
- [ ] Logger `Stream:` (não `Output:`); `Format` apropriado (JSON em prod)
- [ ] Graceful shutdown com `app.ShutdownWithContext(ctx)` + `OnPreShutdown`/`OnPostShutdown`
- [ ] Hooks de start usam `OnListen`; sem `OnShutdown` (deprecated)
- [ ] Dependências de I/O (DB, queue, cache) injetadas — **não** vivem em `State`
- [ ] Testes usam `app.Test(req, -1)`; sem subir socket real
- [ ] Sem `c.UserContext()` ou `c.Context()` (fasthttp) — usa `c` direto ou `c.RequestCtx()`

## Fontes Oficiais

Toda recomendação acima vem destas páginas — consulte-as ao bater dúvida:

- https://docs.gofiber.io/next/ (índice geral)
- https://docs.gofiber.io/next/whats_new (breaking changes v2 → v3)
- https://docs.gofiber.io/next/api/fiber (Config)
- https://docs.gofiber.io/next/api/app (App, Listen, Routes, Group)
- https://docs.gofiber.io/next/api/ctx (Ctx interface, métodos)
- https://docs.gofiber.io/next/api/bind (Bind API unificada)
- https://docs.gofiber.io/next/api/hooks (lifecycle hooks)
- https://docs.gofiber.io/next/guide/routing (parâmetros, constraints, grupos, RouteChain)
- https://docs.gofiber.io/next/guide/error-handling (ErrorHandler, fiber.Error)
- https://docs.gofiber.io/next/guide/validation (StructValidator)
- https://docs.gofiber.io/next/guide/templates (Views, Render, ViewBind)
- https://docs.gofiber.io/next/middleware/ (catálogo de middlewares oficiais)
