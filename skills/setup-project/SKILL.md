---
name: setup-project
description: >
  Configura um repositório para o pipeline: issue tracker e vocabulário de labels,
  a seção `## Commands` (lint/typecheck/test/build/clean) que o implementer lê como
  gate, a seção `## Fluxo` com as saídas recomendadas depois de um grill, e o remote.
  Rode uma vez por repositório, antes do primeiro uso de to-spec, to-tickets ou
  implement. Use quando o usuário disser "setup do projeto", "setup-project",
  "configura o tracker", "prepara o repo para o pipeline" ou variações.
disable-model-invocation: true
---

# Skill: setup-project

Você configura, uma vez por repositório, os contratos que o resto do pipeline lê em runtime. Sem eles as outras skills param com erro — de propósito, para falhar alto em vez de adivinhar.

O output são dois arquivos:

- **arquivo de instruções** — três seções curtas: `## Issue Tracker`, `## Commands`, `## Fluxo`
- **`docs/agents/issue-tracker.md`** — o detalhe pesado (comandos do tracker, tabela de labels)

**Qual é o arquivo de instruções:** `AGENTS.md` na raiz — o formato que a maioria das ferramentas de agente lê. Se o repo já tem `CLAUDE.md` e não tem `AGENTS.md`, use o `CLAUDE.md` existente em vez de criar um segundo arquivo. Se a ferramenta do usuário não carrega `AGENTS.md` sozinha, avise: o arquivo dela precisa apontar para o `AGENTS.md`, senão a seção `## Fluxo` não entra no contexto.

O arquivo de instruções entra no contexto de toda conversa — mantê-lo enxuto importa. O detalhe fica isolado num arquivo que só quem precisa vai ler.

Cinco partes, nesta ordem — a última opcional. Confirme com o usuário ao fim de cada uma antes de gravar.

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

**Por que a regra existe:** o arquivo de instruções misturava decisão estável (stack, tracker, comandos) com fato que muda a cada PR ("este repositório ainda não tem código", "o produto é uma landing de guias grátis"). O segundo tipo apodrece sozinho, e o arquivo não tem dono que o mantenha. Isso já produziu um arquivo de instruções que descrevia um produto que não existe mais, contradizendo o PRD real — enquanto o próprio arquivo enunciava a regra certa ("update it only when a decision actually changes") e a violava na seção seguinte.

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
gh issue create --title "{title}" --body "{body}" --label "{label}"

# Node ID da spec (necessário para vincular sub-issues)
gh issue view {spec-number} --json id -q .id

# Criar ticket e vinculá-lo como sub-issue da spec
TICKET_URL=$(gh issue create --title "{title}" --body "{body}" --label "{label}")
gh api graphql -f query='
  mutation($parent:ID!, $url:String!) {
    addSubIssue(input:{issueId:$parent, subIssueUrl:$url}) { clientMutationId }
  }' -f parent="{spec-node-id}" -f url="$TICKET_URL"

# Listar os tickets de uma spec
gh api graphql -f query='
  query($owner:String!, $repo:String!, $num:Int!) {
    repository(owner:$owner, name:$repo) {
      issue(number:$num) {
        subIssues(first:50) {
          nodes { number title state labels(first:10){ nodes{ name } } }
        }
      }
    }
  }' -f owner={owner} -f repo={repo} -F num={spec-number}

# Listar por label / ver / editar labels
gh issue list --label "{label}"
gh issue view {number}
gh issue edit {number} --add-label "{label}" --remove-label "{label}"
```

`addSubIssue` aceita `subIssueUrl` no lugar do node ID do filho — por isso a URL que o `gh issue create` devolve serve direto, sem uma segunda consulta.

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

> ⚠️ **Sem orquestração AFK neste modo.** A skill `implement` gerencia estado por labels num tracker real e não lê `.scratch/`. Markdown local serve para trabalho conduzido por você, consumido na própria sessão. Avise o usuário disso ao confirmar a escolha.

#### Linear

- Confirme que o MCP do Linear está conectado (liste workspaces/times disponíveis para o usuário escolher).
- Registre: workspace, team key, e os comandos/tools MCP usados para criar e listar issues, e como vincular um ticket como sub-issue da spec.

#### Jira

- Peça: base URL, project key, e método de autenticação já configurado (MCP ou API token).
- Registre os comandos/tools equivalentes a criar, listar, ver e editar labels, e a relação pai-filho entre spec e tickets.

#### Outro / customizado

- Pergunte ao usuário os comandos equivalentes a criar, listar, ver, editar labels e vincular sub-issues neste sistema.

### 4. Definir o vocabulário de labels de triagem

Vocabulário padrão sugerido — apresente ao usuário para confirmação ou ajuste, não grave sem aprovação:

| Label | Significado |
|---|---|
| `ready-for-agent` | Pronta para um agente autônomo (AFK) implementar sem supervisão |
| `ready-for-human` | Bloqueada — aguardando decisão ou ação humana |
| `needs-triage` | Ainda não avaliada por um maintainer |
| `in-progress` | Agente trabalhando |
| `in-review` | Código na branch, aprovado pelo revisor, aguardando merge |

`ready-for-agent` é obrigatória — é a label que `to-spec` e `to-tickets` aplicam ao publicar, e é por ela que o `implement` encontra trabalho. As demais podem ser removidas, renomeadas, ou o usuário pode adicionar outras (ex: `wontfix` para issues rejeitadas, `needs-info` para issues que precisam de mais contexto antes de triar).

**Duas propriedades deste vocabulário não são negociáveis:**

- **Os estados são exclusivos.** Um card anda entre colunas, não acumula colunas. Toda transição remove o estado anterior. Adicionar sem remover deixa a issue em `ready-for-agent` + `in-progress` + `in-review` ao mesmo tempo, e a detecção de estado do `implement` — que pergunta "algo em `in-progress`?" — fica verdadeira para sempre.
- **Não existe label de "concluída".** Issue fechada já é essa informação, e quem a fecha é o próprio tracker, no merge, pelo `Closes #N` do corpo do PR. Uma label `done` duplica o que o GitHub já sabe e exige alguém para mantê-la — é mais um estado para desincronizar. Por isso `in-review` foi redefinida: de "PR aberto" para "código na branch, aguardando merge". É o que de fato acontece, porque o ticket termina antes de o PR existir — o PR é por SPEC, não por ticket. A definição antiga descrevia um estado que o pipeline nunca produz.

