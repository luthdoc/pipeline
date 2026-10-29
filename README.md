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

Pelos rótulos de uma issue — a segunda linha é a da issue que mexe no que o usuário vê:

```
needs-triage  →  grill-with-issues  →  ready-for-agent (implement) | needs-spec (to-spec)
needs-triage  →  grill-with-issues  →  needs-prototype  →  sessão do protótipo  →  close-prototype  →  ready-for-agent | needs-spec
```

| Skill | O que faz |
|---|---|
| `setup-project` | Configura o repo: tracker, labels, `## Commands`, `## Fluxo`, remote, template de PR, CI, rótulos de risco. Uma vez por projeto. |
| `to-prd` | Escreve `docs/prd.md` — requisitos e a lista de SPECs. |
| `to-spec` | Transforma a conversa numa SPEC publicada no tracker. |
| `to-tickets` | Fatia uma SPEC em tickets verticais com dependências. |
| `grill-with-issues` | Entrevista sobre uma issue, reescreve como ticket e roteia: `needs-prototype` se mexe no que o usuário vê; senão `ready-for-agent` ou `needs-spec`. |
| `close-prototype` | Na sessão em que o protótipo foi ajustado, junta issue e protótipo, confirma, reescreve a issue como ticket e troca `needs-prototype` por `ready-for-agent` ou `needs-spec`. |
| `implement` | Loop implementar → revisar → corrigir → re-revisar, com branch e PR. O PR sai com risco, gates e como desfazer. |

## Protótipo nas issues visuais

Issue com `needs-prototype`: uma sessão lê a issue e `docs/agents/prototype.md`, altera o artifact do claude.ai indicado ali com as decisões da issue e fica aberta para o usuário ajustar o protótipo conversando nela. Se `docs/agents/prototype.md` estiver sem link, a sessão para e avisa. Ao fim, `/close-prototype` na mesma sessão. O protótipo é um artifact só por projeto, com as telas; não vira rota no código.

## Agentes

Os agentes vivem dentro da skill `implement`, como prompts em markdown puro — sem frontmatter de nenhuma ferramenta:

- `implement/SKILL.md` — o orquestrador. Roda na sessão principal.
- `implement/implementer.md` — implementa via TDD, roda os gates, commita.
- `implement/reviewer.md` — revisa sem ver as notas do implementador.
- `implement/review-rules.md` — os critérios universais da review.

Cada worker roda num subagente novo, com contexto limpo. Em ferramenta sem subagentes, a fase roda em sequência na própria sessão.

## O que fica no projeto

O pipeline é genérico. O que é de cada projeto mora no repo do projeto, gerado pelo `setup-project`:

- **Arquivo de instruções** (`AGENTS.md`, ou `CLAUDE.md` na falta dele) — seções `## Issue Tracker`, `## Commands` e `## Fluxo`. Em repo sem manifesto, `## Commands` nasce sem labels e é preenchida pelo `implementer` na unidade que cria o manifesto; depois, ele a mantém quando muda um script de gate.
- `docs/agents/issue-tracker.md` — comandos do tracker e vocabulário de labels.
- `docs/agents/prototype.md` — nome e link do artifact do claude.ai com as telas do projeto. Gerado pelo `setup-project`; sem link até o projeto ter o artifact.
- `docs/agents/architecture.md` — mapa de módulos, seams e invariantes, lido pelos agentes. Gerado pelo `setup-project`, mantido pelo `implementer`.
- `CLAUDE.md` com `@AGENTS.md` — só quando a ferramenta não lê `AGENTS.md` sozinha (ex: Claude Code). Criado pelo `setup-project`.
- `docs/agents/review-rules-projeto.md` — critérios de review da stack ou do design system, caminhos extras de risco do PR, e achado do revisor que se repetiu. O `setup-project` cria o arquivo em todo projeto. O revisor lê os critérios junto com os universais. A lista padrão de risco (migração, auth, cobrança, CI, deploy) mora na skill `implement` e não se rebaixa daqui.
- `.github/pull_request_template.md` — as seções em que o dono decide o merge: Como foi testado, Risco, Como desfazer.
- `.github/workflows/ci.yml` — em todo PR, os gates de `## Commands` (exceto `clean`) e, em projeto Node, `npm audit --audit-level=high`.

## Quando um bug chega na main

Um bug na `main` ou em produção é falha de uma camada. O passo é curto — não é "revisar com mais cuidado" na próxima vez:

1. Diga qual camada deveria ter pego: teste, lint, CI, regra de review, ou o arquivo de instruções (`AGENTS.md` / `CLAUDE.md`).
2. Registre a correção nessa camada. Teste de regressão no repositório. Regra no linter. Passo em `.github/workflows/ci.yml`. Critério em `docs/agents/review-rules-projeto.md` — ou no `review-rules.md` do pipeline, se a falha valer para qualquer stack. Convenção do repo, uma linha no arquivo de instruções.
3. Achado do revisor com a mesma classe em dois PRs sobe para `docs/agents/review-rules-projeto.md` mesmo sem ter chegado na `main`. O `implement` faz isso dentro do loop, sem abrir uma terceira rodada.
