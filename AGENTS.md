# AGENTS.md

Repositório de **Agent Skills** seguindo o [Agent Skills spec](https://github.com/vercel-labs/skills), compatível com Claude Code, Codex, Gemini CLI, Cursor e demais agentes que suportam o formato.

## Estrutura

Layout categorizado padrão (`skills/<nome>/SKILL.md`):

```
skills/
├── architecture-best-practices/SKILL.md
├── fiber-v3-best-practices/SKILL.md
├── go-best-practices/SKILL.md
├── go-testing-best-practices/SKILL.md
├── nats-best-practices/SKILL.md
├── postgres-best-practices/SKILL.md
├── qa-best-practices/SKILL.md
└── task-reviewer-best-practices/SKILL.md
```

Cada `SKILL.md` segue o formato canônico:

```markdown
---
name: nome-da-skill
description: Descrição usada pelo agente para decidir quando carregar a skill.
---

# Corpo em Markdown
```

## Instalação

```bash
# Todas as skills do repositório
npx skills add douglasdennys45/skills

# Apenas uma skill
npx skills add douglasdennys45/skills/<nome-da-skill>
```

As skills ficam disponíveis em `~/.claude/skills/` (pessoais) ou `.claude/skills/` (escopadas no projeto) após a instalação.

## Catálogo

| Skill | Domínio |
|---|---|
| `architecture-best-practices` | Next.js · Monorepo · Microfrontends · Clean Architecture |
| `fiber-v3-best-practices` | Go · Fiber v3 · HTTP APIs |
| `go-best-practices` | Go · Effective Go · Padrões idiomáticos |
| `go-testing-best-practices` | Go · testify · gomock · testcontainers |
| `nats-best-practices` | NATS · JetStream · Mensageria |
| `postgres-best-practices` | PostgreSQL 18 · Schema · Migrations |
| `qa-best-practices` | QA · Mutation testing · Pipeline de qualidade |
| `task-reviewer-best-practices` | Code review · Clean Architecture · Go |

Detalhes em [README.md](README.md).

## Convenções

- Idioma: **Português (BR)**.
- Frontmatter obrigatório: `name` (kebab-case) e `description` (frase única, específica, indicando *quando* a skill deve ser acionada).
- Skills relacionadas se referenciam via `[[nome-da-skill]]` no corpo.
- Conteúdo baseado em documentação oficial das tecnologias cobertas.

## Licença

Uso livre. Contribuições e forks bem-vindos.