Este é o **único** vocabulário de labels do pipeline. Nenhum agente define o seu próprio — `implement` e `reviewer` leem daqui.

### 5. Criar as labels no tracker (quando aplicável)

Para Markdown local, não há nada a criar aqui — o vocabulário do passo 4 já é a única fonte de verdade, aplicado diretamente no campo `**Status:**` do corpo de cada arquivo.

Para GitHub Issues, crie (idempotente, `--force` sobrescreve se já existir) as labels confirmadas no passo 4:

```bash
gh label create "ready-for-agent" --color "0E8A16" --description "Pronta para agente autônomo implementar" --force
gh label create "ready-for-human" --color "D93F0B" --description "Bloqueada, aguardando ação humana" --force
gh label create "needs-triage" --color "FBCA04" --description "Ainda não avaliada por um maintainer" --force
gh label create "in-progress" --color "1D76DB" --description "Em desenvolvimento" --force
gh label create "in-review" --color "5319E7" --description "Código na branch, aguardando merge" --force
```

Ajuste os comandos ao vocabulário efetivamente confirmado — não à lista padrão, se o usuário customizou. Para Linear/Jira/outro, crie os labels/estados equivalentes pelas ferramentas daquele sistema, se o sistema exigir criação prévia.

### 6. Confirmar antes de gravar

Mostre ao usuário o rascunho dos dois arquivos (bloco do arquivo de instruções + conteúdo completo de `docs/agents/issue-tracker.md`). Deixe editar antes de gravar.

### 7. Gravar

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
Listar: [comando/operação exata com placeholders]
Ver: [comando/operação exata com placeholders]
Editar labels: [comando/operação exata com placeholders]

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

Pergunte só o que ficar ambíguo. Repo com `test`, `test:watch` e `test:e2e` tem três candidatos e só um é o gate — aí sim pergunte qual.

### 1b. O quinto label: `clean`

`clean` é **opcional** e apaga **artefatos de build**, nunca código-fonte. Ele existe porque o problema que resolve é genérico e a solução é específica de cada stack: um artefato deixado por um build de outra branch faz um gate ficar vermelho sem nenhum arquivo-fonte estar errado, e o agente atribui o vermelho a si mesmo. Declarando `clean`, o pipeline resolve o caso mecânico sem que nenhuma skill precise saber que Next.js existe.

Quem o usa: o `implement` roda `clean` no Bootstrap, antes de medir a baseline — é o que torna a foto honesta. E o `implementer` o roda uma vez, quando um gate fica vermelho apontando para arquivo que o diff dele não tocou, antes de escalar.

Se o manifesto não tiver um script de limpeza, **crie um** e proponha ao usuário:

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

> **`audit` não é um label desta seção.** Auditoria de dependência vive na CI, num checkout limpo. O motivo não é a ferramenta: vulnerabilidade não aparece porque alguém mexeu no manifesto, aparece porque publicaram um aviso sobre um pacote que já estava lá — e um gate que só roda quando o diff toca o manifesto nunca pega o caso comum.

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

## Parte 5 — Critérios de review do projeto (opcional)

O revisor aplica critérios universais — segurança, testes, limpeza, complexidade, code smells — que valem para qualquer stack. Critério que só faz sentido **neste** projeto (regras do framework, tokens do design system, convenções da casa) mora em `docs/agents/review-rules.md`, no formato dos universais: código, regra, por quê, como checar **por leitura**, severidade. O revisor o lê quando ele existe.

Não crie o arquivo por padrão. Pergunte se a stack tem armadilhas que nenhum lint pega — se sim, proponha poucos critérios concretos; se não, pule. Um critério que exige executar algo não entra: vira label de `## Commands` ou passo da CI.

---

## Saída

Ao finalizar, informe:

> ✅ Projeto configurado.
>
> **Tracker:** [tracker] — labels criadas: [lista]. Detalhe em `docs/agents/issue-tracker.md`.
> **Commands:** lint [Ns] · typecheck [Ns] · test [Ns] · build [Ns] · clean [Ns | não declarado]
> **Fluxo:** gravado no arquivo de instruções.
> **Remote:** [URL].
>
> Próximo passo: `/to-spec` para registrar uma unidade de trabalho, ou `/implement`
> direto se a mudança for pequena. Loop AFK: [disponível via `/implement` | indisponível neste modo].
