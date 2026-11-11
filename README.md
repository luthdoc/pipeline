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
to-prd  →  prototype + close-prototype (protótipo inicial)  →  to-spec  →  to-tickets (se não couber numa janela)  →  implement
```

Pelos rótulos de uma issue — a segunda linha é a da issue que mexe no que o usuário vê:

```
needs-triage  →  grill-with-issues  →  ready-for-agent (implement) | needs-spec (to-spec)
needs-triage  →  grill-with-issues  →  needs-prototype  →  prototype <N>  →  close-prototype <N>  →  ready-for-agent | needs-spec
```

| Skill | O que faz |
|---|---|
| `setup-project` | Configura o repo: tracker, labels, `## Commands`, `## Fluxo`, remote, template de PR, CI, rótulos de risco. Uma vez por projeto. |
| `to-prd` | Escreve `docs/prd.md` — requisitos e a lista de SPECs. |
| `to-spec` | Transforma a conversa numa SPEC publicada no tracker. |
| `to-tickets` | Fatia uma SPEC em tickets verticais com dependências. |
| `grill-with-issues` | Entrevista sobre uma issue, reescreve como ticket e roteia: `needs-prototype` se mexe no que o usuário vê; senão `ready-for-agent` ou `needs-spec`. |
| `prototype` | Abre a sessão do protótipo e descobre o momento pelo repo: sem link em `docs/agents/prototype.md`, cria o protótipo inicial a partir do PRD; com link e uma issue `needs-prototype`, altera o artifact existente. |
| `close-prototype` | Fecha a sessão do protótipo. No inicial, grava nome e link em `docs/agents/prototype.md` e cria o arquivo de decisões de cada tela em `docs/telas/`. Na alteração, junta issue e protótipo, confirma, grava as decisões em `docs/telas/`, reescreve a issue como ticket citando os IDs e troca `needs-prototype` por `ready-for-agent` ou `needs-spec`. |
| `implement` | Loop implementar → revisar → corrigir → re-revisar, com branch e PR. O PR sai com risco, gates e como desfazer. |

## Modelo por etapa

Para controlar o uso de tokens, cada sessão abre com o modelo da etapa. No Claude Code: `claude --model <id> [--effort medium] --cloud "<instrução>"`. A instrução vem logo depois de `--cloud`; senão dá "Error: --cloud requires a description".

| Etapa | Modelo | Effort |
|---|---|---|
| `grill-me` / `grill-with-issues`, `to-prd`, `to-spec`, `to-tickets` | `claude-sonnet-5-5` | padrão |
| Sessão do protótipo (`/prototype`, até o `/close-prototype`) | `claude-opus-5-5` | `medium` |
| `implement` | `claude-opus-5-5` | padrão |

O modelo vale para a sessão inteira, por isso entra no comando que a abre, e não no frontmatter da skill.

## Sessões na nuvem

Nunca force push. Se a branch da sessão foi apagada no merge, crie uma branch com nome novo a partir da `origin/main` atualizada.

## Escopo pequeno: commit direto na main

Mudança só de texto ou documentação (`.md` e comentários), com até uns 30 linhas e sem código, CI, config nem migration, não abre PR. O agente mostra o diff ao usuário, espera o OK e commita direto na `main`. Todo o resto segue por branch e PR.

## Protótipo

O protótipo vem sempre antes do código em produção. É um artifact do claude.ai, um só por projeto, do tipo Design, com as telas; não vira rota no código. O mesmo comando, `/prototype`, serve aos dois momentos, e a skill descobre qual é pelo estado do repo:

- **Protótipo inicial** — `docs/agents/prototype.md` sem link. A sessão lê o `docs/prd.md` da `origin/main` e cria o artifact. Com `docs/design.md`, segue o design dele; sem, faz um wireframe neutro (cinza, fonte do sistema, imagens como caixa com X), mas com os fluxos da jornada e as especificações do PRD. O `/close-prototype` grava nome e link em `docs/agents/prototype.md` e as decisões de cada tela em `docs/telas/`, depois do OK do diff, e nunca por PR aberto durante os ajustes.
- **Alteração** — já tem link, e a entrada é uma issue `needs-prototype` saída do grill. A sessão altera o artifact existente, e o `/close-prototype <N>` reescreve a issue e a roteia.

