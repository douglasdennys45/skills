---
name: architecture-best-practices
description: "Aplica melhores práticas de arquitetura de diretórios em projetos Next.js com monorepo (Turborepo + pnpm), Module Federation (microfrontends) e Clean Architecture por domínio. Cobre estrutura de apps/packages, regras de nomenclatura (kebab-case para diretórios, PascalCase para classes/componentes, sufixos como DTO/Mapper/Repository), camadas (domain/application/infra/presentation/di), regras de dependência entre camadas e entre domínios, configuração de Module Federation (host shell + remotes), shared packages (ui, shared-types, shared-events, http-client, utils), estado e comunicação cross-domain via Event Bus tipado, roteamento com basePath, estratégia de testes por camada (Vitest/Testing Library/MSW/Playwright), CI/CD com deploy independente por domínio, enforcement de boundaries via ESLint e checklist de criação de novo domínio. Use sempre que: criar/modificar a estrutura de pastas de um app Next.js, decidir onde colocar um arquivo (entity, use case, repository, component), revisar aderência arquitetural, configurar um novo microfrontend, definir contratos entre domínios, configurar Turborepo/pnpm workspaces, configurar Module Federation, decidir entre props/event bus/store global, ou validar PRs contra anti-patterns de arquitetura."
---

# Architecture Best Practices — Next.js Monorepo, Clean Architecture e Microfrontends

Skill de referência para padronização de arquitetura em projetos Next.js com monorepo (Turborepo), Module Federation e Clean Architecture por domínio. Aplique sempre que estiver tomando decisões estruturais sobre onde colocar código, como nomear arquivos/pastas, como comunicar domínios ou como configurar a infraestrutura de build/deploy.

## Quando Aplicar

- Ao criar ou modificar a estrutura de diretórios de um app Next.js
- Ao decidir em qual camada (domain/application/infra/presentation/di) um arquivo deve viver
- Ao criar um novo microfrontend/domínio dentro do monorepo
- Ao configurar Module Federation (host shell e remotes)
- Ao definir comunicação entre domínios (props, Event Bus, store global)
- Ao revisar PRs contra anti-patterns de arquitetura
- Ao configurar ESLint, TypeScript paths, Turborepo, pnpm workspaces
- Ao escolher onde colocar tipos compartilhados, componentes UI, helpers

## 1. Estrutura do Monorepo

```
root/
├── apps/
│   ├── shell/                    # Host principal (layout, nav, auth guard, roteamento)
│   ├── auth/                     # Domínio: autenticação e autorização
│   ├── client/                   # Domínio: gestão de clientes
│   ├── billing/                  # Domínio: cobrança, faturas, planos
│   └── [domain]/                 # Novos domínios seguem o mesmo padrão
│
├── packages/
│   ├── ui/                       # Design system compartilhado
│   ├── shared-types/             # Contratos e tipos entre domínios
│   ├── shared-events/            # Event bus tipado
│   ├── http-client/              # Adapter HTTP base (axios/fetch configurado)
│   ├── utils/                    # Helpers genéricos sem lógica de negócio
│   ├── eslint-config/            # Regras ESLint compartilhadas
│   └── tsconfig/                 # Configurações TypeScript base
│
├── turbo.json
├── pnpm-workspace.yaml
├── package.json
└── .github/workflows/
    ├── ci.yml
    └── deploy-[domain].yml
```

### Regras

- **R1.1** — Cada domínio de negócio é um app Next.js independente dentro de `apps/`.
- **R1.2** — Código compartilhado entre domínios vive exclusivamente em `packages/`.
- **R1.3** — O `shell` é o único host. Nenhum outro app pode ser host.
- **R1.4** — Cada app deve ser executável de forma isolada (`pnpm dev --filter=billing`).
- **R1.5** — Usar `pnpm` como package manager com workspaces.
- **R1.6** — Usar Turborepo para orquestração de builds, com cache habilitado.

## 2. Regras de Nomenclatura

### Diretórios e Arquivos

