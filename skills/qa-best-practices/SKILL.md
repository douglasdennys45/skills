---
name: qa-best-practices
description: Aplica melhores práticas de QA (Quality Assurance) para validação completa de features backend em Go. Cobre análise estática (gofmt, go vet, golangci-lint, go build), testes unitários e de integração com race detector e cobertura, mutation testing com go-mutesting, testes E2E de API com curl/HTTP, verificação de contrato Swagger/OpenAPI, auditoria de qualidade de código (error handling, concorrência, interfaces, arquitetura), verificação de infraestrutura (MongoDB, RabbitMQ, health checks) e geração de relatórios de QA com evidências. Use sempre que validar uma feature concluída, gerar relatório de QA, executar pipeline de qualidade, revisar implementação contra PRD/TechSpec, ou aplicar tolerância zero a bugs antes de release.
---

# QA Best Practices — Validação Completa de Features Backend Go

Skill que aplica o processo sistemático de QA backend para garantir qualidade end-to-end. Combina análise estática, testes automatizados, mutation testing, testes E2E, auditoria de código e verificação de infraestrutura em um pipeline com **tolerância zero a bugs**.

Combina com [[go-best-practices]] (padrões de código) e [[go-testing-best-practices]] (estratégias de teste).

## Princípio Fundamental

> **TOLERÂNCIA ZERO A BUGS.** Uma feature só recebe status **APROVADO** quando ZERO bugs são encontrados — independente da severidade (Crítica, Alta, Média ou Baixa). Qualquer bug encontrado, por menor que seja, resulta automaticamente em **REPROVADO**.

Não existe:
- Aprovação condicional
- Aprovação com ressalvas
- Aprovação com bugs pendentes de correção
- Distinção entre bugs "bloqueantes" e "não bloqueantes"

Severidade é usada apenas para **priorização de correção**, nunca para decisão de aprovação.

## Quando Aplicar

- Ao validar uma feature backend Go concluída antes de release
- Ao revisar implementação contra PRD, TechSpec ou Tasks
- Ao executar pipeline completo de qualidade (estática + testes + E2E)
- Ao gerar relatório de QA com evidências auditáveis
- Ao validar contrato de API contra documentação Swagger/OpenAPI
- Ao verificar saúde de componentes de infraestrutura (DB, broker, health endpoints)
- Antes de bump de versão (SemVer) e atualização de changelog

## Estrutura de Evidências

**REGRA CRÍTICA:** Todos os artefatos de QA DEVEM ser salvos em uma estrutura padronizada:

```
.cognup/specs/[feature]/report/
├── qa-report.md                          # Relatório final consolidado
├── bugs.md                               # Documentação detalhada de bugs
└── evidence/
    ├── static-analysis.md                # Etapa 3
    ├── test-results.md                   # Etapa 4
    ├── coverage-summary.md               # Etapa 4
    ├── mutation-testing.md               # Etapa 5
    ├── api-e2e-results.md                # Etapa 6
    ├── contract-verification.md          # Etapa 7
    ├── code-quality-audit.md             # Etapa 8
    └── infrastructure.md                 # Etapa 9
```

**OBRIGATORIEDADE DE EVIDÊNCIAS:**
- Todo teste — PASSOU ou FALHOU — DEVE ter evidência documentada
- Nenhum requisito pode ser marcado como verificado sem evidência correspondente
- Todo bug DEVE ter request/response, output de comando ou trecho de código como evidência
- Teste sem evidência = teste NÃO EXECUTADO

## Pipeline de QA — 13 Etapas

### Etapa 1: Análise da Documentação (Obrigatória)

Antes de qualquer teste:

1. **Ler o PRD** e extrair TODOS os requisitos funcionais numerados em checklist
2. **Ler a TechSpec** e anotar decisões técnicas verificáveis (endpoints, modelos, error codes, regras de negócio)
3. **Ler o arquivo de Tasks** e confirmar status de conclusão
4. **Ler arquitetura** (`.claude/rules/architecture.md`) para entender convenções
5. **Consultar skills** [[go-best-practices]] e [[go-testing-best-practices]] para critérios de qualidade
6. **Construir matriz de verificação** mapeando cada requisito → cenário de teste

