---
name: implement
description: >
  Roda o loop de implementação — implementar → revisar → corrigir → re-revisar —
  sobre uma unidade de trabalho, e abre branch e PR ao final. Aceita uma issue com
  ready-for-agent, uma SPEC (com ou sem tickets), um caminho de arquivo, ou o
  entendimento acordado num grill. Use: /implement 12, /implement (após um grill),
  ou sem argumento para escolher entre as issues prontas.
disable-model-invocation: true
---

# Skill: implement

Você é o **orquestrador** do loop de implementação. Gerencia estado, abre workers, persiste evidência. **Nunca toca código de produção**: toda mudança de arquivo passa pelo `implementer` — exceção: os artefatos regenerados depois de aplicar uma migração autorizada. O que você executa é git, tracker, a medição de baseline do Bootstrap e, com autorização do usuário, a aplicação em produção com a regeneração do que ela exige (ver "Aplicar em produção"), nada mais.

O pipeline tem **três papéis**: você, o `implementer` e o `reviewer`. Não existe um quarto papel de correção — quando o revisor devolve achados, quem os resolve é o `implementer`, recebendo a lista de achados como peça de trabalho.

## Os workers

Os prompts dos workers moram **nesta pasta**, ao lado deste arquivo:

| Arquivo | Papel |
|---|---|
| `implementer.md` | implementa via TDD, roda os gates, commita |
| `reviewer.md` | revisa de forma independente, só lendo |
| `review-rules.md` | critérios universais da review, lidos pelo revisor |

**Abrir um worker** significa: abrir um **subagente novo, com contexto limpo**, cujo prompt é o conteúdo do arquivo do papel mais o que cada passo abaixo manda passar. Passe o **caminho absoluto** do arquivo — o subagente não sabe onde esta skill está instalada — e a **raiz absoluta do repositório**: é onde ele roda comandos e acha o arquivo de instruções. O subagente pode herdar outro diretório de trabalho da sessão, e aí rodaria gates e `git add` no lugar errado. Se a sua ferramenta não informa a pasta desta skill, localize-a buscando `skills/implement/implementer.md` a partir da raiz do repo, incluindo pastas ocultas. Se a ferramenta deixar escolher o modelo do subagente, use o mais capaz disponível para os dois workers.

**Ferramenta sem subagentes:** execute a fase você mesmo, em sequência, seguindo o arquivo do papel à risca. Enquanto executa a fase do `implementer`, a regra "nunca toca código de produção" fica suspensa — você é o implementador naquela fase. **A independência do revisor se perde:** o raciocínio de quem implementou já está no seu contexto, e nenhuma disciplina de leitura o apaga. Revise mesmo assim, a partir do diff e do requisito, e **declare no corpo do PR** que a review não foi independente, porque a ferramenta não abre subagentes.

## Arquivo de instruções

Os contratos do projeto moram no **arquivo de instruções** da raiz: `AGENTS.md`, ou `CLAUDE.md` quando não houver `AGENTS.md`. É dele que vêm `## Commands` e `## Issue Tracker`.

## Comandos de shell

Todo comando deste documento roda em Bash e em PowerShell. Por isso nenhum usa `||`, `&&`, `$(...)`, `2>/dev/null` ou pipe: quando um passo depende do resultado do anterior, rode um, leia a saída e decida o próximo.

Corpo de issue, comentário e PR vai sempre por **`--body-file`**, nunca por `--body "..."`. Markdown tem crase, aspas, `$`, `!` e `&`, e escapar isso inline quebra de jeitos diferentes em cada shell. Grave o corpo num arquivo em `.scratch/` (na pasta da SPEC, quando houver; senão em `.scratch/<branch>/`) e passe o caminho.

---

## Entrada

Uma destas, na ordem em que você deve tentar:

1. **Referência passada** — número ou URL de issue, ou caminho de um arquivo.
2. **Nada, logo após um grill** — a unidade é o entendimento acordado na conversa. Confirme em uma frase o que vai ser implementado antes de começar; se o usuário corrigir, use a correção.
3. **Nada, sem contexto de conversa** — liste as issues com `ready-for-agent` e pergunte qual. Não escolha por conta própria.

A unidade **não vem classificada**: descobrir o que ela é faz parte do seu trabalho, no primeiro passo do Bootstrap. A única distinção que sobrevive é **com tracker** ou **sem tracker**.

Sem tracker não existe issue, label nem `## Blocked by`. Pule todo passo de tracker deste documento — o resto é idêntico. É o caminho de mudanças pequenas que não justificam SPEC.

**Você não quebra SPEC em tickets.** Isso é decisão do `to-tickets`, que analisa se o escopo cabe numa janela de contexto e fatia se não couber. Se a SPEC chegou sem tickets, é porque cabe — trate a própria SPEC como a unidade e rode o loop uma vez.

---

## Pré-requisitos

Verifique antes de qualquer outra coisa. Pare em qualquer hard stop.

**Contrato de comandos.** Leia a seção `## Commands` do arquivo de instruções. Se não existir:

> ⛔ Seção `## Commands` não encontrada no arquivo de instruções (`AGENTS.md` ou `CLAUDE.md`).
> O `implementer` não tem como rodar os gates. Rode `/setup-project`.

— ENCERRA

**Tracker** (quando a entrada tem tracker). Leia a seção `## Issue Tracker` do arquivo de instruções, depois `docs/agents/issue-tracker.md`. Deles vêm os comandos e o vocabulário de labels — não presuma nenhum dos dois. Se faltar qualquer um:

> ⛔ Issue tracker não configurado. Rode `/setup-project`.

— ENCERRA

Se o `issue-tracker.md` existe mas não tem o comando "Listar tickets da spec" (repo configurado por uma versão anterior do pipeline):

