# pipeline

Pipeline de desenvolvimento com agentes de IA: do PRD ao PR, com implementação e review independentes. Agnóstico de ferramenta — funciona em Claude Code, Codex, Cursor e em qualquer agente que leia skills no formato `SKILL.md`.

Instala **por projeto**, nunca globalmente: um repo sem o pipeline não carrega nenhuma destas skills no contexto.

## Instalar

Na raiz do projeto:

```bash
npx skills add luthdoc/pipeline
```

Atualizar para a versão mais recente:

```bash
npx skills update -p
```

Depois, uma vez por projeto, rode a skill `setup-project`.

## Fluxo

```
to-prd  →  to-spec  →  to-tickets (se não couber numa janela)  →  implement
```

| Skill | O que faz |
|---|---|
| `setup-project` | Configura o repo: tracker, labels, `## Commands`, `## Fluxo`, remote. Uma vez por projeto. |
| `to-prd` | Escreve `docs/prd.md` — requisitos e a lista de SPECs. |
| `to-spec` | Transforma a conversa numa SPEC publicada no tracker. |
| `to-tickets` | Fatia uma SPEC em tickets verticais com dependências. |
| `implement` | Loop implementar → revisar → corrigir → re-revisar, com branch e PR. |

## Agentes

Os agentes vivem dentro da skill `implement`, como prompts em markdown puro — sem frontmatter de nenhuma ferramenta:

- `implement/SKILL.md` — o orquestrador. Roda na sessão principal.
- `implement/implementer.md` — implementa via TDD, roda os gates, commita.
- `implement/reviewer.md` — revisa sem ver as notas do implementador.
- `implement/review-rules.md` — os critérios universais da review.

Cada worker roda num subagente novo, com contexto limpo. Em ferramenta sem subagentes, a fase roda em sequência na própria sessão.

## O que fica no projeto

O pipeline é genérico. O que é de cada projeto mora no repo do projeto, gerado pelo `setup-project`:

- **Arquivo de instruções** (`AGENTS.md`, ou `CLAUDE.md` na falta dele) — seções `## Issue Tracker`, `## Commands` e `## Fluxo`.
- `docs/agents/issue-tracker.md` — comandos do tracker e vocabulário de labels.
- `docs/agents/architecture.md` — mapa de módulos, seams e invariantes. Mantido pelo `implementer`.
- `docs/agents/review-rules-projeto.md` *(opcional)* — critérios de review específicos da stack ou do design system do projeto. O revisor lê junto com os universais.