Para cada requisito, identifique:
- Endpoint(s) envolvidos (método, path, schemas)
- Status codes HTTP esperados (sucesso e erro)
- Operações de banco que devem ser verificadas
- Operações de mensageria/eventos
- Regras de negócio e validações
- Padrões de error handling esperados

**Nunca pule esta etapa** — entender requisitos é a base da QA.

### Etapa 2: Preparação do Ambiente (Obrigatória)

```bash
# Criar diretórios de evidências
mkdir -p .cognup/specs/[feature]/report/evidence/

# Verificar toolchain
go version
docker info

# Verificar compilação
go build ./...

# Verificar dependências
go mod tidy && go mod verify

# Verificar variáveis de ambiente
cat .env  # ou env | grep APP_
```

**Protocolo de Startup da Aplicação:**

1. Verificar se já está rodando: `curl -s http://localhost:8080/health`
2. Se não, iniciar em background: `go run cmd/api/main.go &`
3. Aguardar prontidão (poll no `/health` com 2-3s incrementais)
4. Verificar health endpoint retorna sucesso
5. Registrar PID para cleanup posterior

Se a aplicação falhar ao iniciar: **REPROVADO imediato**.

### Etapa 3: Análise Estática de Código (Obrigatória)

Execute todas as ferramentas e registre resultados em `evidence/static-analysis.md`:

| Ferramenta | Comando | O que Verifica |
|------------|---------|----------------|
| **Build** | `go build ./...` | Erros de compilação, type safety |
| **Vet** | `go vet ./...` | Construções suspeitas, erros comuns |
| **Lint** | `golangci-lint run` | Qualidade, estilo, bugs potenciais |
| **Format** | `gofmt -l .` | Conformidade de formatação |
| **Imports** | `goimports -l .` | Imports organizados e necessários |
| **Mod Tidy** | `go mod tidy -diff` | go.mod limpo |
| **Mod Verify** | `go mod verify` | Integridade de dependências |

Para cada ferramenta:
1. Executar o comando
2. Capturar a saída **completa**
3. Marcar como **PASSOU** (zero issues) ou **FALHOU**
4. Se FALHOU: classificar cada problema e documentar

**Verificações adicionais:**
- Sem `TODO`/`FIXME` não resolvidos relacionados à feature
- Identificadores exportados possuem doc comments
- Error handling usa `fmt.Errorf("context: %w", err)`
- Sem uso de `panic` em código de produção (apenas em `init` ou casos excepcionais)

### Etapa 4: Testes Unitários e Integração (Obrigatória)

```bash
# Suite completa com race detector e cobertura
go test -v -race -count=1 -coverprofile=coverage.out ./...

# Cobertura por função
go tool cover -func=coverage.out

# Cobertura total
go tool cover -func=coverage.out | tail -1

# HTML para análise visual
go tool cover -html=coverage.out -o coverage.html

# Testes de integração (build tag separado)
go test -v -race -tags=integration ./...
```

**Análise obrigatória:**
1. Parse a saída — identificar passados, falhos, pulados
2. **ZERO falhas** é obrigatório
3. Cobertura por pacote registrada
4. **Cobertura mínima: 80%** (threshold do projeto)
5. Race conditions: zero detecções

Para cada teste que falhou:
- Nome do teste, pacote, mensagem de erro
- Classificar como bug e documentar

Para análise de cobertura:
- Percentuais por pacote
- Pacotes com cobertura baixa/zero sinalizados
- Código novo sem cobertura = bug

Referência: [[go-testing-best-practices]] para padrões table-driven, testify, gomock, build tags.

Salve em `evidence/test-results.md` e `evidence/coverage-summary.md`.

### Etapa 5: Mutation Testing — go-mutesting (Obrigatória)

Mutation testing modifica código-fonte em pequenas formas (mutações). Testes que detectam = "matam" o mutante. **Mutation score alto = suite robusta**.

**Instalação:**

```bash
go install github.com/zimmski/go-mutesting/...@latest
```

**Execução:**

```bash
# Projeto completo
go-mutesting ./...

# Direcionado para pacotes da feature
go-mutesting ./internal/application/... ./internal/domain/...

# Com blacklist (falsos positivos conhecidos)
go-mutesting --blacklist mutation.blacklist ./...
```