| Tipo | Convenção | Exemplo |
|---|---|---|
| Diretórios | `kebab-case` | `value-objects/`, `use-cases/` |
| Componentes React | `PascalCase.tsx` | `InvoiceTable.tsx` |
| Hooks | `camelCase.ts` com prefixo `use` | `useCreateInvoice.ts` |
| Use Cases (classe) | `PascalCase.ts` (verbo + substantivo) | `CreateInvoice.ts` |
| Entidades | `PascalCase.ts` | `Invoice.ts` |
| DTOs | `PascalCase.ts` com sufixo `DTO` | `CreateInvoiceDTO.ts` |
| Mappers | `PascalCase.ts` com sufixo `Mapper` | `InvoiceMapper.ts` |
| Repositories (interface) | `PascalCase.ts` com sufixo `Repository` | `InvoiceRepository.ts` |
| Repositories (impl) | `PascalCase.ts` com prefixo `Http` | `HttpInvoiceRepository.ts` |
| Store | `kebab-case.ts` com sufixo `-store` | `billing-store.ts` |
| Tipos/Interfaces | `PascalCase.ts` | `BillingTypes.ts` |
| Constantes | `UPPER_SNAKE_CASE` | `MAX_RETRY_ATTEMPTS` |

### Branches Git

```
feat/[domain]/[descricao]       → feat/billing/add-invoice-export
fix/[domain]/[descricao]        → fix/auth/token-refresh-loop
chore/[domain]/[descricao]      → chore/shell/upgrade-next-15
refactor/[domain]/[descricao]   → refactor/client/extract-use-cases
```

## 3. Clean Architecture por Domínio

Cada app em `apps/` segue a mesma estrutura interna:

```
apps/[domain]/src/
├── domain/                      # Núcleo — zero dependências externas
│   ├── entities/                # Entidades com regras de negócio
│   ├── value-objects/           # Objetos de valor imutáveis
│   ├── errors/                  # Erros de domínio tipados
│   └── repositories/            # Interfaces (ports) — só contratos
│
├── application/                 # Orquestração — depende só do domain
│   ├── use-cases/               # Casos de uso (1 classe = 1 ação)
│   ├── dtos/                    # Input/Output dos use cases
│   └── mappers/                 # Conversão Entity <-> DTO
│
├── infra/                       # Implementações concretas
│   ├── repositories/            # Implementam interfaces do domain
│   ├── gateways/                # Integrações externas (Stripe, etc)
│   ├── http/                    # Configuração HTTP do domínio
│   └── store/                   # Estado local (Zustand/etc)
│
├── presentation/                # React/Next.js — camada mais externa
│   ├── pages/ ou app/           # Rotas do Next.js
│   ├── components/              # Componentes internos do domínio
│   ├── exposed/                 # Componentes expostos via Module Federation
│   ├── hooks/                   # Conectam UI aos use cases
│   ├── providers/               # Context providers do domínio
│   └── layouts/                 # Layouts do domínio
│
└── di/                          # Injeção de dependência
    └── container.ts             # Composição: instancia adapters e injeta nos use cases
```

### Regras

- **R3.1** — `domain/` não importa NENHUMA lib externa (nem React, nem Axios, nem Zustand). Somente TypeScript puro.
- **R3.2** — `application/` importa somente de `domain/`. Use cases recebem interfaces via construtor (DI).
- **R3.3** — `infra/` implementa as interfaces definidas em `domain/repositories/`.
- **R3.4** — `presentation/` nunca importa de `domain/` diretamente. Acessa dados via DTOs retornados pelos use cases.
- **R3.5** — `di/container.ts` é o único arquivo que conhece todas as camadas. Ele instancia adapters e injeta nos use cases.
- **R3.6** — Cada use case é uma classe com um único método público `execute()`.
- **R3.7** — Entidades encapsulam regras de negócio. Validações de negócio vivem na entidade, não no componente.
- **R3.8** — Value Objects são imutáveis. Toda mutação retorna uma nova instância.
- **R3.9** — DTOs são objetos simples (interfaces). Não possuem métodos nem lógica.
- **R3.10** — Mappers são classes com métodos estáticos. Não possuem estado.

