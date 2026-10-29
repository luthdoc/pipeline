---
name: setup-project
description: >
  Configura um repositório para o pipeline: issue tracker e vocabulário de labels,
  a seção `## Commands` (lint/typecheck/test/build/clean) que o implementer lê como
  gate, a seção `## Fluxo` com as saídas recomendadas depois de um grill, o remote,
  o template de PR, a CI com esses gates e `npm audit`, os rótulos de risco e o
  `review-rules-projeto.md`. Rode uma vez por repositório, antes do primeiro uso de
  to-spec, to-tickets ou implement. Use quando o usuário disser "setup do projeto",
  "setup-project", "configura o tracker", "prepara o repo para o pipeline" ou variações.
disable-model-invocation: true
---

# Skill: setup-project

Você configura, uma vez por repositório, os contratos que o resto do pipeline lê em runtime. Sem eles as outras skills param com erro — de propósito, para falhar alto em vez de adivinhar.

O output são os arquivos abaixo. Os três primeiros são o contrato que o resto do pipeline lê; o restante é o que faz o PR chegar decidível sem abrir o diff.

- **arquivo de instruções** — três seções curtas: `## Issue Tracker`, `## Commands`, `## Fluxo`
- **`docs/agents/issue-tracker.md`** — o detalhe pesado (comandos do tracker, tabela de labels)
- **`docs/agents/architecture.md`** — o mapa do código que `to-spec`, `to-tickets`, `implementer` e revisor leem (Parte 5)
- **`docs/agents/review-rules-projeto.md`** — critérios deste projeto e os caminhos extras de risco (Parte 6). Sempre criado.
- **`.github/pull_request_template.md`** — as seções Como foi testado, Risco e Como desfazer (Parte 7)
- **`.github/workflows/ci.yml`** — os gates de `## Commands` e, em projeto Node, `npm audit` (Parte 7)

**Qual é o arquivo de instruções:** `AGENTS.md` na raiz — o formato que a maioria das ferramentas de agente lê. Se o repo já tem `CLAUDE.md` e não tem `AGENTS.md`, use o `CLAUDE.md` existente em vez de criar um segundo arquivo.

**Ferramenta que não lê `AGENTS.md` sozinha:** crie o ponteiro — não basta avisar, porque sem ele a seção `## Fluxo` nunca entra no contexto. Descubra a ferramenta pela pasta onde as skills do pipeline foram instaladas. Skills em `.claude/skills/` → é o Claude Code: crie `CLAUDE.md` na raiz com a linha `@AGENTS.md`, ou acrescente essa linha ao `CLAUDE.md` existente se ela faltar. Outra ferramenta com arquivo próprio → a mesma ideia, na convenção dela. Informe o que criou.

O arquivo de instruções entra no contexto de toda conversa — mantê-lo enxuto importa. O detalhe fica isolado num arquivo que só quem precisa vai ler.

Sete partes, nesta ordem.

**O que você pergunta, e o que não.** O dono do repositório pode não ler código. Pergunte só decisões de processo: qual tracker, qual vocabulário de labels, qual o limiar do `## Fluxo`. O que é técnico — comandos do tracker, script `clean`, mapa do código, critérios de review — você decide, grava e informa em uma linha, sem pedir aprovação de rascunho.

## Portabilidade dos comandos — regra dura

O pipeline roda em Bash e em PowerShell. Todo comando que você documentar — e todo comando que outra skill lê daqui — tem que rodar nos dois:

- **Nada de composição de shell:** `||`, `&&`, `$(...)`, `2>/dev/null`, pipe, `VAR=...`. Quando um passo depende do resultado do anterior, documente como passos separados: "rode X, leia a saída, use-a em Y".
- **Corpo de issue, comentário e PR vai por `--body-file <arquivo>`**, nunca `--body "..."`. Markdown tem crase, aspas, `$`, `!` e `&`, e escapar isso inline quebra de jeitos diferentes em cada shell. O arquivo fica em `.scratch/`.

## O modelo de artefatos

Independente do tracker, a hierarquia é sempre a mesma — muda só como ela é materializada:

| Conceito | Gerado por | GitHub / Linear | Markdown local |
|---|---|---|---|
| **Spec** — um objetivo inteiro | `to-spec` | Issue | Arquivo no nível da feature |
| **Ticket** — uma fatia vertical da spec | `to-tickets` | Sub-issue daquela issue | Arquivo em `issues/` dentro da pasta da feature |

Com **tracker real**, a issue é canônica. `to-spec` e `to-tickets` também deixam uma cópia de trabalho em `.scratch/<slug>/`, que é ignorada pelo git — serve para organizar e reler a unidade em curso sem uma chamada de rede a cada leitura. Cópia de trabalho não é fonte de verdade: divergiu, vale a issue, e `.scratch/` pode ser apagado a qualquer momento sem perda.

