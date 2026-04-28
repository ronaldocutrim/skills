# Minhas Skills do Claude Code

Repositório pessoal das minhas skills do Claude Code. Cada subpasta é uma skill independente, ativada automaticamente pelo Claude quando o contexto da conversa bate com a `description` declarada.

## O que é uma Skill

Uma skill é um pacote de instruções (e opcionalmente scripts/recursos) que o Claude carrega sob demanda. Diferente de uma regra global no `CLAUDE.md`, uma skill **só consome contexto quando é realmente necessária**, porque o Claude lê apenas o `name` + `description` no início da conversa e decide carregar o `SKILL.md` completo só quando faz sentido.

Use skills para:
- **Workflows específicos** — "revisar PR", "preparar release", "investigar bug de produção".
- **Domínios técnicos** — convenções de um framework, regras de um repositório legado, padrões de uma API interna.
- **Capacidades reutilizáveis** — gerar relatórios, manipular CSV, transformar dados, integrar com ferramentas internas.

Não use skills pra:
- Configurar comportamentos automáticos (use **hooks** em `settings.json`).
- Lembrar fatos sobre você ou o projeto (use **memória** em `~/.claude/projects/.../memory/`).
- Atalhos triviais de texto (use **slash commands** customizados).

## Estrutura de uma Skill

Cada skill mora numa pasta própria e precisa de pelo menos um arquivo `SKILL.md`:

```
minha-skill/
├── SKILL.md          # obrigatório — instruções + frontmatter
├── reference.md      # opcional — referência longa carregada sob demanda
├── scripts/          # opcional — scripts executáveis
│   └── helper.sh
└── templates/        # opcional — templates, snippets, exemplos
    └── exemplo.tsx
```

### Frontmatter obrigatório do `SKILL.md`

```markdown
---
name: minha-skill
description: Descrição clara de QUANDO usar a skill. O Claude lê isso pra decidir se carrega o resto do arquivo, então seja específico sobre os gatilhos.
---

# Minha Skill

Conteúdo da skill: instruções, passos, exemplos, regras...
```

**A `description` é a parte mais importante.** Ela é o único texto que o Claude vê antes de decidir carregar a skill. Escreva como gatilho, não como resumo:

- ❌ "Skill que ajuda com testes" — vago, não diz quando ativar.
- ✅ "Use quando o usuário pedir pra escrever, debugar ou refatorar testes Jest/Vitest no frontend. Triggers: 'adicionar teste', 'corrigir teste quebrado', 'cobrir caso X'."

## Como instalar uma skill

Skills podem ficar em três lugares — escolha pela escala:

| Local | Escopo | Quando usar |
|---|---|---|
| `~/.claude/skills/<nome>/` | Usuário (todas as sessões) | Skills pessoais que valem em qualquer projeto |
| `<projeto>/.claude/skills/<nome>/` | Só esse projeto | Skills atreladas a um repositório específico |
| Plugin bundle | Distribuição | Compartilhar com time/comunidade |

Pra ativar uma skill desse repositório como skill pessoal, faça um symlink:

```bash
ln -s /Users/whoami/Workspace/utils/skills/minha-skill ~/.claude/skills/minha-skill
```

Ou copie:

```bash
cp -r /Users/whoami/Workspace/utils/skills/minha-skill ~/.claude/skills/
```

Depois, valide listando as skills disponíveis numa nova sessão do Claude Code — a skill aparece na seção de skills se o frontmatter estiver válido.

## Como criar uma skill nova

1. **Copie o template**: `cp -r example-skill minha-skill-nova`
2. **Edite o `SKILL.md`**: ajuste `name` e `description` (gatilhos!) e escreva o corpo.
3. **Teste a ativação**: instale via symlink, abra uma sessão nova, e mande uma mensagem que deveria disparar a skill. Se o Claude não invocar, a `description` provavelmente está vaga demais.
4. **Itere a `description`** até as ativações ficarem consistentes.

### Boas práticas de conteúdo

- **Comece pelo "quando"**, depois o "como". As primeiras linhas devem confirmar que a skill é apropriada antes de gastar contexto explicando passos.
- **Listas e checklists** funcionam melhor que prosa. O Claude segue procedimentos passo-a-passo bem.
- **Arquivos auxiliares**: pra conteúdo longo (ex.: referência completa de uma API), coloque em `reference.md` e instrua o Claude a ler só quando precisar — economiza contexto.
- **Scripts**: chame por caminho relativo à skill (`scripts/foo.sh`). O Claude resolve o caminho a partir da pasta da skill.
- **Determinismo**: se a skill faz uma operação repetível (linting, geração de boilerplate), prefira chamar um script ao invés de descrever passos em prosa — é mais barato e confiável.

## Como invocar uma skill

O Claude invoca automaticamente quando a `description` casa com o pedido. Você também pode forçar:

- Digitar `/<nome-da-skill>` (se o usuário, não o Claude, invocar).
- Mencionar explicitamente: "use a skill `minha-skill` pra fazer X".

## Skills vs. outros mecanismos

| Quero... | Use |
|---|---|
| Que o Claude faça X automaticamente quando Y acontece | **Hook** (`settings.json`) |
| Que o Claude lembre que eu sou senior dev de Go | **Memória** (`memory/`) |
| Que `/deploy` execute uma sequência fixa de comandos | **Slash command** customizado |
| Encapsular um workflow técnico carregado sob demanda | **Skill** (este repositório) |

## Estrutura deste repositório

```
skills/
├── README.md             # este arquivo
├── .gitignore
└── example-skill/        # template — copie pra começar uma skill nova
    └── SKILL.md
```

## Referências

- [Documentação oficial de Skills (Claude Code)](https://docs.claude.com/en/docs/claude-code/skills)
- [Anatomia de uma skill](https://docs.claude.com/en/docs/claude-code/skills#anatomy-of-a-skill)