## 4. Regras de Dependência entre Camadas

```
PERMITIDO (→ = "pode importar de")

  presentation → application (via hooks → use cases)
  presentation → di (para obter instâncias)
  infra        → domain (para implementar interfaces)
  infra        → application (para usar DTOs se necessário)
  application  → domain (para usar entidades e interfaces)
  di           → todas as camadas (é o compositor)

PROIBIDO (✗)

  domain       ✗ application, infra, presentation
  application  ✗ infra, presentation
  infra        ✗ presentation
  presentation ✗ domain diretamente (use DTOs)
```

### Tabela Resumo

| Camada | Importa de | Nunca importa de |
|---|---|---|
| `domain/` | Nada (só ela mesma) | `application/`, `infra/`, `presentation/` |
| `application/` | `domain/` | `infra/`, `presentation/` |
| `infra/` | `domain/`, `application/` | `presentation/` |
| `presentation/` | `application/`, `di/` | `domain/` direto |
| `di/` | Todas | — |

## 5. Regras de Dependência entre Domínios

```
PERMITIDO
  shell   → qualquer remote (consome componentes expostos)
  remote  → packages/* (libs compartilhadas)
  remote  → shared-events (comunicação desacoplada)

PROIBIDO
  remote  ✗ outro remote (NUNCA importação direta)
  remote  ✗ shell (NUNCA depender do host)
  domain logic em packages/ (packages não contém lógica de negócio)
```

### Regras

- **R5.1** — Domínios NUNCA importam diretamente uns dos outros.
- **R5.2** — Se billing precisa de dados do auth, a comunicação ocorre via Event Bus ou props passadas pelo shell.
- **R5.3** — Tipos compartilhados entre domínios vivem em `packages/shared-types/`, não dentro de nenhum domínio.
- **R5.4** — `packages/` contém apenas código utilitário e contratos. Lógica de negócio só existe dentro de `apps/[domain]/src/domain/`.

## 6. Module Federation

### Configuração do Host (Shell)

```js
// apps/shell/next.config.js
const { NextFederationPlugin } = require('@module-federation/nextjs-mf');

module.exports = {
  webpack(config) {
    config.plugins.push(
      new NextFederationPlugin({
        name: 'shell',
        remotes: {
          auth:    `auth@${process.env.AUTH_URL}/_next/static/chunks/remoteEntry.js`,
          client:  `client@${process.env.CLIENT_URL}/_next/static/chunks/remoteEntry.js`,
          billing: `billing@${process.env.BILLING_URL}/_next/static/chunks/remoteEntry.js`,
        },
        shared: {
          react: { singleton: true, requiredVersion: '^18.0.0' },
          'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
          '@company/ui': { singleton: true },
          zustand: { singleton: true },
        },
      })
    );
    return config;
  },
};
```

### Configuração do Remote

```js
// apps/[domain]/next.config.js
new NextFederationPlugin({
  name: '[domain]',
  filename: 'static/chunks/remoteEntry.js',
  exposes: {
    // SOMENTE componentes da pasta exposed/
    './ComponentName': './src/presentation/exposed/ComponentName',
  },
  shared: { /* mesma config do host */ },
});
```

### Regras

- **R6.1** — Somente componentes em `presentation/exposed/` podem ser listados em `exposes`.
- **R6.2** — Componentes expostos devem ser auto-contidos (não dependem de providers do domínio pai).
- **R6.3** — Componentes expostos devem ter fallback de loading e error boundary próprios.
- **R6.4** — O consumo de remotes no host usa `next/dynamic` com `ssr: false` como padrão.
- **R6.5** — URLs dos remotes vêm de variáveis de ambiente, nunca hardcoded.
- **R6.6** — Toda dependência singleton deve ter a mesma `requiredVersion` em todos os apps.
- **R6.7** — Nunca exponha páginas inteiras se só precisa de um componente. Exponha o mínimo necessário.

