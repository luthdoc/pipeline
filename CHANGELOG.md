# Changelog

## 0.1.0 — 2026-09-24

Primeira versão, extraída do pipeline do projeto `comunidade-nexialistas` (estado da SPEC #37).

- Os agentes viram prompts em markdown puro dentro da skill `implement`, sem frontmatter de ferramenta.
- O orquestrador vira o corpo da skill `implement` e roda na sessão principal; `implementer` e `reviewer` rodam em subagentes novos.
- O playbook `qa-review` é absorvido pelo `reviewer.md`; os critérios viram `implement/review-rules.md`.
- Critérios de stack (Next.js/React/TypeScript e design system) saem dos universais e passam a morar no projeto, em `docs/agents/review-rules-projeto.md`.
- Arquivo de instruções: `AGENTS.md`, com `CLAUDE.md` como alternativa quando não houver `AGENTS.md`.