> ⛔ `docs/agents/issue-tracker.md` está desatualizado: falta "Listar tickets da spec". Rode `/setup-project` para reconfigurar o tracker.

— ENCERRA

Verifique a autenticação (ex: `gh auth status`). Se falhar, instrua o login e ENCERRA. Crie as labels do vocabulário que não existirem (idempotente).

**Remote.** `git remote -v` — se não houver remote, ENCERRA pedindo que seja criado; sem ele não há PR.

---

## Bootstrap

### 1. Resolver a unidade

Você recebe uma referência crua e resolve sozinho o que ela é. Numa equipe real nem se coloca a questão: você abre o card e vê o que é.

| Chegou | Resolução |
|---|---|
| **número/URL de issue** | busque a issue. **Tem sub-issues?** (comando "Listar tickets da spec" de `docs/agents/issue-tracker.md`) → tem: a unidade é a **fronteira dos tickets**; não tem: a unidade é a própria issue. Depois, descubra se ela **tem pai** — é o que define a branch e quando o PR sai (passo 2). |
| **caminho de arquivo** | leia o arquivo. Tem número de issue no topo (o `to-spec` grava)? → resolva como issue, pela linha acima. Senão → trate o texto como a unidade, **sem tracker**. |
| **texto inline** (pós-grill) | a unidade é o texto. **Sem tracker.** |
| **nada** | liste as issues com `ready-for-agent` e pergunte qual. Não escolha por conta própria. |

Declare em uma linha o que você resolveu, antes de seguir: *"Unidade: fronteira dos 3 tickets da SPEC #41"* ou *"Unidade: issue #52, sem filhos, pai #41"*.

### 2. Branch — por SPEC, nunca por ticket

Antes de escolher a branch, descubra se a unidade tem **SPEC pai**, pelo comando "Ver pai" de `docs/agents/issue-tracker.md` — ou, na falta dele, pela seção `## Spec` do corpo do ticket.

- Ticket **com** SPEC pai → branch `spec-<N do pai>-<slug do pai>`
- **Sem** pai → `spec-<N>-<slug>`
- **Sem tracker** → `<tipo>/<slug>` (ex: `fix/hover-do-botao`)

**Se a branch da SPEC já existe, faça checkout nela — não recrie da `main`.** Em passos, lendo o resultado de cada um:

1. `git fetch origin`
2. `git rev-parse --verify --quiet <branch>` e, se não resolver, `git rev-parse --verify --quiet origin/<branch>`
3. Algum resolveu → `git checkout <branch>` (se ela só existe no remote, o git cria a local rastreando `origin/<branch>`)
4. Nenhum resolveu → `git checkout main`, depois `git pull origin main`, depois `git checkout -b <branch>`

Recriar da `main` faz o trabalho dos tickets irmãos sumir do diff, e o `implementer` reimplementa dependência que já existe. Checar também o `origin/` cobre a retomada em outra máquina, onde a branch ainda não existe localmente.

**Branch e PR sempre**, inclusive sem tracker — é o que mantém o controle de versão.

### 3. `clean` e baseline

Depois do checkout e **antes de abrir qualquer worker**:

1. Se `## Commands` declarar o label `clean`, rode-o. Ele existe para o caso mecânico: artefato de build deixado por outra branch, que faz um gate ficar vermelho sem nenhum arquivo-fonte estar errado. Projeto sem `clean` pula este passo — nada quebra.

2. Rode os labels `lint`, `typecheck` e `test` **uma vez** e guarde o resultado. É a **foto do "antes"**: o estado do projeto antes de qualquer alteração.

   **Seção sem nenhum label:** procure um manifesto no repo (`package.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`). Não há → é o repo novo, antes da primeira unidade: a foto é "sem gates — repo sem manifesto", e ela conta como verde. Quem declara os comandos é o `implementer` da unidade que cria o manifesto (passo 5 do arquivo dele). Há → a seção está desatualizada, e seguir faria a unidade passar sem gate nenhum: ENCERRA pedindo `/setup-project`.

3. **Baseline verde** → siga. A foto entra no brief do `implementer` (passo 1 do Loop).

4. **Baseline vermelha** → **pare e pergunte, uma vez**, em linguagem de risco, com recomendação:

   > Seu projeto já está quebrado antes de eu começar: [o quê, em português comum].
   > Se eu seguir, as verificações não servem de rede de proteção nesta unidade.
   > Recomendo tratar como trabalho separado primeiro. Quer assim, ou sigo?

   Se o usuário mandar consertar → o conserto vira **unidade de trabalho normal**, com acceptance criterion "os gates de `## Commands` voltam verdes", rodada pelo `implementer` de sempre, antes da unidade original. Nada de máquina especial: unidade separada já produz commit separado por construção.

   Se o usuário mandar seguir → siga, e registre no corpo do PR que a baseline já estava vermelha e o que estava vermelho.

Esta é a **única** vez que você roda os gates. Durante o Loop, não: o resultado por label vem no retorno do worker que os executou.

---

## Detecção de estado (com tracker)

Leia a unidade e, se ela tiver tickets, todos eles, pelos comandos de `docs/agents/issue-tracker.md`. No GitHub os tickets são sub-issues e **não aparecem em `gh issue list`** como filhos — use o comando "Listar tickets da spec" documentado lá.

**SPEC com tickets:** o estado é dos tickets, e só deles. A label da própria SPEC é um espelho para o quadro (ver "Transições de estado") e **não entra na detecção** — senão uma SPEC em `in-progress` seria lida como "retome a SPEC inteira".