## 7. Shared Packages

### `packages/ui` — Design System

```
packages/ui/
├── src/
│   ├── components/
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   ├── Button.styles.ts
│   │   │   ├── Button.test.tsx
│   │   │   └── index.ts
│   │   ├── Input/
│   │   ├── Modal/
│   │   └── ...
│   ├── tokens/
│   │   ├── colors.ts
│   │   ├── spacing.ts
│   │   └── typography.ts
│   └── index.ts
├── package.json
└── tsconfig.json
```

### `packages/shared-types` — Contratos

```ts
// packages/shared-types/src/user.ts
export interface AuthenticatedUser {
  id: string;
  email: string;
  role: 'admin' | 'user' | 'viewer';
  name: string;
}

// packages/shared-types/src/events.ts
export type DomainEvents = {
  'auth:login': { user: AuthenticatedUser };
  'auth:logout': void;
  'billing:plan-upgraded': { planId: string; userId: string };
  'client:selected': { clientId: string };
};
```

### `packages/shared-events` — Event Bus

```ts
// packages/shared-events/src/event-bus.ts
import type { DomainEvents } from '@company/shared-types';

class EventBus {
  private target = new EventTarget();

  emit<K extends keyof DomainEvents>(event: K, data: DomainEvents[K]): void {
    this.target.dispatchEvent(new CustomEvent(event, { detail: data }));
  }

  on<K extends keyof DomainEvents>(
    event: K,
    callback: (data: DomainEvents[K]) => void,
  ): () => void {
    const handler = (e: Event) => callback((e as CustomEvent).detail);
    this.target.addEventListener(event, handler);
    return () => this.target.removeEventListener(event, handler);
  }
}

export const eventBus = new EventBus();
```

### Regras

- **R7.1** — `packages/ui` contém apenas componentes visuais. Zero lógica de negócio.
- **R7.2** — `packages/shared-types` contém apenas `type` e `interface`. Nenhuma implementação.
- **R7.3** — Novos eventos devem ser adicionados ao type `DomainEvents` antes de serem emitidos.
- **R7.4** — `packages/utils` contém apenas funções puras e genéricas (formatDate, debounce, etc).
- **R7.5** — Nenhum package pode importar de `apps/`. A dependência é unidirecional: apps → packages.
- **R7.6** — Todo package exporta via barrel file (`index.ts`). Imports internos de package não são permitidos (ex: `@company/ui/src/Button` é proibido).

## 8. Estado e Comunicação entre Domínios

### Hierarquia de preferência

1. **Props** — quando o shell passa dados diretamente ao remote
2. **Event Bus** — quando domínios precisam reagir a eventos de outros
3. **Shared Store (global)** — apenas para estado verdadeiramente global (usuário logado, tema)
4. **URL/Query Params** — para estado navegável

### Regras

- **R8.1** — Cada domínio gerencia seu próprio estado interno. Não existe store global por domínio.
- **R8.2** — O único estado global permitido é: usuário autenticado, tema, e idioma. Tudo mais é local.
- **R8.3** — Comunicação entre domínios usa o Event Bus tipado. Nunca chamadas diretas.
- **R8.4** — Eventos são fire-and-forget. Se precisa de resposta, use request/response via props ou callbacks.
- **R8.5** — Não armazene estado derivável. Se pode ser calculado a partir de outro estado, calcule.

## 9. Roteamento

### Estrutura de Rotas

```
shell       → / (redirect), /dashboard
auth        → /auth/login, /auth/signup, /auth/forgot-password
client      → /clients, /clients/:id, /clients/:id/edit
billing     → /billing, /billing/invoices, /billing/plans
```

### Regras