Com **markdown local**, o arquivo em `.scratch/` é a própria fonte de verdade, e nada é publicado.

Em nenhum dos dois modos existe uma segunda cópia versionada no repo. Documentação de spec commitada em `docs/` diverge da issue e depois ninguém sabe qual vale.

## O que o arquivo de instruções guarda — regra canônica

**O arquivo de instruções não guarda fato que envelhece.**

O critério não é o tamanho da seção — é **se aquela informação envelhece sozinha**. Decisão estável fica; fato que muda a cada PR vira ponteiro.

| Tipo de informação | Envelhece? | Onde mora |
|---|---|---|
| Como o pipeline roda neste repo (`## Issue Tracker`, `## Commands`, `## Fluxo`) | não | arquivo de instruções |
| Como o deploy funciona; o resumo do design system que precede toda tarefa de UI | não | arquivo de instruções, curto, com ponteiro para o documento completo |
| O que o produto é, e quais são os requisitos | **sim** | `docs/prd.md` — o arquivo de instruções aponta |
| Estado atual do código: módulos, seams, invariantes | **sim** | `docs/agents/architecture.md` — o arquivo de instruções aponta |
| O que já foi construído, quantas telas existem, o que falta | **sim** | não mora em lugar nenhum: é o histórico do git e o quadro de issues |

**Por que a regra existe:** fato que muda a cada PR ("este repositório ainda não tem código", "o produto é uma landing de guias grátis") apodrece sozinho, e o arquivo de instruções não tem dono que o mantenha. Misturado com decisão estável, ele acaba descrevendo um produto que não existe mais e contradizendo o PRD real.

Ao gravar ou reconfigurar: se você encontrar uma seção `## Status`, `## Product` ou equivalente descrevendo o que já existe no código, **substitua por um ponteiro** e mova o conteúdo para o documento que tem dono.

## Parte 1 — Issue tracker

### 1. Explorar antes de perguntar

Antes de fazer qualquer pergunta, levante o estado atual do repositório:

- `git remote -v` — há remote? Aponta para GitHub, GitLab, ou outro host?
- `AGENTS.md` ou `CLAUDE.md` na raiz — já existe uma seção `## Issue Tracker`? Se sim, essa é uma reconfiguração, não um setup do zero.
- `docs/agents/issue-tracker.md` — já existe, mesmo que a seção no arquivo de instruções tenha sido apagada? Sinal de configuração anterior a recuperar ou substituir.
- `.scratch/` — já existem pastas de feature aqui? Sinal de que uma convenção de markdown local já está em uso, mesmo sem registro formal.

### 2. Propor um default inferido, não abrir um menu em branco

Com base na exploração, lidere com a recomendação e deixe o usuário aceitar em uma palavra ou trocar:

- Se o remote aponta para GitHub → proponha **GitHub Issues**
- Se não há remote configurado, ou `.scratch/` já está em uso → proponha **Markdown local**
- Caso contrário, apresente as opções sem assumir: **GitHub Issues** (via `gh`), **Markdown local**, **Linear** (via MCP), **Jira**, **Outro / customizado**

Se já existir configuração anterior (seção no arquivo de instruções ou `docs/agents/issue-tracker.md`), pergunte primeiro se é reconfiguração ou se deve manter a atual — não pule direto para a proposta de default.

### 3. Configurar de acordo com o tracker escolhido

#### GitHub Issues

- Confirme autenticação: `gh auth status`
- Confirme remote: `git remote -v`. Se não houver remote, pare e instrua a rodar o Bootstrap do `to-prd` primeiro.
- A spec vira uma issue; cada ticket vira uma **sub-issue** daquela issue, usando a relação nativa do GitHub.

> ⚠️ O `gh` **não tem comando nativo de sub-issue** (verificado na 2.88.1: nem `gh issue create` nem `gh issue edit` têm flag de parent). O vínculo é feito pela API GraphQL, com a mutation `addSubIssue`. Documente os comandos abaixo literalmente — sem eles, `to-tickets` publica issues soltas e o `implement` não acha os tickets da spec.

Comandos a documentar (com placeholders `{...}`):