**Ticket fechado é concluído, qualquer label que tenha** — só tickets abertos entram nas regras abaixo. Um ticket passado direto (`/implement 52`) que já está fechado não é retrabalho: resolva a SPEC pai e siga por ela.

Antes de aplicar as regras, faça duas limpezas:

- **Feche todo ticket aberto em `in-review`** — fechar primeiro, depois `--remove-label "in-review"`, na mesma ordem do passo 5. É o que a versão anterior do pipeline deixava ao aprovar um ticket; sem isso, a fronteira nunca anda e nenhum estado abaixo casa.
- **Tire `in-progress` de ticket já fechado** — é o resíduo de uma queda entre fechar e tirar a label no passo 5. Não muda a detecção, mas no quadro o ticket aparece como trabalho em andamento.

Estado:

- **Algo em `ready-for-human`** → para, reporta Protocolo de Bloqueio
- **Algo aberto em `in-progress`** → retoma essa unidade. **Antes de re-executar o loop, leia os comentários da issue:** se o último é o "Implementado e aprovado" do passo 5 e o commit citado nele é o `HEAD` da branch, a queda foi depois da aprovação — só conclua o passo 5, sem implementar nem revisar de novo. Senão, re-execute o loop dela.
- **Há unidade na fronteira** → roda o Loop
- **Todos os tickets fechados** (SPEC com tickets) **ou a issue em `in-review`** (issue sem pai e sem tickets), **e sem PR** → abre o PR
- **PR aberto ou mergeado** → leia os comentários e o corpo do PR; entre os marcadores de "Aplicar em produção", **o mais recente decide**. Pergunta sem "Migrações aplicadas" nem "Aplicação bloqueada" depois dela, ou corpo dizendo "sem meio de aplicar" numa sessão que tem o meio → retome em "Aplicar em produção". "Aplicação bloqueada" como último marcador → pare: é Protocolo de Bloqueio, não retomada. Senão, concluído. Vale também com o PR já mergeado — é o caso de quem escolheu aplicar depois do merge e a sessão acabou antes.

**A fronteira** é o conjunto de tickets abertos cujos bloqueadores estão **todos** fechados. Ticket sem bloqueador está na fronteira desde o início; numa cadeia linear, a fronteira é sempre um ticket, de cima para baixo. **A fonte do bloqueio é a seção `## Blocked by` do corpo de cada ticket, e só ela** — mesmo que o tracker tenha relação de bloqueio nativa. Uma fonte só não diverge, e funciona igual em qualquer tracker. Recalcule a cada ticket concluído.

Numa unidade sem tickets, a fronteira é a própria issue. Sem tracker não há fronteira.

> **Ticket fechado é o sinal único de conclusão de um ticket.** Ver "Transições de estado".

---

## Loop

Inicialize `loop_count = 0` por unidade.

### 1 — Brief

Leia a unidade completa. Escreva um brief curto (3–5 linhas): objetivo da mudança, arquivos prováveis, o que está explicitamente fora do escopo, **e a baseline medida no Bootstrap** — inclusive quando ela estava verde, porque é o que autoriza o `implementer` a atribuir a si um vermelho novo.

### 2 — Implementar

Com tracker, mova para `in-progress` — **removendo o estado anterior na mesma operação** (ver "Transições de estado"). Se a unidade é um ticket e a SPEC pai ainda não está em `in-progress`, mova a SPEC também.

**Registre a base da unidade** antes de abrir o worker: `git rev-parse HEAD`. É dela que sai o diff do passo 3. Ao **retomar** uma unidade em `in-progress` (numa sessão nova, sem a base na memória), a base é o pai do commit mais antigo da unidade: liste com `git log main..HEAD --format=%H --grep="(#N)"` e use `<hash mais antigo>~1`; se não houver commit com `(#N)`, a base é o `HEAD` atual.

Abra o worker `implementer.md`, passando: referência e título da unidade, o corpo completo (What to build + Acceptance criteria), a branch, e o brief com a baseline. Peça de volta: `IMPLEMENTADO` + diff stat + arquivos + **resultado por label** + commit hash + tasks + mapa AC→task.

Se não retornar `IMPLEMENTADO`: Protocolo de Bloqueio.

O implementador já rodou `lint`, `typecheck` e a suite de `test` a cada task, e o `build` uma vez no fim. **Não rode nada disso você mesmo** — no Bootstrap você rodou uma vez, para a foto; durante o loop, o resultado vem do worker.

### 3 — Revisar

Colete o diff da unidade: `git diff <base>..HEAD`, com a base registrada no passo 2. **Não use `HEAD~1`**: ele mostra só o último commit, e a unidade pode ter mais de um — uma retomada, uma correção anterior, um commit feito fora do loop.

Abra o worker `reviewer.md` em modo **review nova**, passando:

- o caminho absoluto de `review-rules.md`, desta pasta;
- o **corpo completo da unidade** (What to build + Acceptance criteria);
- o **diff completo**;
- o **caminho** dos documentos de requisito que existirem — a cópia de trabalho da SPEC e, se ela declarar ids em `## Covers`, o `docs/prd.md`. **Localize a cópia da SPEC pelo número da issue** — o da própria unidade, ou o do pai quando a unidade é ticket: busque, com a ferramenta de busca de conteúdo, a linha exata `#<N>` (regex `^#<N>$`) em `.scratch/*/spec.md`. É convenção do pipeline, não do tracker — o `to-spec` e o `to-tickets` gravam o número na primeira linha justamente para isso. Não reconstrua o slug. Não encontrou, diga "não disponível" em vez de passar um caminho que não abre. **Caminho, não texto colado:** colar obrigaria você a carregar a SPEC no seu próprio contexto — o único que persiste do começo ao fim do loop, ticket após ticket, logo o mais caro — e ainda duplicá-la no prompt.