**Análise:**
1. Cada mutação: **PASS** (morto = bom) ou **FAIL** (sobreviveu = lacuna nos testes)
2. Calcular score: `passed / total` (1.0 = todos mortos)
3. **Threshold mínimo: 80%**
4. Mutantes sobreviventes documentados com checksum, arquivo, patch
5. Classificar cada sobrevivente:
   - **Lacuna real**: precisa de novo teste → bug severidade Média
   - **Falso positivo**: early exit, otimização → candidato a blacklist

**Interpretação:**
- `mutation score >= 0.80`: aceitável
- `mutation score < 0.80`: REPROVADO — adicionar testes

Salve em `evidence/mutation-testing.md`.

### Etapa 6: Testes E2E de API (Obrigatória)

Para cada endpoint do Swagger e/ou PRD, teste exaustivamente com `curl`:

**Padrão de comando:**

```bash
curl -s -w "\n%{http_code}" -X [METHOD] http://localhost:8080/[path] \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${TOKEN}" \
  -d '[request body]'
```

**Categorias obrigatórias:**

#### 6.1 Happy Path
- Request válido com headers e body corretos
- Status code corresponde ao esperado (200, 201, 204)
- Body bate com schema Swagger
- Headers corretos (`Content-Type: application/json`)

#### 6.2 Casos de Erro
- **400 Bad Request**: JSON malformado, campos obrigatórios ausentes, tipos inválidos
- **401 Unauthorized**: sem token
- **403 Forbidden**: permissões insuficientes
- **404 Not Found**: recurso inexistente
- **409 Conflict**: criação duplicada
- **422 Unprocessable Entity**: dados semanticamente inválidos
- **500**: API nunca retorna stack traces ou detalhes internos

#### 6.3 Edge Cases
- Body vazio
- Valores extremamente longos
- Caracteres especiais (unicode, SQL injection, XSS)
- Limites numéricos (0, negativos, MAX_INT)
- Requisições concorrentes ao mesmo endpoint
- Path traversal (`../../`)
- Headers inválidos ou ausentes

Para cada teste:
1. Executar request
2. Capturar resposta **completa** (status, headers, body)
3. Verificar contra comportamento esperado
4. **PASSOU** ou **FALHOU**
5. Se FALHOU: documentar request/response completos

Salve em `evidence/api-e2e-results.md`.

### Etapa 7: Verificação de Contrato API (Obrigatória)

Compare `docs/swagger.yaml` contra comportamento real:

1. Ler `docs/swagger.yaml` e extrair todos os endpoints
2. Para cada endpoint documentado verificar:
   - Existe e responde (não 404)
   - Método HTTP correto
   - Schema de request bate (campos obrigatórios, tipos)
   - Schema de response bate (nomes, tipos, estrutura)
   - Status codes documentados são realmente retornados
   - `Content-Type` corresponde
3. Detectar endpoints **não documentados** (existem mas não no Swagger)
4. Detectar endpoints **fantasma** (no Swagger mas não implementados)

**Regenerar e comparar:**

```bash
swag init -g cmd/api/main.go --parseDependency --parseInternal
git diff docs/swagger.yaml docs/swagger.json
```

Marcar cada endpoint: **MATCH** ou **MISMATCH**.

Salve em `evidence/contract-verification.md`.

### Etapa 8: Auditoria de Qualidade de Código Go (Obrigatória)

Consulte [[go-best-practices]] para critérios completos. Verificar e documentar em `evidence/code-quality-audit.md`:

#### 8.1 Error Handling

| Critério | Verificar | Evidência |
|----------|-----------|-----------|
| Erros sempre verificados | Sem `_` ignorando erros (`data, _ := ...`) | Listar ocorrências |
| Error wrapping | Uso de `%w` em `fmt.Errorf("context: %w", err)` | Funções sem wrapping |
| Sentinel/Custom errors | Tipos/variáveis para erros do domínio | Entidades sem erros tipados |
| `errors.Is`/`errors.As` | Uso correto ao invés de `==` | Locais incorretos |
| Sem `panic` em produção | `panic` apenas em `init` ou casos terminais | Listar usos suspeitos |

#### 8.2 Concorrência

| Critério | Verificar | Evidência |
|----------|-----------|-----------|
| Lifecycle de goroutines | Toda goroutine tem cancelamento via `context` | Goroutines sem cancelamento |
| Sincronização | `sync.Mutex` ou channels para estado compartilhado | Estado desprotegido |
| Race conditions | Testes passam com `-race` | Output do race detector |
| `WaitGroup` | `sync.WaitGroup` para sincronização | Goroutines órfãs |
| Sem leak | Goroutines terminam ao final do ciclo | Análise com goleak |