```bash
# Criar spec — devolve a URL da issue no stdout
gh issue create --title "{title}" --body-file {arquivo} --label "{label}"

# Node ID da spec (necessário para vincular sub-issues)
gh issue view {spec-number} --json id -q .id

# Criar ticket — devolve a URL do ticket no stdout; leia-a e use no passo seguinte
gh issue create --title "{title}" --body-file {arquivo} --label "{label}"

# Vincular o ticket como sub-issue da spec
gh api graphql -f query='
  mutation($parent:ID!, $url:String!) {
    addSubIssue(input:{issueId:$parent, subIssueUrl:$url}) { clientMutationId }
  }' -f parent={spec-node-id} -f url={ticket-url}

# Listar os tickets de uma spec (com o corpo, de onde sai o ## Blocked by)
gh api graphql -f query='
  query($owner:String!, $repo:String!, $num:Int!) {
    repository(owner:$owner, name:$repo) {
      issue(number:$num) {
        subIssues(first:50) {
          nodes { number title state body labels(first:10){ nodes{ name } } }
        }
      }
    }
  }' -f owner={owner} -f repo={repo} -F num={spec-number}

# Ver o pai (a spec) de um ticket — sem pai, parent vem null
gh api graphql -f query='
  query($owner:String!, $repo:String!, $num:Int!) {
    repository(owner:$owner, name:$repo) {
      issue(number:$num) { parent { number title } }
    }
  }' -f owner={owner} -f repo={repo} -F num={ticket-number}

# Listar por label / ver / editar labels
gh issue list --label "{label}"
gh issue view {number}
gh issue edit {number} --add-label "{label}" --remove-label "{label}"

# Fechar (ticket aprovado)
gh issue close {number} --reason completed
```

`addSubIssue` aceita `subIssueUrl` no lugar do node ID do filho — por isso a URL que o `gh issue create` devolve serve direto, sem uma segunda consulta. O agente a lê do stdout e a passa no comando seguinte; não há variável de shell no meio.

**Bloqueio entre tickets não usa relação nativa**, mesmo o GitHub tendo uma. A fonte única é a seção `## Blocked by` do corpo de cada ticket: funciona igual em qualquer tracker, e uma fonte só não diverge. Por isso a query de tickets traz o `body`.

#### Markdown local (`.scratch/`)

- Uma pasta por feature: `.scratch/{feature-slug}/`
- **Spec** — no nível da feature: `.scratch/{feature-slug}/spec.md`. Corpo conforme o `<spec-template>` que vive dentro da skill `to-spec`.
- **Tickets** — um nível abaixo, em `issues/`: `.scratch/{feature-slug}/issues/{NN}-{slug}.md`, numerados a partir de `01` em ordem de dependência (bloqueadores primeiro). Corpo conforme o `<local-ticket-template>` que vive dentro da skill `to-tickets`.
- O nível `issues/` existe para separar tickets da spec, não para agrupar — o agrupamento já é a pasta da feature.
- Adicione `.scratch/` ao `.gitignore` se ainda não estiver. É espaço de trabalho da sessão, não documentação do projeto.
- Comandos a documentar (usando as ferramentas do agente em vez de shell):
  - Criar spec: escrever `.scratch/{feature-slug}/spec.md`
  - Criar ticket: escrever `.scratch/{feature-slug}/issues/{NN}-{slug}.md`
  - Listar: buscar `\*\*Status:\*\* {label}` em `.scratch/*/issues/*.md`
  - Ver: ler o arquivo correspondente
  - Editar label: editar o campo `**Status:**` no corpo do arquivo
  - Fechar: editar o campo `**Status:**` para `closed`
  - Vincular ticket à spec: gravar o ticket em `issues/` dentro da pasta da spec
  - Listar tickets da spec: ler `.scratch/{feature-slug}/issues/*.md`
  - Ver pai: a spec é `.scratch/{feature-slug}/spec.md`, na mesma pasta do ticket

> ⚠️ **Sem orquestração AFK neste modo.** A skill `implement` gerencia estado por labels num tracker real; de `.scratch/` ela só lê a cópia de trabalho da SPEC, nunca o `**Status:**` dos tickets. Markdown local serve para trabalho conduzido por você, consumido na própria sessão. Avise o usuário disso ao confirmar a escolha.

#### Linear

- Confirme que o MCP do Linear está conectado (liste workspaces/times disponíveis para o usuário escolher).
- Registre: workspace, team key, e os comandos/tools MCP usados para criar e listar issues, vincular um ticket como sub-issue da spec, listar os tickets de uma spec e ver o pai de um ticket.

#### Jira

- Peça: base URL, project key, e método de autenticação já configurado (MCP ou API token).
- Registre os comandos/tools equivalentes a criar, listar, ver, editar labels, fechar, vincular ticket à spec, listar os tickets de uma spec e ver o pai de um ticket.

#### Outro / customizado

- Descubra os comandos equivalentes a criar, listar, ver, editar labels, fechar, vincular ticket à spec, listar os tickets de uma spec e ver o pai de um ticket. Pergunte ao usuário só o que não der para descobrir (ex: qual sistema, onde está a credencial).

### 4. Definir o vocabulário de labels de triagem

Vocabulário padrão sugerido — apresente ao usuário para confirmação ou ajuste, não grave sem aprovação:

| Label | Significado |
|---|---|
| `ready-for-agent` | Pronta para um agente autônomo (AFK) implementar sem supervisão |
| `ready-for-human` | Bloqueada — aguardando decisão ou ação humana |
| `needs-triage` | Ainda não avaliada por um maintainer |
| `needs-spec` | Precisa de SPEC antes de implementar |
| `in-progress` | Agente no loop — implementando ou em review |
| `in-review` | Aprovada pelo revisor, aguardando merge (só SPEC ou issue sem tickets) |

`ready-for-agent` é obrigatória — é a label que `to-spec` e `to-tickets` aplicam ao publicar, e é por ela que o `implement` encontra trabalho. As demais podem ser removidas, renomeadas, ou o usuário pode adicionar outras (ex: `wontfix` para issues rejeitadas, `needs-info` para issues que precisam de mais contexto antes de triar).

**Duas propriedades deste vocabulário não são negociáveis:**

- **Os estados são exclusivos.** Um card anda entre colunas, não acumula colunas. Toda transição remove o estado anterior. Adicionar sem remover deixa a issue em `ready-for-agent` + `in-progress` + `in-review` ao mesmo tempo, e a detecção de estado do `implement` — que pergunta "algo em `in-progress`?" — fica verdadeira para sempre.
- **Não existe label de "concluída".** Issue fechada já é essa informação. Uma label `done` duplica o que o tracker já sabe e exige alguém para mantê-la — é mais um estado para desincronizar. Quem fecha depende do que a issue é: o **ticket** é fechado pelo `implement` assim que o revisor aprova, e é isso que faz o progresso de sub-issues da SPEC andar durante a execução; a **SPEC** (ou a issue sem tickets) fecha no merge, pelo `Closes #N` do corpo do PR. Por isso `in-review` só aparece em SPEC ou issue sem tickets: um ticket vai de `in-progress` direto para fechado. A review é parte do loop e não ganha label própria.

Este é o **único** vocabulário de labels do pipeline. Nenhum agente define o seu próprio — `implement` e `reviewer` leem daqui.

### 5. Criar as labels no tracker (quando aplicável)

Para Markdown local, não há nada a criar aqui — o vocabulário do passo 4 já é a única fonte de verdade, aplicado diretamente no campo `**Status:**` do corpo de cada arquivo.

Para GitHub Issues, crie (idempotente, `--force` sobrescreve se já existir) as labels confirmadas no passo 4:

```bash
gh label create "ready-for-agent" --color "0E8A16" --description "Pronta para agente autônomo implementar" --force
gh label create "ready-for-human" --color "D93F0B" --description "Bloqueada, aguardando ação humana" --force
gh label create "needs-triage" --color "FBCA04" --description "Ainda não avaliada por um maintainer" --force
gh label create "needs-spec" --color "C5DEF5" --description "Precisa de SPEC antes de implementar" --force
gh label create "in-progress" --color "1D76DB" --description "Agente no loop: implementação e review" --force
gh label create "in-review" --color "5319E7" --description "Aprovada, aguardando merge (SPEC ou issue sem tickets)" --force
```

Ajuste os comandos ao vocabulário efetivamente confirmado — não à lista padrão, se o usuário customizou. Para Linear/Jira/outro, crie os labels/estados equivalentes pelas ferramentas daquele sistema, se o sistema exigir criação prévia.

### 6. Gravar

**`docs/agents/issue-tracker.md`** — detalhe completo:

```markdown
# Issue Tracker

Tracker: [github-issues | markdown-local | linear | jira | outro]
Orquestração AFK: [sim | não — markdown-local não suporta]

## Estrutura

Spec: [onde a spec é publicada]
Tickets: [onde os tickets são publicados, e como se ligam à spec]

## Comandos

Criar spec: [comando/operação exata com placeholders]
Criar ticket: [comando/operação exata com placeholders]
Vincular ticket à spec: [comando/operação exata com placeholders]
Listar tickets da spec: [comando/operação exata com placeholders — traz estado, labels e corpo]
Ver pai: [comando/operação exata com placeholders]
Listar: [comando/operação exata com placeholders]
Ver: [comando/operação exata com placeholders]
Editar labels: [comando/operação exata com placeholders]
Fechar: [comando/operação exata com placeholders]

Bloqueio entre tickets: seção `## Blocked by` do corpo — fonte única, sem relação nativa.

## Vocabulário de Labels

| Label | Significado |
|---|---|
| ready-for-agent | ... |
| ... | ... |
```

**Arquivo de instruções** — bloco resumido:

```markdown
## Issue Tracker

Tracker: [github-issues | markdown-local | linear | jira | outro]. Ver `docs/agents/issue-tracker.md` para estrutura, comandos e vocabulário de labels.
```

Se a seção `## Issue Tracker` já existir no arquivo de instruções, **atualize o bloco in-place** — não duplique, não toque nas seções vizinhas do arquivo.

