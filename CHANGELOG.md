# Changelog

## 0.2.0 — 2026-09-24

Correções dos 11 WARN do audit de ponta a ponta sobre a 0.1.0.

- **Diff da unidade:** o orquestrador registra a base antes de cada implementer e revisa `<base>..HEAD`, não `HEAD~1`. A retomada deriva a base dos commits `(#N)`.
- **Contrato do tracker:** o `setup-project` documenta "Listar tickets da spec" (com o corpo) e "Ver pai". O `implement` localiza a cópia da SPEC direto em `.scratch/`, sem comando do tracker.
- **Bloqueio:** a seção `## Blocked by` do corpo é a fonte única. O `to-tickets` não cria relação nativa de bloqueio.
- **`architecture.md`:** gerado pelo `setup-project` (nova Parte 5). Os leitores toleram a ausência, e o `implementer` o cria se faltar.
- **Portabilidade:** nenhum comando usa composição de shell (`||`, `$(...)`, `2>/dev/null`). O checkout da branch vira passos e também checa `origin/<branch>`. Todo corpo vai por `--body-file`.
- **SPEC fatiada:** espelha `in-progress` no primeiro ticket e `in-review` ao abrir o PR. A detecção de estado ignora essa label.
- **Workers:** recebem a raiz absoluta do repositório.
- **Claude Code:** o `setup-project` cria `CLAUDE.md` com `@AGENTS.md` quando as skills estão em `.claude/skills/`.
- **Sem perguntas de código ao humano:** seams (`to-spec`), quebra em tickets (`to-tickets`), script `clean` e critérios de projeto (`setup-project`) são decididos pelo agente e informados.
- **RED:** um AC já satisfeito por task anterior da mesma peça mantém o teste como guarda de regressão, com o `arquivo:linha` no retorno.

## 0.1.0 — 2026-09-24

Primeira versão, extraída do pipeline do projeto `comunidade-nexialistas` (estado da SPEC #37).

- Os agentes viram prompts em markdown puro dentro da skill `implement`, sem frontmatter de ferramenta.
- O orquestrador vira o corpo da skill `implement` e roda na sessão principal; `implementer` e `reviewer` rodam em subagentes novos.
- O playbook `qa-review` é absorvido pelo `reviewer.md`; os critérios viram `implement/review-rules.md`.
- Critérios de stack (Next.js/React/TypeScript e design system) saem dos universais e passam a morar no projeto, em `docs/agents/review-rules-projeto.md`.
- Arquivo de instruções: `AGENTS.md`, com `CLAUDE.md` como alternativa quando não houver `AGENTS.md`.