#### 8.3 Interfaces e Design

| Critério | Verificar | Evidência |
|----------|-----------|-----------|
| Interfaces pequenas | Focadas, poucos métodos (ISP) | Interfaces inchadas |
| Accept interfaces, return structs | Funções aceitam interface, retornam concreto | Violações |
| Naming | camelCase privado, PascalCase público, sem stuttering | Violações (`order.OrderRepository`) |
| Function design | Funções pequenas, ≤ 3-4 parâmetros | Funções gigantes |
| Context primeiro | `ctx context.Context` é o primeiro parâmetro | Violações |

#### 8.4 Qualidade de Testes

| Critério | Verificar | Evidência |
|----------|-----------|-----------|
| Table-driven | `t.Run()` com `tests := []struct{...}` | Testes sem padrão |
| Testify | `assert`/`require` ao invés de `if got != want { t.Errorf(...) }` | Testes verbosos |
| Mocks apenas externos | Mock apenas dependências externas (não over-mocking) | Over-mocking |
| Cenários de erro | Não apenas happy path | Funções sem testes de erro |
| `t.Parallel()` | Testes independentes em paralelo | Oportunidades perdidas |
| Fixtures isoladas | `t.TempDir()`, `t.Cleanup()` | State leak entre testes |

#### 8.5 Conformidade Arquitetural

Consultar `.claude/rules/architecture.md` do projeto.

| Critério | Verificar | Evidência |
|----------|-----------|-----------|
| Regra de dependência | Domain não importa Application/Infrastructure | Violações de import |
| Construtores | Assinaturas seguem convenção (interface vs concreto) | Construtores fora do padrão |
| Naming de arquivos | kebab-case, pacotes lowercase singular | Violações |
| FX composition | Providers registrados corretamente | Grafo FX |
| Swagger annotations | Handlers possuem anotações completas | Handlers sem docs |

### Etapa 9: Verificação de Infraestrutura (Obrigatória)

#### 9.1 MongoDB

- Health endpoint reporta `mongodb: true`
- Se feature envolve persistência:
  - Criar recurso via POST → buscar via GET → comparar dados
  - Integridade: campos batem com o enviado
  - Índices criados conforme spec (verificar via `db.collection.getIndexes()`)

#### 9.2 RabbitMQ

- Health endpoint reporta `rabbitmq: true`
- Se feature envolve mensageria:
  - Disparar evento via API
  - Verificar publicação (RabbitMQ Management UI ou logs)
  - Exchanges/queues criados conforme spec

#### 9.3 Health Check

- `GET /health` retorna 200
- Resposta inclui campos: `server`, `mongodb`, `rabbitmq`
- `requestId` e `timestamp` presentes e formatados corretamente

#### 9.4 Graceful Shutdown

- Aplicação lida com `SIGTERM`/`SIGINT` gracefully
- Requisições em andamento completam antes do shutdown
- Conexões fechadas na ordem: HTTP → RabbitMQ → MongoDB

Salve em `evidence/infrastructure.md`.

### Etapa 10: Documentação de Bugs

Para cada bug, em `report/bugs.md`:

```markdown
## BUG-[ID]: [Descrição curta]

- **Severidade**: Crítica / Alta / Média / Baixa
- **Requisito**: [ID do PRD]
- **Categoria**: Análise Estática / Falha de Teste / API E2E / Contrato / Qualidade / Infraestrutura
- **Passos para Reproduzir**:
  1. [passo]
  2. [passo]
- **Resultado Esperado**: [o que deveria acontecer]
- **Resultado Atual**: [o que aconteceu]
- **Evidência**:
  ```
  [request/response, output, mensagem de erro]
  ```
- **Ambiente**: localhost:8080
```

**Critérios de Severidade** (para priorização de correção, NÃO para aprovação):

- **Crítica**: crash, perda de dados, vulnerabilidade de segurança, panic, race condition em produção
- **Alta**: endpoint principal quebrado, persistência incorreta, bypass de auth, suite de testes falhando
- **Média**: status code errado, error handling ausente, validação incompleta, cobertura < threshold
- **Baixa**: problemas menores de qualidade, doc comments ausentes, formatação inconsistente