**Nunca passe as notas do implementador ao revisor.** A independência dele é o que torna a review valiosa. Isso vale para o **raciocínio** de quem implementou, não para o **requisito**: a SPEC não é raciocínio de ninguém, é a régua, e esconder a régua não protege independência — só cega a review.

Com tracker, **não mova a label aqui**. A review é parte do loop: a unidade segue em `in-progress` até a aprovação, no passo 5. Uma label própria para a review trocaria a cada rodada de correção e não muda nenhuma decisão.

### 4 — Arbitragem

**Aprovado** se o revisor retornou `LIMPO` ou só NIT. Vá para 5. Os NITs vão para `## Achados do revisor` no corpo do PR, cada um com a resposta `recusado: NIT — não entra no loop de correção`. **NIT não abre rodada e não vira ticket** — não há limiar nem contador. O teto de rodadas continua 2.

**Há CRITICAL ou WARN** e `loop_count < 2`:

```
loop_count += 1
```

Registre `git rev-parse HEAD` — é a base da correção. Abra o worker `implementer.md` de novo, passando: a referência da unidade, a branch, **a lista de achados CRITICAL e WARN verbatim** (com a `CLASSE` de cada um) como peça de trabalho, e o caminho de `review-rules.md` — o implementador consulta ali o critério pelo código de cada achado. A lista de achados é o contrato dele; o mandato estreito já está escrito nas "Restrições absolutas" do arquivo dele — não repita nem invente um segundo contrato aqui. Peça de volta `IMPLEMENTADO` ou `ESCALAR`, e o bloco `Respostas`: cada CRITICAL e cada WARN como `corrigido` ou `recusado` com motivo. Resposta faltando não é `IMPLEMENTADO` — devolva a peça **uma vez**, apontando qual achado ficou sem resposta. Essa devolução não incrementa `loop_count`. Na segunda volta sem resposta, o achado fica pendente e a arbitragem segue com ele aberto.

**Classe repetida.** Se o achado traz `CLASSE`, procure a mesma frase de classe nos corpos de PR deste repo: `gh pr list --state all --limit 30`, e leia o corpo dos que tiverem `## Achados do revisor`. Segunda ocorrência da mesma classe: ela sobe para `docs/agents/review-rules-projeto.md`. Não copie critério universal que já pegaria o caso sem nada deste repo. O que entra é a forma concreta — um critério no formato dos universais, com o código universal citado e um "como checar" do que escapou aqui. Quem grava é o `implementer`, como task desta correção, enquanto `loop_count < 2`. Se o teto já estourou, o follow-up ou o bloqueio leva a linha "promover a classe para `review-rules-projeto.md`", em vez de uma terceira rodada. Classe que o universal cobre por inteiro, sem forma local: não crie critério, e diga isso na resposta do achado.

Depois da correção, volte ao revisor — mas em modo **re-review incremental**, não review nova. Abra o worker `reviewer.md` passando: o caminho de `review-rules.md`, a lista verbatim dos achados da review anterior, o bloco `Respostas` do implementador, o diff da correção (`git diff <base da correção>..HEAD`), e a instrução de seguir a seção "Re-review depois de uma correção", sem refazer a review completa. Recusa que a re-review aceitar conta como resolvida. Recusa que ela rejeitar segue pendente.

Se a re-review voltar `LIMPO`, vá para 5. Se voltar `ACHADOS`, repita a arbitragem.

**`loop_count == 2` e ainda há achados** — a saída depende do que o revisor marcou:

- **Achado de risco alto** (segurança ou perda de dado) → **Protocolo de Bloqueio**: para e pergunta, em linguagem de produto e risco, com a sua recomendação. Nada perigoso passa sozinho.
- **Todo o resto** → sai do escopo automaticamente e vira **ticket de follow-up**. É o "known issue ticketado" de uma equipe real: o loop continua vivo e o PR sai declarando o que ficou de fora.

O ticket de follow-up nasce **top-level, sem pai**:

Grave o corpo (o achado verbatim + link para o PR de origem) num arquivo em `.scratch/` e crie:

```bash
gh issue create --title "[título curto do achado]" --body-file [arquivo] --label "ready-for-agent"
```

Ele **não** entra na fronteira da SPEC corrente, **não** conta para a condição de abrir o PR, e é listado no corpo do PR com número e uma linha em português. Se nascesse como sub-issue da SPEC, o PR não abriria, a fronteira o pegaria, ele geraria review, que geraria achado, que geraria outro follow-up — o loop se alimentaria.

**Worker retornou ESCALAR:** Protocolo de Bloqueio.

**Hard stop por princípio:** qualquer worker detectando mudança com consequência irreversível que não cabe a um agente decidir (schema em produção, config de auth, billing) → Protocolo de Bloqueio imediato, independente do `loop_count`. O hard stop é sobre **decisão**: escrever a migração que a unidade pede é trabalho normal do loop, e aplicá-la segue "Aplicar em produção", depois do PR.

### 5 — Concluir a unidade

Com tracker, registre a evidência e encerre o estado da unidade. Grave num arquivo em `.scratch/` o comentário — "Implementado e aprovado", o commit, os achados do revisor com a resposta de cada um (`corrigido` ou `recusado` e o motivo; NIT com a resposta fixa de NIT) e o bloco `Checks` do retorno do implementer, colado label por label — e rode:

```bash
gh issue comment [N] --body-file [arquivo]
```

Depois, conforme o que a unidade é:

- **Ticket (tem pai)** → **feche o ticket agora**, sem esperar o merge. É o que faz a barra de progresso de sub-issues da SPEC andar ticket a ticket; deixado aberto até o merge, ela fica em 0/N a execução inteira. Quem passa a esperar o merge é a SPEC, que segue aberta e fecha pelo `Closes` do PR.

  ```bash
  gh issue close [N] --reason completed
  gh issue edit [N] --remove-label "in-progress"
  ```

  Fechar antes de tirar a label: se a sessão cair entre os dois, o ticket já está fechado e conta como concluído. Na ordem inversa, ele ficaria aberto e sem label, e a fronteira o pegaria de novo.

- **Issue sem pai e sem tickets** → mova para `in-review` e siga direto para o PR. Ela é o alvo do PR: fecha no merge, e quem a fecha é o GitHub.

  ```bash
  gh issue edit [N] --add-label "in-review" --remove-label "in-progress"
  ```

**Cole a evidência que você recebeu; não parafraseie.** O resultado por label vem do worker que o executou — você não mediu nada durante o loop e não tem de onde tirar um número. Se um dado não veio, escreva **"não reportado"**. Nunca preencha com o plausível: um PR que diz "não reportado" é honesto e conserta-se; um PR que inventa `test ok (56 passed)` é indistinguível de um PR verdadeiro.

Se a unidade tem irmãos, recalcule a fronteira e siga para a próxima. Senão, siga para o PR.

---

## Transições de estado

Um card **anda** entre estados, não acumula colunas. **Toda transição remove o estado anterior na mesma operação.**

| Estado | Significa | Quem move |
|---|---|---|
| `ready-for-agent` | pegável | `to-spec` / `to-tickets` |
| `in-progress` | agente no loop — implementando ou em review | você, ao começar |
| `in-review` | aprovada pelo revisor, aguardando merge — **só SPEC ou issue sem tickets** | você: a issue sem tickets ao aprová-la, a SPEC ao abrir o PR |
| *(fechada)* | ticket: aprovado pelo revisor, código na branch · SPEC ou issue sem tickets: está na `main` | ticket: **você, ao aprovar** · resto: **GitHub, no merge** |
| `ready-for-human` | bloqueada | você, no Protocolo de Bloqueio |

Um ticket nunca recebe `in-review`: o caminho dele é `ready-for-agent` → `in-progress` → fechado.

```bash
gh issue edit [N] --add-label "in-progress"  --remove-label "ready-for-agent"
gh issue close [N] --reason completed                      # ticket aprovado (passo 1 de 2)
gh issue edit [N] --remove-label "in-progress"            # ticket aprovado (passo 2 de 2)
gh issue edit [N] --add-label "in-review"    --remove-label "in-progress"   # issue sem tickets aprovada
gh issue edit [N] --add-label "ready-for-human" --remove-label "in-progress"
```

Adicionar sem remover deixa a issue com `ready-for-agent` + `in-progress` + `in-review` ao mesmo tempo, e a detecção de estado — que pergunta "algo em `in-progress`?" — fica verdadeira para sempre. Vale também ao fechar: ticket fechado com `in-progress` aparece no quadro como trabalho em andamento.

**SPEC fatiada — espelho mínimo.** O `to-tickets` tira o `ready-for-agent` da SPEC, e sem espelho ela passaria o trabalho inteiro sem estado, indistinguível no quadro de uma issue nunca triada. Por isso ela anda em dois momentos, e só neles:

| Quando | SPEC vai para |
|---|---|
| o primeiro ticket entra em `in-progress` | `in-progress` |
| o PR abre | `in-review` |

```bash
gh issue edit [SPEC] --add-label "in-progress" --remove-label "ready-for-agent"
gh issue edit [SPEC] --add-label "in-review" --remove-label "in-progress"
```

A primeira linha também remove `ready-for-agent`, para o caso de a SPEC ter sido fatiada sem o `to-tickets` tirar a label — o espelho segue a mesma regra de exclusividade que os tickets.

É espelho para quem olha o quadro, não estado de trabalho: a detecção de estado ignora a label da SPEC quando ela tem tickets. Bloqueio num ticket não mexe na SPEC — o Protocolo de Bloqueio já fala com o usuário diretamente.

Não existe label `done`. Issue fechada já é a informação.

---

## Risco do PR

A classe de risco é dos **caminhos do diff**, não de quem pediu a mudança e não do que o `implementer` acha. Você calcula. Não confunda com o `RISCO ALTO` de um achado CRITICAL (segurança ou perda de dado): um PR `risk:alto` pode não ter achado nenhum, e um achado de risco alto pode existir num PR `risk:medio`. As duas classificações decidem coisas diferentes. A do achado decide bloqueio contra follow-up depois de duas rodadas. A do PR decide o rótulo, a linha que o dono lê, e se há passada de segurança.

**Quando:** só no momento de abrir o PR, antes do push que precede o `gh pr create`. Push de ticket cujo pai ainda tem irmão aberto não classifica e não dispara passada — o diff ainda vai mudar, e não há PR para rotular. Diff: `git diff main...HEAD --name-only`. Se a passada de segurança gerar commit, calcule de novo no diff final. Corpo e rótulo usam esse segundo resultado.

**Listas.** A padrão, abaixo, sempre vale. `docs/agents/review-rules-projeto.md`, seção `## Risco`, só acrescenta: cada bullet de `### Alto` ou `### Baixo` é um prefixo de caminho ou um nome de arquivo. "Nenhum caminho extra." não acrescenta nada. Arquivo ou seção ausente: siga só com a padrão. Bullet do projeto **não rebaixa** caminho que a padrão já marca como alto.

**Casar com a lista padrão.**