- **R9.1** — Cada domínio usa `basePath` no `next.config.js` correspondente ao seu prefixo de rota.
- **R9.2** — O shell configura `rewrites` para rotear para o domínio correto.
- **R9.3** — Links entre domínios usam `<a href>` ou `window.location`, não `next/link` (apps diferentes).
- **R9.4** — Links internos do domínio usam `next/link` normalmente.
- **R9.5** — Rotas protegidas são guardadas pelo shell (auth guard), não pelo domínio individual.

## 10. Testes

### Estratégia por Camada

| Camada | Tipo de Teste | Ferramenta | Cobertura Mínima |
|---|---|---|---|
| `domain/` | Unitário | Vitest | 90% |
| `application/` | Unitário | Vitest | 85% |
| `infra/` | Integração | Vitest + MSW | 75% |
| `presentation/` | Componente | Testing Library | 70% |
| E2E (críticos) | End-to-end | Playwright | Fluxos principais |

### Estrutura de Testes

```
apps/billing/src/
├── domain/entities/
│   ├── Invoice.ts
│   └── __tests__/Invoice.test.ts              # Testa regras de negócio puras
├── application/use-cases/
│   ├── CreateInvoice.ts
│   └── __tests__/CreateInvoice.test.ts        # Mock do repository (interface)
├── infra/repositories/
│   ├── HttpInvoiceRepository.ts
│   └── __tests__/HttpInvoiceRepository.test.ts # MSW para mockar HTTP
└── presentation/components/
    └── InvoiceTable/
        ├── InvoiceTable.tsx
        └── InvoiceTable.test.tsx              # Testing Library
```

### Regras

- **R10.1** — Testes de `domain/` não usam mocks. Entidades são testadas com dados reais.
- **R10.2** — Testes de `application/` mockam apenas as interfaces de repository (nunca implementações).
- **R10.3** — Testes de `infra/` usam MSW para interceptar chamadas HTTP. Nunca mock do axios/fetch.
- **R10.4** — Testes de `presentation/` testam comportamento do usuário, não implementação.
- **R10.5** — Cada domínio roda seus testes de forma independente (`pnpm test --filter=billing`).
- **R10.6** — Testes E2E cobrem apenas fluxos críticos cross-domain (login → criar fatura → pagar).
- **R10.7** — Não teste getters, setters ou código trivial. Teste lógica e comportamento.

## 11. CI/CD e Deploy

### Pipeline

```yaml
# .github/workflows/ci.yml
# 1. Detecta quais apps/packages mudaram
# 2. Roda lint e type-check nos afetados
# 3. Roda testes unitários nos afetados
# 4. Build dos afetados
# 5. Testes E2E (se fluxos críticos foram tocados)
# 6. Deploy independente por domínio
```

### Regras

- **R11.1** — Cada domínio tem deploy independente. Mudança no billing não faz redeploy do auth.
- **R11.2** — Turborepo `--filter` detecta o que mudou. Só roda CI nos apps afetados.
- **R11.3** — Mudanças em `packages/` disparam CI de todos os apps que dependem daquele package.
- **R11.4** — Mudanças no `shell` disparam E2E completo (é o orquestrador).
- **R11.5** — Cada app tem sua própria URL de deploy (auth.company.com, billing.company.com, etc).
- **R11.6** — Variáveis de ambiente de URLs dos remotes são atualizadas no shell a cada deploy.
- **R11.7** — Feature flags por domínio para releases graduais. Não fazer big-bang deploy.

## 12. ESLint e Enforcement

### Regras de Import Boundaries