### Decisões de tela

O artifact e o código não guardam o porquê de nada, e um não serve para corrigir o outro quando divergem. Por isso cada tela tem um arquivo de decisões em `docs/telas/<tela>.md`, que é a fonte da verdade da tela: o protótipo e o código derivam dele, e quem diverge do arquivo está errado. Tokens e componentes continuam no design doc; regras de negócio, no `docs/prd.md`.

- **Formato** — cada decisão tem ID por tela (`PDV-07`), decisão, por quê, origem, estado (`vale` ou `substituída por`) e onde já está aplicada (protótipo, código). Decisão que muda não é editada: ganha ID novo, e a antiga passa a `substituída por`.
- **Mapa** — `docs/telas/README.md` lista cada tela com o arquivo, o prefixo de ID e as rotas e componentes que ela cobre. É por ele que o revisor sabe a qual tela um arquivo do diff pertence.
- **Quem escreve** — o `setup-project` cria o README com o mapa vazio. O `/close-prototype` grava as decisões aprovadas (no inicial, cria o arquivo de cada tela), com "protótipo: sim, código: não". O `implementer` preenche as rotas e componentes no mapa e marca "código: sim" no mesmo commit que aplica a decisão.
- **Quem lê** — o `/prototype` antes de alterar o artifact, o `implementer` antes de mexer em UI, e o revisor, que cobra pela CL7.

Nos dois momentos, os ajustes vêm pelo chat da sessão: comentário deixado no artifact não chega a ela.

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
- `docs/agents/prototype.md` — nome e link do artifact do claude.ai com as telas do projeto. Gerado pelo `setup-project`; sem link até o `/close-prototype` do protótipo inicial.
- `docs/agents/architecture.md` — mapa de módulos, seams e invariantes, lido pelos agentes. Gerado pelo `setup-project`, mantido pelo `implementer`.
- `docs/telas/` — decisões de tela, uma tela por arquivo, e o mapa tela → rotas e componentes em `docs/telas/README.md`. Fonte da verdade da tela, acima do protótipo e do código. O `setup-project` cria o README; o `/close-prototype` grava as decisões; o `implementer` mantém o mapa e marca o que aplicou.
- `CLAUDE.md` com `@AGENTS.md` — só quando a ferramenta não lê `AGENTS.md` sozinha (ex: Claude Code). Criado pelo `setup-project`.
- `docs/agents/review-rules-projeto.md` — critérios de review da stack ou do design system, caminhos extras de risco do PR, e achado do revisor que se repetiu. O `setup-project` cria o arquivo em todo projeto. O revisor lê os critérios junto com os universais. A lista padrão de risco (migração, auth, cobrança, CI, deploy) mora na skill `implement` e não se rebaixa daqui.
- `.github/pull_request_template.md` — as seções em que o dono decide o merge: Como foi testado, Risco, Como desfazer.
- `.github/workflows/ci.yml` — em todo PR, os gates de `## Commands` (exceto `clean`) e, em projeto Node, `npm audit --audit-level=high`.

## Quando um bug chega na main

Um bug na `main` ou em produção é falha de uma camada. O passo é curto — não é "revisar com mais cuidado" na próxima vez:

1. Diga qual camada deveria ter pego: teste, lint, CI, regra de review, ou o arquivo de instruções (`AGENTS.md` / `CLAUDE.md`).
2. Registre a correção nessa camada. Teste de regressão no repositório. Regra no linter. Passo em `.github/workflows/ci.yml`. Critério em `docs/agents/review-rules-projeto.md` — ou no `review-rules.md` do pipeline, se a falha valer para qualquer stack. Convenção do repo, uma linha no arquivo de instruções.
3. Achado do revisor com a mesma classe em dois PRs sobe para `docs/agents/review-rules-projeto.md` mesmo sem ter chegado na `main`. O `implement` faz isso dentro do loop, sem abrir uma terceira rodada.