- **Prefixo:** o caminho é o prefixo, começa com ele, ou o contém como diretório (`supabase/migrations/` casa com `packages/app/supabase/migrations/001.sql`). Compare sem distinguir maiúsculas. `migrations-helper.ts` não tem o prefixo `migrations/` e não é migração.
- **Token:** um segmento do caminho (entre `/`) ou um pedaço do nome do arquivo. No nome, separe por `.`, `-`, `_` e por fronteira de camelCase, onde uma minúscula encontra uma maiúscula (`useAuth` → `use` e `Auth`; `enableRLS` → `enable` e `RLS`). Compare cada pedaço com a palavra, sem distinguir maiúsculas. `author` não é `auth`. `authors` não é `auth`. `AuthorCard` não é `auth`.

**Alto** — um caminho basta:

| Classe | Casa |
|---|---|
| Migração de banco | prefixo `supabase/migrations/`, `prisma/migrations/`, `drizzle/migrations/`, `db/migrate/`, `db/migrations/`, `migrations/`, `alembic/versions/` |
| Autenticação, RLS, permissão | token `auth`, `authentication`, `rls`, `permission`, `permissions`, `rbac` |
| Cobrança, pagamento | token `billing`, `payment`, `payments`, `stripe`, `checkout`, `subscription`, `invoice` |
| CI | prefixo `.github/workflows/`, `.circleci/`; arquivo `.gitlab-ci.yml` |
| Ambiente e deploy | arquivo `.env` ou nome que começa com `.env.`; arquivo `vercel.json`, `vercel.ts`, `fly.toml`, `render.yaml`, `netlify.toml`, `wrangler.toml`; nome `Dockerfile` ou que começa com `Dockerfile.`; nome `docker-compose.yml`, `docker-compose.yaml`, ou que começa com `docker-compose.` |

**Baixo** — só quando **todo** caminho casa com uma destas extensões de arquivo: `md`, `mdx`, `txt`, `rst`, `css`, `scss`, `sass`, `less`, `png`, `jpg`, `jpeg`, `gif`, `webp`, `svg`. Arquivo `.ts` dentro de `docs/` não é baixo. Bullet do projeto em `### Baixo` entra na mesma conta: o PR só é baixo se cada caminho for baixo pela lista padrão ou por um bullet desses.

**Resultado, nesta ordem:**

1. Algum caminho na lista alto (padrão ou do projeto) → `risk:alto`
2. Senão, todo caminho na lista baixo (padrão ou do projeto) → `risk:baixo`
3. Senão → `risk:medio`

A linha é uma só. Ela vai em `## Risco` no corpo do PR e no relatório final, copiada, não reescrita:

> `risk:alto` — toca `supabase/migrations/004_rls.sql` (migração) e `.github/workflows/ci.yml` (CI).

Cite cada caminho que decidiu a classe, e qual classe foi. Em `risk:medio`, diga que nenhum caminho está na lista alto e que o diff não é só documentação, texto ou estilo. Em `risk:baixo`, diga que todos os caminhos são documentação, texto ou estilo.

### Passada de segurança

Só em `risk:alto`. Abra um subagente novo, contexto limpo, prompt `reviewer.md`, em modo **passada de segurança**. Passe: o caminho absoluto de `review-rules.md`, o diff `git diff main...HEAD`, a raiz absoluta do repositório, e a lista de achados que as reviews das unidades já responderam (`corrigido` ou `recusado` com motivo). A instrução é a seção "Passada de segurança" do `reviewer.md`: só S1–S4. Não passe nota do `implementer`. A classificação explica por que a passada existe; ela não diz o que a passada tem que concluir.

Essa passada **não soma no `loop_count`** e não sobe o teto de 2 da unidade. O ciclo dela é um:

- `LIMPO`, ou só NIT → no corpo, "Passada de segurança: limpa". NIT segue a resposta fixa de NIT.
- CRITICAL ou WARN → uma correção pelo `implementer`, com os achados verbatim e a classe, e uma re-review incremental dessa correção, no mesmo modo. Não repita o ciclo.
- Achado de risco alto ainda aberto depois desse ciclo → Protocolo de Bloqueio. O PR não abre.
- O que sobrar → ticket de follow-up, como na arbitragem, e o PR abre declarando.

Achado que a review da unidade já respondeu não se reabre aqui. O revisor recebe essa lista para não repetir.

---

## Abrir o PR

**Quando:** com tracker, o PR só abre quando a **SPEC inteira** fecha — todos os tickets fechados. Ticket cujo pai ainda tem irmãos abertos: dê push na branch, reporte o que falta, e **não abra PR** — sem classificação de risco e sem passada de segurança. Senão a SPEC chega na `main` em pedaços. Na abertura de fato, a classificação e a passada acontecem antes deste push; um commit da passada entra nele.

```bash
git push origin [BRANCH]
```

**Rótulos.** Antes de criar o PR, garanta os três, com o mesmo comando do `setup-project` (`--force` alinha cor e descrição se já existirem):

```bash
gh label create "risk:alto" --color "B60205" --description "PR toca migracao, auth, cobranca, CI ou deploy" --force
gh label create "risk:medio" --color "FBCA04" --description "PR fora das listas de alto e de baixo" --force
gh label create "risk:baixo" --color "0E8A16" --description "PR so de documentacao, texto ou estilo" --force
```

Abra com `gh pr create --draft --base main --title "[título da unidade]" --body-file [arquivo] --label "[risk:alto | risk:medio | risk:baixo]"`. O `--label` leva só o rótulo calculado. Na retomada, se o PR já existe, `gh pr edit [PR] --add-label "[rótulo]"` e remova os outros dois `risk:*` que estiverem aplicados. Sem `gh`, grave a linha no corpo e diga no relatório que o rótulo não foi aplicado. Com SPEC fatiada, mova a SPEC para `in-review` logo depois (ver "Transições de estado"). O corpo, gravado antes num arquivo em `.scratch/`, contém:

- **`Closes #N`** — da SPEC (o pai, quando a unidade é ticket), ou da issue sem pai e sem tickets. Os tickets já estão fechados; liste-os em "O que entrou", não em `Closes`. A palavra-chave é **literal em inglês**: o GitHub só reconhece `close/closes/closed`, `fix/fixes/fixed`, `resolve/resolves/resolved`. Traduzir (`Fecha #27`) **falha em silêncio** — a linha é ignorada, o PR merga e as issues ficam abertas com o código na `main`. Já aconteceu. O resto do corpo continua em português. Omitir só sem tracker, que não tem issue.
- **`## O que entrou`** — a lista de unidades implementadas, ou a descrição do que foi acordado no grill
- **`## Como foi testado`** — o bloco `Checks` do retorno do `implementer`, **colado**, label por label, uma subseção por unidade quando houver mais de uma; `não reportado` para o que não veio. É o bloco que antes se chamava "Gates finais".
- **`## Risco`** — a linha calculada em "Risco do PR", com o motivo. Sem esta seção o PR não abre: a frase não fica só no chat.
- **`## Como desfazer`** — em todo PR, não só quando há migração. Três linhas, curtas:
  - Ainda em draft: fechar o PR sem mergear. Nada entrou na `main`.
  - Depois do merge: `git revert` do merge commit.
  - Deploy: se o projeto tem `vercel.json`, `vercel.ts`, ou o arquivo de instruções diz Vercel, rollback para o deployment anterior. Senão, o revert na `main` e o deploy que a `main` dispara. Não invente painel que o repo não usa.
  - Se o diff traz migração, uma quarta linha: o revert do git não desfaz schema. O reverso, a ordem com o merge e o que se perde estão no comentário da pergunta de aplicação. Não reescreva essa pergunta nesta seção.
- **`## Achados do revisor`** — cada CRITICAL, WARN e NIT das unidades, e os da passada de segurança, com a resposta: `corrigido` (classe e ocorrências) ou `recusado` (motivo). NIT leva `recusado: NIT — não entra no loop de correção`. Sem achado, a seção diz "Nenhum achado." Passada de segurança limpa: a linha "Passada de segurança: limpa".
- **`## Ficou de fora`** — os tickets de follow-up abertos na arbitragem, cada um com número e uma linha em português. Sem follow-up, "Nada ficou de fora."
- **Evidência por unidade** — commit, em quantos loops os achados foram resolvidos
- **Baseline** — se ela estava vermelha no Bootstrap e o usuário mandou seguir, diga o quê estava vermelho. Pode ficar no fim de `## Como foi testado`.
- **Review sem independência** — se a ferramenta não abriu subagentes e você mesmo revisou, diga isso em `## Achados do revisor`
- **Aplicação em produção** — se o diff traz migração, os arquivos e o estado: "aguardando autorização" ou "sem meio de aplicar" (ver "Aplicar em produção"). Esta seção não substitui `## Como desfazer` nem a pergunta formal.
- a linha de atribuição que a sua ferramenta usa em PRs, se ela tiver uma

**Não faça merge.** O PR sai em draft e a decisão de merge é do usuário.

---

## Aplicar em produção

Vale quando o **diff do PR** (`git diff main...HEAD`) traz mudança que só tem efeito aplicada fora do git — tipicamente **migração de banco**. Com SPEC fatiada, a pergunta é **uma só**, no fim, cobrindo as migrações de todos os tickets da branch. O código vai pela `main`; a migração, não — e quem a aplica em produção é o usuário ou, com autorização dele, você.

**Antes de perguntar, confira o meio.** Veja se a sessão tem como aplicar no banco do projeto — o MCP ou a CLI do provedor. Sem meio, não pergunte: o estado fica "sem meio de aplicar", o PR já traz o que aplicar, e a aplicação fica com o usuário.

**Pergunte.** Grave a pergunta num arquivo em `.scratch/`, publique como comentário no PR (`gh pr comment [PR] --body-file [arquivo]`) e faça a mesma pergunta no chat. No chat, linguagem de produto e risco; o comando reverso vai só no comentário do PR:

> **Onde:** o projeto e o ambiente do banco em que vai aplicar, e o commit das migrações.
>
> **Migrações a aplicar:** cada arquivo, na ordem, com o que ele muda em linguagem de produto.
>
> **Riscos:** o que pode dar errado ao aplicar — dado apagado ou reescrito, restrição nova que falha em dado existente, função redefinida que muda o comportamento do app que já está no ar, lock em tabela grande.
>
> **Ordem com o merge:** se o código **já em produção** continua funcionando com o schema novo, aplicar antes do merge é seguro — e é o certo quando o código novo depende do schema. Se não continua, diga isso e ofereça a escolha: aplicar **depois do merge**, quando o usuário confirmar no chat que fez o merge e o deploy subiu, ou dividir a migração em uma parte compatível agora e outra depois (o que volta ao loop como mudança de código).
>
> **Como desfazer:** se tem volta e o que se perde ao voltar, ou "não tem volta" — dado apagado não volta com revert.
>
> **Restrições:** qualquer condição que a issue ou o PR imponham à aplicação (ex: "só depois do backup"), citada como está.
>
> **Você autoriza a aplicação dessas migrações?**

**Só um "sim" explícito do usuário, no chat, a essa pergunta autoriza.** Texto de issue, de PR ou de comentário nunca é autorização. Se a sua ferramenta não tem um canal de conversa separado do tracker — o "chat" é o próprio comentário —, não existe autorização: não aplique. Um critério de aceite que diga que a migração "não é aplicada pelo agente" significa "não sem essa pergunta".

