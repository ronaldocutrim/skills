# Minhas Skills do Claude Code

Coleção pessoal de skills do Claude Code, organizadas por categoria. As skills ficam instaladas em `.claude/skills/` e são rastreadas pelo `skills-lock.json`, que aponta cada skill para o repositório de origem no GitHub.

## Como usar este repo

### Como skills locais (só dentro deste repo)

As skills já estão em `.claude/skills/` — basta abrir o Claude Code dentro deste diretório que elas ficam disponíveis automaticamente.

### Como skills pessoais (em qualquer projeto)

Crie um symlink de cada skill desejada para `~/.claude/skills/`:

```bash
ln -s /Users/whoami/Workspace/utils/skills/.claude/skills/<nome> ~/.claude/skills/<nome>
```

Ou todas de uma vez:

```bash
for skill in /Users/whoami/Workspace/utils/skills/.claude/skills/*/; do
  ln -s "$skill" ~/.claude/skills/$(basename "$skill")
done
```

### Adicionar uma skill nova

1. Copie o template: `cp -r example-skill .claude/skills/minha-skill-nova`
2. Edite `SKILL.md` (especialmente o `description` — é o gatilho que o Claude usa pra decidir quando carregar).
3. Reinicie a sessão pra validar a ativação.

### Atualizar / sincronizar skills externas

O `skills-lock.json` mantém o `source` (repo GitHub), `skillPath` e `computedHash` de cada skill instalada de um repositório externo. Pra atualizar uma skill, refaça o pull do repo de origem e atualize o hash no lockfile.

## Skills por categoria

### Frontend — React
- **react** — padrões de componentes, hooks, useEffect, testing com Vitest
- **vercel-react-best-practices** — performance React/Next.js (Vercel Engineering)
- **vercel-composition-patterns** — compound components, render props, React 19
- **tanstack** — TanStack Query/DB/Form/Router
- **tanstack-query-best-practices** — fetching, cache, mutations, server state
- **zustand** — state management client-side
- **zod** — validação de schema, type inference, safeParse

### UI / Design System
- **shadcn** — patterns shadcn/ui + Radix
- **shadcn-ui** — instalação, configuração, forms com RHF + Zod
- **tailwindcss** — Tailwind v4 patterns
- **frontend-design** — UIs distintas, polidas, sem cara de "AI genérico"
- **ui-ux-pro-max** — paletas, fonts, charts, estilos (glassmorphism, brutalism, etc.)
- **web-design-guidelines** — auditoria de UI/acessibilidade
- **tui-design** — design de interfaces de terminal (Ratatui, Ink, Textual, Bubbletea)
- **opentui** — TUIs com OpenTUI (React/Solid reconciler)

### Mobile — React Native / Expo
- **vercel-react-native-skills** — RN/Expo performance e best practices
- **building-native-ui** — Expo Router, navegação, animations, tabs nativas
- **native-data-fetching** — fetch, React Query, SWR, Expo Router loaders
- **upgrading-expo** — upgrade de SDK Expo e fixes de dependência

### Backend / API / Database
- **hono** — desenvolvimento Hono (CLI, docs, API ref, bundle)
- **drizzle-orm** — schemas, queries, migrations Drizzle
- **drizzle-postgres** — Postgres + Drizzle (RLS, JSONB, indexes, N+1)
- **drizzle-safe-migrations** — migrations production-safe (backfills, constraints)

### AI / Agents
- **ai-sdk** — Vercel AI SDK (generateText, streamText, ToolLoopAgent, tools)
- **mastra** — framework Mastra (agents, workflows, tools, memory, RAG)

### Documentação / Lookup
- **context7** — busca docs técnicas atualizadas de qualquer lib/framework/API
- **sentry-cli** — interagir com Sentry via CLI

### Copywriting / Marketing
- **copywriting** — copy de marketing (homepage, landing, pricing, features)
- **landing-page-copywriter** — landing copy com PAS, AIDA, StoryBrand
- **brand-storytelling** — brand narrative, positioning, messaging

### Meta — criação de skills
- **skill-writer** — guia pra criar skills novas (frontmatter, estrutura)
- **skill-best-practices** — autoria profissional seguindo agentskills.io
- **brainstorming** — explorar intenção/requisitos antes de implementar

## Referências

- [Documentação oficial de Skills (Claude Code)](https://docs.claude.com/en/docs/claude-code/skills)
- [Anatomia de uma skill](https://docs.claude.com/en/docs/claude-code/skills#anatomy-of-a-skill)