Esses dois arquivos juntos são o contrato que `to-spec`, `to-tickets` e o `implement` leem para saber onde e como publicar — mesmo espírito que `## Commands` cumpre para o `implementer`, mas com o detalhe pesado isolado fora do arquivo de instruções. Sem eles, essas skills devem parar com erro.

## Parte 2 — `## Commands`

O contrato de execução. O `implementer` lê esta seção para saber o que rodar; sem ela, para com hard stop. O `implement` também a lê, mas só no Bootstrap, para medir a baseline.

### 1. Derivar, não perguntar

Leia o manifesto do projeto e extraia os comandos reais:

| Stack | Onde olhar |
|---|---|
| Node | `scripts` do `package.json` |
| Rust | `Cargo.toml` (`cargo clippy`, `cargo build`, `cargo test`) |
| Go | `go vet`, `go build ./...`, `go test ./...` |
| Python | `pyproject.toml` — `ruff`, `mypy`, `pytest` |

Ambiguidade também é sua de resolver, não do usuário. Repo com `test`, `test:watch` e `test:e2e` tem três candidatos e só um é o gate: escolha o que roda a suite inteira uma vez e termina — nunca o modo watch, que não termina, nem o e2e, lento demais para gate por task — e informe a escolha em uma linha.

**Repo sem manifesto ainda** — o caso do repo novo, antes da primeira SPEC: não há de onde derivar. Grave a seção só com o título e a linha `Nenhum label ainda: o implementer declara os comandos na unidade que criar o manifesto.`, pule os passos 1b a 4 e informe isso na saída. Não escreva comando a partir da tabela Stack do PRD — é plano, e a seção nunca documenta comando que ainda não existe. É o mesmo tratamento do `architecture.md` num repo sem código (Parte 5): o `implementer` preenche a seção no mesmo commit em que o manifesto nasce, e a mantém quando muda um script de gate depois; o revisor cobra (CL6).

### 1b. O quinto label: `clean`

`clean` é **opcional** e apaga **artefatos de build**, nunca código-fonte. Ele existe porque o problema que resolve é genérico e a solução é específica de cada stack: um artefato deixado por um build de outra branch faz um gate ficar vermelho sem nenhum arquivo-fonte estar errado, e o agente atribui o vermelho a si mesmo. Declarando `clean`, o pipeline resolve o caso mecânico sem que nenhuma skill precise saber que Next.js existe.

Quem o usa: o `implement` roda `clean` no Bootstrap, antes de medir a baseline — é o que torna a foto honesta. E o `implementer` o roda uma vez, quando um gate fica vermelho apontando para arquivo que o diff dele não tocou, antes de escalar.

Se o manifesto não tiver um script de limpeza, **crie um** e informe em uma linha o que ele apaga — é decisão técnica, não pede aprovação:

| Stack | Proposta |
|---|---|
| Next.js | apagar `.next/` |
| Vite | apagar `dist/` e `node_modules/.vite` |
| Rust | `cargo clean` |
| Go | `go clean -cache` |
| Python | apagar `.mypy_cache/`, `.pytest_cache/`, `__pycache__/` |

> ⚠️ **Portabilidade — regra dura.** Use sempre o **runtime da stack**, nunca comandos de shell. `rm -rf` não existe no PowerShell e `rmdir /s` não existe no Bash; o pipeline roda nos dois. Em Node:
>
> ```json
> "clean": "node -e \"require('fs').rmSync('.next',{recursive:true,force:true})\""
> ```
>
> Em Python, `python -c "import shutil; shutil.rmtree(...)"`. Em Rust e Go, os comandos nativos já são portáteis.
>
> O `clean` fica no **manifesto** (o `package.json`, o `pyproject.toml`), e a linha do `## Commands` só o invoca. Escrever o comando destrutivo inline na seção coloca um `rm` num arquivo de prosa que ninguém revisa.

**Ordem obrigatória ao gravar:** o script entra no manifesto **junto ou antes** da linha no `## Commands`. A seção nunca deve documentar um comando que ainda não existe.

### 2. Medir

Rode cada comando uma vez e cronometre — inclusive o `clean`. Você precisa do número para o passo seguinte, e uma medição também prova que o comando funciona neste repo — comando escrito no arquivo de instruções que não roda é o mesmo que não ter seção.

### 3. Gravar

```markdown
## Commands

- lint: `<comando>`
- typecheck: `<comando>`
- build: `<comando>`
- test: `<comando>`
- clean: `<comando>`   ← opcional
```

Cinco labels, uma seção. **Não crie seções separadas para "CI" e "testes"** — a divisão faz o agente escolher qual ler, e escolha errada aqui produz um gate que passa sem ter rodado nada.

Se algum label não existir neste stack (projeto sem typecheck, por exemplo), omita a linha. Os agentes rodam os labels que existem, e o `clean` ausente só significa que o passo de limpeza é pulado — nada quebra.