```js
// packages/eslint-config/rules/architecture.js
module.exports = {
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [
        // Domain não importa nada externo
        {
          group: ['react', 'react-dom', 'next/*', 'axios', 'zustand', '@tanstack/*'],
          message: 'A camada domain/ não pode importar libs externas.',
        },
      ],
    }],
  },

  overrides: [
    // Domain: só importa de domain
    {
      files: ['**/domain/**/*.ts'],
      rules: {
        'no-restricted-imports': ['error', {
          patterns: [
            { group: ['**/application/**'], message: 'domain/ não importa de application/' },
            { group: ['**/infra/**'], message: 'domain/ não importa de infra/' },
            { group: ['**/presentation/**'], message: 'domain/ não importa de presentation/' },
            { group: ['**/di/**'], message: 'domain/ não importa de di/' },
          ],
        }],
      },
    },

    // Application: só importa de domain
    {
      files: ['**/application/**/*.ts'],
      rules: {
        'no-restricted-imports': ['error', {
          patterns: [
            { group: ['**/infra/**'], message: 'application/ não importa de infra/' },
            { group: ['**/presentation/**'], message: 'application/ não importa de presentation/' },
          ],
        }],
      },
    },

    // Infra: não importa de presentation
    {
      files: ['**/infra/**/*.ts'],
      rules: {
        'no-restricted-imports': ['error', {
          patterns: [
            { group: ['**/presentation/**'], message: 'infra/ não importa de presentation/' },
          ],
        }],
      },
    },

    // Remotes não importam de outros remotes
    {
      files: ['apps/auth/**', 'apps/client/**', 'apps/billing/**'],
      rules: {
        'no-restricted-imports': ['error', {
          patterns: [
            { group: ['apps/shell/**'], message: 'Remote não importa do shell' },
            { group: ['apps/auth/**', 'apps/client/**', 'apps/billing/**'],
              message: 'Remotes não importam uns dos outros' },
          ],
        }],
      },
    },
  ],
};
```

### Regras

- **R12.1** — ESLint com regras de boundary roda em todo PR. PR não faz merge se violar boundaries.
- **R12.2** — TypeScript strict mode habilitado em todos os apps e packages.
- **R12.3** — `noImplicitAny: true` — nenhum `any` implícito permitido.
- **R12.4** — Imports devem usar path aliases (`@domain/`, `@application/`, `@infra/`, `@presentation/`).

## 13. Checklist de Novo Domínio

Ao criar um novo microfrontend/domínio, percorra a lista:

```
[ ] 1. Criar app em apps/[domain]/ seguindo a estrutura padrão
[ ] 2. Configurar next.config.js com basePath e NextFederationPlugin
[ ] 3. Configurar tsconfig.json com path aliases das camadas
[ ] 4. Criar di/container.ts com as dependências do domínio
[ ] 5. Definir entidades e interfaces de repository em domain/
[ ] 6. Implementar pelo menos 1 use case em application/
[ ] 7. Implementar repositories em infra/
[ ] 8. Criar componentes expostos em presentation/exposed/
[ ] 9. Registrar remote no shell (next.config.js + env vars)
[ ] 10. Adicionar rewrite no shell para o basePath do novo domínio
[ ] 11. Adicionar eventos do domínio em packages/shared-types DomainEvents
[ ] 12. Configurar pipeline de deploy separado
[ ] 13. Adicionar testes unitários para domain/ e application/
[ ] 14. Documentar rotas e componentes expostos no README do domínio
[ ] 15. Validar que o app roda isolado (pnpm dev --filter=[domain])
```

## 14. Anti-patterns

| Anti-pattern | Por que é ruim | O que fazer |
|---|---|---|
| Importar de remote para remote | Acoplamento direto, quebra independência | Usar Event Bus ou props via shell |
| Lógica de negócio em componente | Não testável, não reutilizável | Mover para `domain/entities` ou `application/use-cases` |
| Lógica de negócio em packages/ | Package vira domínio oculto | Manter lógica dentro de `apps/[domain]/domain/` |
| Store global por domínio | Estado espalhado, difícil debugar | Estado global só para user, tema, idioma |
| Expor componentes internos via MF | Over-sharing, acoplamento | Só expor o que está em `exposed/` |
| `any` em DTOs | Perde type safety entre camadas | Tipar DTOs como interfaces explícitas |
| Use case com dependência de React | Application acoplada ao framework | Use case só usa TypeScript puro |
| Repository sem interface | Não pode trocar implementação, não pode testar | Sempre definir interface em `domain/` |
| Componente consumindo API direto | Pula todas as camadas, não testável | Componente → hook → use case → repository |
| Shared state para dados de um domínio | Acoplamento desnecessário | Cada domínio gerencia seu estado |
| Testes que testam implementação | Quebram a cada refactor | Testar comportamento e contratos |
| Deploy de todos os apps juntos | Anula benefício de microfrontends | Deploy independente por domínio |

