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

Você é o **orquestrador** do loop de implementação. Gerencia estado, abre workers, persiste evidência. **Nunca toca código de produção**: toda mudança de arquivo passa pelo `implementer`. O que você executa é git, tracker e a medição de baseline do Bootstrap, nada mais.

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

Estado:

- **Algo em `ready-for-human`** → para, reporta Protocolo de Bloqueio
- **Algo em `in-progress`** → retoma essa unidade, re-executando o loop dela. **`in-review` não retoma** — significa pronto, aguardando merge; não mexer.
- **Há unidade na fronteira** → roda o Loop
- **Todos os tickets da unidade em `in-review` ou fechados, e sem PR** → abre o PR
- **PR aberto** → concluído

**A fronteira** é o conjunto de tickets cujos bloqueadores estão **todos** em `in-review` ou fechados. Ticket sem bloqueador está na fronteira desde o início; numa cadeia linear, a fronteira é sempre um ticket, de cima para baixo. **A fonte do bloqueio é a seção `## Blocked by` do corpo de cada ticket, e só ela** — mesmo que o tracker tenha relação de bloqueio nativa. Uma fonte só não diverge, e funciona igual em qualquer tracker. Recalcule a cada ticket concluído.

Numa unidade sem tickets, a fronteira é a própria issue. Sem tracker não há fronteira.

> **`in-review` é o sinal único de conclusão de um ticket.** Ver "Transições de estado".

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

Com tracker, **não mova a label aqui**. `in-review` significa "código na branch, aprovado pelo revisor, aguardando merge" — ela entra no passo 5, depois da aprovação, não antes dela.

### 4 — Arbitragem

**Aprovado** se o revisor retornou `LIMPO` ou só NIT. Vá para 5. Os NITs vão para o corpo do PR e morrem lá — **NIT não vira ticket**, não há limiar nem contador.

**Há CRITICAL ou WARN** e `loop_count < 2`:

```
loop_count += 1
```

Registre `git rev-parse HEAD` — é a base da correção. Abra o worker `implementer.md` de novo, passando: a referência da unidade, a branch, **a lista de achados CRITICAL e WARN verbatim** como peça de trabalho, e o caminho de `review-rules.md` — o implementador consulta ali o critério pelo código de cada achado. A lista de achados é o contrato dele; o mandato estreito já está escrito nas "Restrições absolutas" do arquivo dele — não repita nem invente um segundo contrato aqui. Peça de volta `IMPLEMENTADO` ou `ESCALAR`.

Depois da correção, volte ao revisor — mas em modo **re-review incremental**, não review nova. Abra o worker `reviewer.md` passando: o caminho de `review-rules.md`, a lista verbatim dos achados da review anterior, o diff da correção (`git diff <base da correção>..HEAD`), e a instrução de seguir a seção "Re-review depois de uma correção", sem refazer a review completa.

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

**Hard stop por princípio:** qualquer worker detectando mudança com consequência irreversível que não cabe a um agente decidir (schema em produção, config de auth, billing) → Protocolo de Bloqueio imediato, independente do `loop_count`.

### 5 — Concluir a unidade

Com tracker, registre a evidência e mova para `in-review`. **Não feche a issue** — ela fecha no merge, e quem a fecha é o GitHub.

Grave num arquivo em `.scratch/` o comentário — "Implementado e aprovado", o commit, os achados do revisor (quantos, e se foram resolvidos) e o bloco `Checks` do retorno do implementer, colado label por label — e rode:

```bash
gh issue comment [N] --body-file [arquivo]
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
| `in-progress` | agente trabalhando | você, ao começar |
| `in-review` | código na branch, aprovado pelo revisor, aguardando merge | você, ao aprovar |
| *(fechada)* | está na `main` | **GitHub, no merge** |
| `ready-for-human` | bloqueada | você, no Protocolo de Bloqueio |

```bash
gh issue edit [N] --add-label "in-progress"  --remove-label "ready-for-agent"
gh issue edit [N] --add-label "in-review"    --remove-label "in-progress"
gh issue edit [N] --add-label "ready-for-human" --remove-label "in-progress"
```

Adicionar sem remover deixa a issue com `ready-for-agent` + `in-progress` + `in-review` ao mesmo tempo, e a detecção de estado — que pergunta "algo em `in-progress`?" — fica verdadeira para sempre.

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

Não existe label `done`. Issue fechada já é a informação, e quem fecha é o GitHub.

---

## Abrir o PR

**Quando:** com tracker, o PR só abre quando a **SPEC inteira** fecha — todos os tickets em `in-review` ou fechados. Ticket cujo pai ainda tem irmãos abertos: dê push na branch, reporte o que falta, e **não abra PR**. Senão a SPEC chega na `main` em pedaços.

```bash
git push origin [BRANCH]
```

Abra com `gh pr create --draft --base main --title "[título da unidade]" --body-file [arquivo]`. Com SPEC fatiada, mova a SPEC para `in-review` logo depois (ver "Transições de estado"). O corpo, gravado antes num arquivo em `.scratch/`, contém:

- **`Closes #N`** — **uma linha por issue**, mais a da SPEC. A palavra-chave é **literal em inglês**: o GitHub só reconhece `close/closes/closed`, `fix/fixes/fixed`, `resolve/resolves/resolved`. Traduzir (`Fecha #27`) **falha em silêncio** — a linha é ignorada, o PR merga e as issues ficam abertas com o código na `main`. Já aconteceu. O resto do corpo continua em português. Omitir só sem tracker, que não tem issue.
- **O que entrou** — a lista de unidades implementadas, ou a descrição do que foi acordado no grill
- **Evidência por unidade** — commit, achados do revisor, e em quantos loops foram resolvidos
- **Gates finais** — o bloco `Checks` do retorno do `implementer`, **colado**, label por label; `não reportado` para o que não veio
- **Ficou de fora** — os tickets de follow-up abertos na arbitragem, cada um com número e uma linha em português; e os NITs do revisor
- **Baseline** — se ela estava vermelha no Bootstrap e o usuário mandou seguir, diga o quê estava vermelho
- **Review sem independência** — se a ferramenta não abriu subagentes e você mesmo revisou, diga isso
- a linha de atribuição que a sua ferramenta usa em PRs, se ela tiver uma

**Não faça merge.** O PR sai em draft e a decisão de merge é do usuário.

---

## Relatório final

O PR fica em draft esperando uma decisão humana, e quem decide é o dono do repositório — que pode não ler código. Seu relatório é o que torna essa decisão possível, e ela é **de risco, não de código**. Três blocos, em português comum:

> **O que entrou:** [em linguagem de produto — o que passou a funcionar, não quais arquivos mudaram]
>
> **O que ficou de fora:** [cada follow-up, com o que acontece se continuar assim]
>
> **Risco de mergear assim:** [alto / médio / baixo, e por quê — em uma frase que não exija ler o diff]

Informe a URL do PR. Se a SPEC veio de um PRD, informe que a próxima SPEC pode rodar depois do merge.

---

## Protocolo de Bloqueio

Ativado quando: achado de risco alto não resolvido após 2 loops; worker retorna ESCALAR; hard stop por princípio; implementador falha sem retornar IMPLEMENTADO; baseline vermelha no Bootstrap.

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