> **`audit` não é um label desta seção.** Auditoria de dependência vive na CI, num checkout limpo. O motivo não é a ferramenta: vulnerabilidade não aparece porque alguém mexeu no manifesto, aparece porque publicaram um aviso sobre um pacote que já estava lá — e um gate que só roda quando o diff toca o manifesto nunca pega o caso comum. Quem grava `.github/workflows/ci.yml` é a Parte 7. Esta seção continua sem label `audit`, e sem uma segunda seção chamada "CI".

### 4. Reportar o custo ao usuário

Mostre os tempos medidos e explique a consequência, porque ela define o comportamento do `implementer`:

- `lint`, `typecheck` e `test` são o **gate por task** — rodam a cada task fechada
- `build` roda **uma vez**, no fim da unidade

Se a suite de `test` passar de ~60s neste repo, avise o usuário: a esse custo ela sai do gate por task e passa a rodar junto do `build`, uma vez no fim. Registre a exceção como uma linha logo abaixo da seção.

---

## Parte 3 — `## Fluxo`

A recomendação que qualquer agente dá ao usuário quando um entendimento compartilhado é atingido — depois de um grill, ou de um pedido que virou conversa.

```markdown
## Fluxo

Ao chegar num entendimento compartilhado de uma mudança, recomende uma das três saídas:

- Cabe numa sessão e não muda contrato público (schema, rota, auth, API) → `/implement` direto, sem SPEC nem issue.
- Muda contrato público, ou o registro rastreado importa → `/to-spec`.
- Mais de uma SPEC prevista → `/to-prd` antes.

Recomende, não decida — a escolha é do usuário. Em qualquer das três, branch e PR sempre.
```

Confirme o critério com o usuário antes de gravar: o limiar entre "implementa direto" e "vira SPEC" é dele, não seu.

---

## Parte 4 — Remote e `.gitignore`

**Remote:** `git remote -v`. Sem remote não há PR, e o loop de implementação encerra no bootstrap. Se não houver, crie agora (`gh repo create`) ou instrua o usuário — não siga adiante em silêncio.

**`.gitignore`:** garanta a linha `.scratch/`. A cópia de trabalho é efêmera por design; versioná-la recria a duplicação que o modelo de artefatos evita.

---

## Parte 5 — `docs/agents/architecture.md`

O mapa do código para os agentes: `to-spec`, `to-tickets`, `implementer` e revisor o leem para não redescobrir o repo a cada execução. Não é documento para o dono ler; quem o mantém é o `implementer`, e quem cobra é o revisor (CL6) — a mesma posse que ele tem sobre `## Commands`.

Se ele já existe, não reescreva. Senão, explore o código e grave, curto:

```markdown
# Arquitetura

Mapa de módulos, seams de teste e invariantes deste repo. Documento vivo: o `implementer` o atualiza quando uma unidade cria módulo ou camada, move um seam, muda uma fronteira ou um invariante.

## Onde cada coisa mora

| Área | Módulo | Nota |
|---|---|---|

## Seams de teste

[onde a lógica testável mora e onde os testes rodam; o seam preferido para código novo]

## Invariantes

[regras que nenhum lint pega e que o código respeita hoje — direção de dependência, fronteira servidor/cliente, onde input externo é validado]
```

Escreva só o que o código mostra hoje — não o que o PRD promete. Repo sem código ainda → grave só o esqueleto; o `implementer` preenche conforme os módulos nascem. Aponte para ele no arquivo de instruções, com uma linha fora das três seções do pipeline — ex: `Mapa do código (módulos, seams, invariantes): docs/agents/architecture.md`. Se já houver um ponteiro, não duplique. Informe em uma linha que o gravou.

## Parte 6 — `docs/agents/review-rules-projeto.md`

O revisor aplica critérios universais — segurança, testes, limpeza, complexidade, code smells — que valem para qualquer stack. O que é **deste** projeto mora em `docs/agents/review-rules-projeto.md`: critério de framework ou de design system, caminho extra na classificação de risco do PR, e achado do revisor que se repetiu. O `implement` e o revisor leem esse arquivo. Ele deixa de ser opcional: todo projeto configurado por esta skill o tem.

**Arquivo ausente:** grave o esqueleto abaixo. **Arquivo presente:** não reescreva critério que já está lá. Se faltar a seção `## Risco`, acrescente só ela no fim e avise. Se `## Risco` já existir, não mexa — o projeto pode ter estendido a lista.

Não pergunte "a stack tem armadilhas?". Decida você. Se houver armadilha que nenhum lint pega (ex.: fronteira servidor/cliente num framework de SSR), grave poucos critérios concretos abaixo do esqueleto, no formato dos universais: código, regra, por quê, como checar **por leitura**, severidade. Se não houver, o esqueleto basta, e você informa que não acrescentou critério de stack. Critério que exige executar algo não entra: vira label de `## Commands` ou passo da CI.