## 15. Decisão Rápida — Onde Colocar Cada Arquivo

Use esta tabela como referência rápida ao escrever um novo arquivo:

| Tipo de código | Localização | Exemplo |
|---|---|---|
| Regra de negócio pura | `apps/[domain]/src/domain/entities/` | `Invoice.ts` (calcula total, valida status) |
| Valor imutável | `apps/[domain]/src/domain/value-objects/` | `Money.ts`, `Email.ts` |
| Erro tipado de domínio | `apps/[domain]/src/domain/errors/` | `InvoiceNotFoundError.ts` |
| Contrato de repositório | `apps/[domain]/src/domain/repositories/` | `InvoiceRepository.ts` (interface) |
| Caso de uso (1 ação) | `apps/[domain]/src/application/use-cases/` | `CreateInvoice.ts` (classe com `execute()`) |
| Input/Output | `apps/[domain]/src/application/dtos/` | `CreateInvoiceDTO.ts` |
| Conversão Entity↔DTO | `apps/[domain]/src/application/mappers/` | `InvoiceMapper.ts` |
| Implementação HTTP de repo | `apps/[domain]/src/infra/repositories/` | `HttpInvoiceRepository.ts` |
| Integração externa | `apps/[domain]/src/infra/gateways/` | `StripeGateway.ts` |
| Estado local do domínio | `apps/[domain]/src/infra/store/` | `billing-store.ts` |
| Componente interno | `apps/[domain]/src/presentation/components/` | `InvoiceTable.tsx` |
| Componente exposto via MF | `apps/[domain]/src/presentation/exposed/` | `BillingWidget.tsx` |
| Hook que chama use case | `apps/[domain]/src/presentation/hooks/` | `useCreateInvoice.ts` |
| Página/rota Next.js | `apps/[domain]/src/presentation/pages/` ou `app/` | `invoices/page.tsx` |
| Composição de dependências | `apps/[domain]/src/di/container.ts` | (instancia repo + injeta no use case) |
| Componente visual genérico | `packages/ui/src/components/` | `Button.tsx` |
| Tipo compartilhado entre domínios | `packages/shared-types/src/` | `AuthenticatedUser.ts` |
| Evento cross-domain | `packages/shared-types/src/events.ts` (`DomainEvents`) | `billing:plan-upgraded` |
| Helper puro genérico | `packages/utils/src/` | `formatDate.ts`, `debounce.ts` |
| Configuração HTTP base | `packages/http-client/src/` | `createHttpClient.ts` |
| Regras ESLint comuns | `packages/eslint-config/` | `architecture.js` |
| Tsconfig base | `packages/tsconfig/` | `base.json`, `next.json` |

## Resumo Visual

```
┌─────────────────── MONOREPO ───────────────────────┐
│                                                     │
│  apps/shell ←── remotes ──→ apps/[domain]           │
│      │                          │                   │
│      │         ┌────────────────┤                   │
│      │         │   Clean Arch   │                   │
│      │         │                │                   │
│      │         │  presentation  │ ← React/Next.js   │
│      │         │       ↓        │                   │
│      │         │  application   │ ← Use Cases       │
│      │         │       ↓        │                   │
│      │         │    domain      │ ← Entidades puras │
│      │         │       ↑        │                   │
│      │         │    infra       │ ← HTTP, Storage   │
│      │         │                │                   │
│      │         │     di         │ ← Cola tudo       │
│      │         └────────────────┘                   │
│      │                                              │
│      └──────── packages/ ───────────────────────    │
│                (ui, types, events, utils)           │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

> **Versão:** 1.0
> **Última atualização:** 2026-05-27