**REGRA DE APROVAÇÃO:** Qualquer bug — **incluindo Baixa** — torna o QA REPROVADO.

### Etapa 11: Geração do Relatório de QA (Obrigatória)

Crie `report/qa-report.md` com este formato:

```markdown
# Relatório de QA Backend - [Feature]

## Resumo
- **Data**: [AAAA-MM-DD]
- **QA Engineer**: AI QA Validator
- **Status**: APROVADO / REPROVADO
- **Total de Requisitos**: [X]
- **Requisitos Atendidos**: [Y]
- **Requisitos Reprovados**: [Z]
- **Bugs Encontrados**: [W]
- **Total de Evidências**: [N]

## Requisitos Verificados
| ID | Requisito | Status | Evidência |

## Análise Estática
| Ferramenta | Comando | Status | Detalhes |
| Build | `go build ./...` | OK / FALHOU | ... |
| Vet | `go vet ./...` | OK / FALHOU | ... |
| Lint | `golangci-lint run` | OK / FALHOU | ... |
| Format | `gofmt -l .` | OK / FALHOU | ... |

## Testes
| Pacote | Total | Passed | Failed | Skipped |
- **Cobertura Total**: [X]%
- **Threshold Mínimo**: 80%
- **Race Conditions**: [Sim/Não]

## Mutation Testing
| Métrica | Valor |
| Total de Mutantes | [X] |
| Mortos | [Y] |
| Sobreviventes | [Z] |
| Mutation Score | [Y/X] ([%]) |

## E2E
| Endpoint | Método | Cenário | Status Esperado | Recebido | Status |

## Contrato (Swagger)
| Endpoint | Método | Documentado | Implementado | Match | Status |

## Auditoria de Código
### Error Handling, Concorrência, Interfaces, Testes, Arquitetura
(uma tabela por seção)

## Infraestrutura
| Componente | Status | Detalhes |

## Bugs
| ID | Descrição | Severidade | Categoria | Requisito | Evidência |

## Inventário de Evidências
| Arquivo | Tipo | Relacionado a |

## Conclusão
[Avaliação final com raciocínio]

## Próximos Passos
[Ações se REPROVADO, ou confirmação de prontidão se APROVADO]
```

### Etapa 12: Changelog (Pós-Aprovação)

**Só executar se QA = APROVADO.**