```markdown
# Critérios de review do projeto

Critérios que só fazem sentido neste repositório. O revisor lê este arquivo junto com os universais da skill `implement`. Cada critério: código, regra, por quê, como checar por leitura, severidade.

Bug que chega na `main` e achado que se repete em dois PRs entram aqui quando a correção for uma regra de review. O passo está no README do pipeline, seção "Quando um bug chega na main".

## Risco

A lista padrão do pipeline (skill `implement`, seção "Risco do PR") continua valendo e não se copia para cá. Os bullets abaixo só acrescentam caminhos. Nenhum bullet rebaixa um caminho que a lista padrão já marca como alto.

### Alto

Nenhum caminho extra.

### Baixo

Nenhum caminho extra.
```

Um bullet de extensão é um prefixo de caminho (`services/payments/`) ou um nome de arquivo (`billing.ts`). "Nenhum caminho extra." significa que a lista padrão basta; quando o projeto for estender, essa linha sai e entram os bullets.

---

## Parte 7 — CI, template de PR e rótulos de risco

O PR que o `implement` abre tem que chegar com três respostas que o dono lê sem o diff: como foi testado, qual é o risco, como desfazer. O template fixa as seções. A CI roda os gates num checkout limpo, fora da máquina de quem implementou. Os rótulos `risk:*` são o que a lista de PRs mostra.

Nada disso é vocabulário de triagem. `risk:alto`, `risk:medio` e `risk:baixo` não entram na tabela de estados da Parte 1, não são exclusivos com `in-progress` e não vão no campo `**Status:**` do markdown local. São rótulo de PR.

### 1. Não sobrescrever o que já existe

Antes de gravar, veja se o arquivo está no repo:

- `.github/pull_request_template.md`
- `.github/PULL_REQUEST_TEMPLATE.md` ou `docs/pull_request_template.md` — o GitHub aceita os três. Se qualquer um existir, o projeto já escolheu o template: não crie um segundo.
- `.github/workflows/ci.yml`

**Arquivo que já existe não é reescrito.** Avise o caminho e, em uma linha, o que falta em relação ao que esta parte gravaria (as seções do template; no workflow, o job `gates` com os labels de `## Commands` e, em Node, o job `audit`). Quem reconcilia é o dono. Setup que sobrescreve CI alheia troca workflow revisado por arquivo gerado. Arquivo ausente, crie. Diretório `.github/workflows/` ausente, crie também.

### 2. Template de PR

Grave `.github/pull_request_template.md` com estas seções, nesta ordem. O `implement` preenche as mesmas ao abrir o PR. O template é o que aparece para quem abre PR à mão.

```markdown
## O que entrou

<!-- Em linguagem de produto: o que passou a funcionar. -->

## Como foi testado

<!-- Gates de ## Commands, colados label por label. O que não rodou fica "não reportado". -->

## Risco

<!-- `risk:alto`, `risk:medio` ou `risk:baixo`, e o motivo pelos caminhos do diff. -->

## Como desfazer

<!-- Antes do merge: fechar o PR. Depois: revert do merge e rollback do deploy. Migração de banco não volta com revert do git. -->

## Achados do revisor

<!-- Cada achado com a resposta: corrigido, ou recusado com motivo. Apague a seção se não houver. -->

## Ficou de fora

<!-- Follow-ups, e o que a baseline já tinha de vermelho. Apague a seção se não houver. -->
```

### 3. Workflow de CI

Grave `.github/workflows/ci.yml`. Dispara em todo `pull_request`. Não dispare em `push` só porque é comum: o contrato é o PR.

**Job `gates`.** Um passo por label de `## Commands`, exceto `clean`. `clean` apaga artefato local; o checkout da CI já é limpo. Copie o comando da seção literalmente — `npm test` quando a seção diz `npm run test:ci` é outro gate. Cada gate é um `run:` próprio. Sem `||` e sem `continue-on-error`: gate vermelho falha o job.

Antes dos gates, o setup da stack:

| Stack | Antes dos gates |
|---|---|
| Node (`package.json`) | `actions/checkout@v7`, `actions/setup-node@v7`. Versão de Node: `.nvmrc`, `.node-version` ou `engines.node`; se nenhuma existir, `22`. Cache `npm`. `npm ci` se existir `package-lock.json`; senão `npm install`. |
| Rust (`Cargo.toml`) | `actions/checkout@v7`, `dtolnay/rust-toolchain@stable` |
| Go (`go.mod`) | `actions/checkout@v7`, `actions/setup-go@v7` |
| Python (`pyproject.toml`) | `actions/checkout@v7`, `actions/setup-python@v7`, e o install que os comandos da seção exigem |