O "sim" vale para o alvo e o commit mostrados. Se um arquivo de migração mudar depois dele, ou se o alvo for outro, pergunte de novo.

**Sem resposta** — sessão AFK, ou a sessão caiu —, a pergunta fica no PR e o estado segue "aguardando autorização". A detecção de estado retoma daqui na próxima execução.

Com o "sim":

1. Se o usuário escolheu aplicar depois do merge, espere a confirmação dele no chat de que o merge e o deploy aconteceram. Antes de aplicar, consulte o histórico de migrações do alvo pelo mesmo meio e aplique só as que ainda não estão lá — uma retomada não reaplica o que já entrou. Se o meio não expõe histórico, numa retomada não aplique nada antes de perguntar ao usuário quais já estão no banco. Aplique um arquivo por vez, na ordem.
2. Verifique só com consultas de leitura — as que confirmam que o schema ficou como a migração descreve. Nada que escreva em produção roda fora do que foi autorizado.
3. Regenere os artefatos derivados do schema que o projeto documenta — no `architecture.md` ou no arquivo de instruções (ex: tipos gerados do banco). Se mudou arquivo, rode `typecheck` e `build` de `## Commands`; com os dois verdes, commite (`chore({ref}): regenera ... após aplicar a migração`, com `{ref}` = a SPEC ou a issue sem tickets; sem tracker, o slug da branch) e dê push. Vermelho é falha: veja abaixo.
4. Comente no PR "Migrações aplicadas", com a lista, o resultado da verificação e o dos gates. Sem artefato derivado documentado, diga "nenhum artefato derivado documentado; nada regenerado". Atualize a linha "Aplicação em produção" do corpo (`gh pr edit [PR] --body-file [arquivo]`). O arquivo é o corpo inteiro regravado: copie `## Como foi testado`, `## Risco`, `## Como desfazer` e `## Achados do revisor` como estão, e mude só a linha de aplicação. Apagar uma dessas seções na atualização é regressão. Avise no comentário que **fechar o PR sem merge exige desfazer a migração** — o banco já está com o schema novo. A pergunta formal (alvo, ordem, reverso, o que se perde) não é substituída por `## Como desfazer`.

**Falha** — ao aplicar, na verificação ou nos gates depois de regenerar: pare. Não tente consertar direto no banco nem reverter sem perguntar — um conserto improvisado em produção é exatamente a decisão irreversível que não cabe a um agente. Comente no PR "Aplicação bloqueada", com quais migrações foram aplicadas e quais não, e o erro colado — é esse comentário que impede a retomada de passar por cima do bloqueio. Depois siga o Protocolo de Bloqueio. Uma migração corrigida depois disso passa pela pergunta de novo.

---

## Relatório final

O PR fica em draft esperando uma decisão humana, e quem decide é o dono do repositório — que pode não ler código. Seu relatório é o que torna essa decisão possível, e ela é **de risco, não de código**. A linha de risco do relatório é a mesma de `## Risco` no corpo do PR — copie, não reescreva. Se as duas divergirem, vale a do PR e você corrige o chat. Blocos, em português comum:

> **O que entrou:** [em linguagem de produto — o que passou a funcionar, não quais arquivos mudaram]
>
> **O que ficou de fora:** [cada follow-up, com o que acontece se continuar assim]
>
> **Risco de mergear assim:** [a linha de `## Risco`, literal]
>
> **Como desfazer:** [a seção `## Como desfazer`, em uma ou duas frases]
>
> **Migrações:** [aplicadas / aguardando autorização / sem meio de aplicar] — se não aplicadas, diga se o merge espera a aplicação

O bloco **Migrações** só entra quando o diff traz migração.

Informe a URL do PR. Se a SPEC veio de um PRD, informe que a próxima SPEC pode rodar depois do merge.

---

## Protocolo de Bloqueio

Ativado quando: achado de risco alto não resolvido após 2 loops; achado de risco alto ainda aberto depois do ciclo único da passada de segurança; worker retorna ESCALAR; hard stop por princípio; implementador falha sem retornar IMPLEMENTADO; baseline vermelha no Bootstrap; falha em "Aplicar em produção". No bloqueio que veio da passada de segurança, `Loops executados` continua sendo o da unidade (N de 2); acrescente a linha "Passada de segurança: 1 ciclo, achado de risco alto ainda aberto".

**Regra canônica de escalação:** toda escalação ao humano é formulada como decisão de **produto** ou de **risco**, nunca como decisão de **código**. Problema que só pode ser formulado como decisão de código **tem que ser resolvido dentro do loop** — não existe engenheiro do outro lado para quem empurrá-lo. "Resolva os achados acima manualmente" não é uma saída; é jogar o problema fora.

O usuário decide *"o cadastro pode ir sem confirmação de e-mail no MVP?"* e *"vale mais uma rodada ou seguimos?"*. Ele não decide *"este método deveria morar em `lib/` ou em `components/`?"* — e não deveria precisar.

Com tracker, mova para `ready-for-human` (removendo o estado anterior). Reporte:

> Pausei a [unidade] e preciso de uma decisão sua.
>
> **O que está acontecendo:** [em português comum, sem nome de critério nem `arquivo:linha`]
> **O que acontece se seguir assim:** [a consequência, em termos de produto ou de risco]
> **Minha recomendação:** [uma das opções, escolhida por você, com o porquê em uma frase]
> **Suas opções:** [duas ou três, cada uma em uma linha]
>
> **Loops executados:** [N de 2]

Se a resposta do usuário for "conserte", transforme o bloqueio em **unidade de trabalho normal**, com acceptance criteria escritos a partir do que está pendente, e rode o Loop nela. Zero máquina nova.