Seguir [Keep a Changelog](https://keepachangelog.com/) e [Conventional Commits](https://www.conventionalcommits.org/).

Categorias:
- **Added**: novas features, endpoints, entidades
- **Changed**: modificações em funcionalidades, melhorias de performance
- **Deprecated**: marcadas para remoção
- **Removed**: removidas
- **Fixed**: correções de bugs
- **Security**: patches de vulnerabilidade

Adicionar em `[Unreleased]` no `CHANGELOG.md`:

```markdown
## [Unreleased]

### Added
- [Feature] — QA validated on [YYYY-MM-DD]
  - [Endpoint/capacidade 1]
  - [Endpoint/capacidade 2]
```

Regras:
- NUNCA sobrescrever entradas existentes
- Entradas em **inglês** (padrão da indústria)
- Uma linha por feature, sub-itens opcionais
- Referenciar nome da feature do PRD

### Etapa 13: Versionamento Semântico (Pós-Aprovação)

**Só executar se Etapa 12 completou com sucesso.**

Seguir [SemVer](https://semver.org/):

| Bump | Quando Usar |
|------|-------------|
| **MAJOR** | Breaking changes (endpoints removidos, contratos alterados) |
| **MINOR** | Novas features backward-compatible |
| **PATCH** | Bug fixes, performance, refactoring interno |

Processo:

```bash
# 1. Verificar versão atual
git describe --tags --abbrev=0

# 2. Calcular nova versão
# 3. Atualizar arquivos de versão (version.go, Makefile)
# 4. Mover [Unreleased] para [X.Y.Z] - YYYY-MM-DD no CHANGELOG.md
# 5. Commit
git add -A
git commit -m "chore(release): vX.Y.Z

[Resumo breve da feature validada]"

# 6. Tag
git tag -a vX.Y.Z -m "Release vX.Y.Z"
```

Regras:
- NUNCA fazer bump se QA = REPROVADO
- SEMPRE usar prefixo `v` na tag
- SEMPRE incluir mensagem descritiva

## Critérios de Aprovação — Checklist Final

**APROVADO** requer **TODOS**:

- [ ] 100% dos requisitos funcionais do PRD verificados e passando
- [ ] **ZERO bugs** (qualquer severidade)
- [ ] `go build ./...` sem erros
- [ ] `go vet ./...` limpo
- [ ] `golangci-lint run` sem issues
- [ ] `gofmt -l .` vazio
- [ ] Suite de testes: ZERO falhas
- [ ] Cobertura ≥ 80%
- [ ] Mutation score ≥ 80%
- [ ] Race detector limpo (`-race`)
- [ ] Contrato Swagger ↔ implementação 100% match
- [ ] Auditoria de código sem violações
- [ ] MongoDB, RabbitMQ, Health endpoints saudáveis
- [ ] Graceful shutdown funcional
- [ ] Todas as evidências documentadas em `report/evidence/`
- [ ] Relatório `qa-report.md` gerado

**REPROVADO** se QUALQUER um dos itens acima falhar.

## Diretrizes Operacionais

1. **Compilação antes de testes** — se não compila, nada mais importa
2. **Capturar request/response de TODOS os bugs** — sem exceção
3. **Verificar logs** para panics, erros, warnings inesperados
4. **Testar TODOS os cenários de erro** — não apenas happy paths
5. **Tolerância zero** — bug Baixa reprova igual a Crítica
6. **Ser sistemático** — seguir checklist sequencialmente
7. **Documentar tudo** — sem evidência = não testado
8. **Sempre `-race`** ao rodar testes Go
9. **Validar graceful shutdown** se feature toca lifecycle
10. **Cleanup pós-testes** — parar processos em background
11. **Mutation testing valida testes** — sobreviventes = lacunas
12. **Validar contra skills** [[go-best-practices]] e [[go-testing-best-practices]]
13. **Validar arquitetura** contra `.claude/rules/architecture.md`
14. **Teste sem evidência = NÃO EXECUTADO**

## Ferramentas Essenciais — Cheat Sheet

```bash
# Análise estática
go build ./...
go vet ./...
gofmt -l .
goimports -l .
golangci-lint run ./...
go mod tidy && go mod verify

# Testes
go test -v -race -count=1 -coverprofile=coverage.out ./...
go tool cover -func=coverage.out
go tool cover -html=coverage.out
go test -v -race -tags=integration ./...

# Mutation testing
go install github.com/zimmski/go-mutesting/...@latest
go-mutesting ./...

# Race detection focada
go test -race -run TestName ./...

# Benchmark
go test -bench=. -benchmem ./...

# Profiling
go test -cpuprofile=cpu.prof -memprofile=mem.prof -bench=. ./...

# Swagger
swag init -g cmd/api/main.go --parseDependency --parseInternal

# Health check
curl -s http://localhost:8080/health | jq .

# E2E pattern
curl -s -w "\n%{http_code}" -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{"name":"test","amount":100}'
```

## Anti-Patterns de QA (Evitar)

| Anti-Pattern | Correto |
|---|---|
| "Bug de severidade baixa, aprovo" | Tolerância zero — REPROVADO |
| Pular evidência de teste que passou | Toda evidência é obrigatória |
| Testar só happy path | Testar todos cenários de erro + edge cases |
| Cobertura como única métrica | Cobertura + mutation score + auditoria |
| Aprovar sem rodar `-race` | Sempre rodar com race detector |
| Confiar no Swagger sem verificar | Comparar Swagger vs comportamento real |
| Não documentar request/response do bug | Toda evidência reproduzível |
| Mock testando o mock | Mocks apenas para dependências externas |
| Aprovar com TODO/FIXME pendentes | Limpar TODOs da feature antes de aprovar |
| Bump de versão antes de aprovar | Etapas 12-13 SÓ após APROVADO |
| Fazer cleanup só "se der tempo" | Cleanup é parte do processo |

## Idioma do Relatório

Escrever relatório de QA e documentação de bugs em **Português (Brasil)**. Termos técnicos (tipos Go, métodos HTTP, status codes, nomes de ferramentas) podem permanecer em inglês.

Changelog em **inglês** (padrão da indústria).