Pine a **major** (`@v7`), nunca `@main`. Major nova dessas actions, quando sair, substitui a da tabela; não trave o workflow num patch.

Envolva só os passos dos gates com os marcadores. O `implementer` atualiza esse bloco quando `## Commands` muda, e não reescreve o resto do arquivo:

```yaml
name: ci

on:
  pull_request:

jobs:
  gates:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      # pipeline:commands
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test
      - run: npm run build
      # pipeline:commands-fim

  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm audit --audit-level=high
```

O exemplo é Node com lockfile e os quatro labels. Gere o yaml da stack real. Label ausente em `## Commands` não vira passo.

**Job `audit`, só em projeto Node** (existe `package.json`). Outra stack não ganha audit inventado — nem `cargo audit`, nem `pip-audit` — se o repo não usa essa ferramenta. O comando é `npm audit --audit-level=high`: falha em `high` e `critical`. `moderate` e `low` não falham o job. Aviso transitivo sem correção publicada deixaria todo PR vermelho, e ninguém age. É o mesmo nível que `review-rules.md` já cita. Com `package-lock.json`, instale com `npm ci`; sem lock, `npm install`, e avise que o resultado do audit não reproduz entre máquinas.

**Repo sem manifesto** (`## Commands` sem labels): grave o workflow mesmo assim. Job `gates` com checkout e, dentro dos marcadores, só o comentário de que o `implementer` preenche os passos na unidade que criar o manifesto. Sem `package.json`, sem job `audit`. O arquivo nasce aqui; o comando de gate nasce com o código — o mesmo trato do `architecture.md` num repo sem código.

Não coloque chave de API neste arquivo. Não coloque review de IA nem review de segurança de serviço pago. Gates declarados e, no Node, `npm audit`.

Não rode esta CI daqui. A Parte 2 já mediu os comandos; este workflow roda no GitHub, no PR.

### 4. Rótulos de risco

Com remote GitHub e `gh` autenticado, crie os três. `--force` é o que torna o comando idempotente: rótulo novo é criado, rótulo existente tem cor e descrição alinhadas a esta lista.

```bash
gh label create "risk:alto" --color "B60205" --description "PR toca migracao, auth, cobranca, CI ou deploy" --force
gh label create "risk:medio" --color "FBCA04" --description "PR fora das listas de alto e de baixo" --force
gh label create "risk:baixo" --color "0E8A16" --description "PR so de documentacao, texto ou estilo" --force
```

A descrição vai sem acento porque o que o dono lê no PR é a linha de `## Risco`, não esta string. O nome do rótulo é o contrato: `risk:alto`, `risk:medio`, `risk:baixo`.

Sem `gh`, ou sem remote GitHub, não invente rótulo. Avise que a linha de risco no corpo do PR continua valendo e o rótulo espera um GitHub. Markdown local não tem rótulo de PR.

### 5. O que não entra no arquivo de instruções

CI, template e rótulo de risco não ganham seção em `AGENTS.md`. Workflow é arquivo, não decisão de processo, e o arquivo de instruções não guarda o que a Parte 2 já separou de propósito. Uma seção "CI" ao lado de `## Commands` continua proibida.

---

## Saída

Ao finalizar, informe:

> ✅ Projeto configurado.
>
> **Tracker:** [tracker] — labels criadas: [lista]. Detalhe em `docs/agents/issue-tracker.md`.
> **Commands:** [lint [Ns] · typecheck [Ns] · test [Ns] · build [Ns] · clean [Ns | não declarado] | sem labels — repo sem manifesto; a unidade que criar o manifesto os declara]
> **Fluxo:** gravado no arquivo de instruções.
> **Remote:** [URL].
> **Arquitetura:** `docs/agents/architecture.md` [gerado | esqueleto — repo sem código | já existia].
> **Ponteiro:** [`CLAUDE.md` → `@AGENTS.md` criado | não necessário].
> **Critérios do projeto:** `docs/agents/review-rules-projeto.md` [criado | já existia; `## Risco` acrescentada | já existia com `## Risco`, não mexi]. Critérios de stack acrescentados: [N | nenhum].
> **Template de PR:** [`.github/pull_request_template.md` criado | já existia em [caminho], não mexi. Falta: [seções, ou "nada"]].
> **CI:** [`.github/workflows/ci.yml` criado — gates: [labels] · audit: [sim, `--audit-level=high` | não — repo sem `package.json`] | já existia, não mexi. Falta: [o que diverge, ou "nada"]].
> **Rótulos de risco:** [`risk:alto`, `risk:medio`, `risk:baixo` | não criados — sem GitHub].
>
> Próximo passo: `/to-spec` para registrar uma unidade de trabalho, ou `/implement`
> direto se a mudança for pequena. Loop AFK: [disponível via `/implement` | indisponível neste modo].
