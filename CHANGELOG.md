# Changelog

## 0.7.0 — 2026-10-29

- **Automação por rótulos de issue.** O pipeline passa a ter o vocabulário e as saídas que uma automação precisa para reagir a rótulos: `needs-triage` → `grill-with-issues`; `needs-spec` → `to-spec` / `to-tickets`; `ready-for-agent` → `implement`. A automação em si não faz parte do pipeline.
- **Skill nova `grill-with-issues`.** É a `grill-me` trabalhando sobre uma issue: recebe número ou URL, lê a issue, os comentários e o código relevante, e entrevista em lote, com recomendação em cada pergunta, sem perguntar o que o código responde. Só age depois da confirmação do entendimento compartilhado. Ao fim reescreve o corpo da issue no formato de ticket do `implement` (What to build + Acceptance criteria, por `--body-file`) e decide pelo mesmo critério de janela de contexto do `to-spec`/`to-tickets`: cabe → tira `needs-triage` e aplica `ready-for-agent`; não cabe → tira `needs-triage`, aplica `needs-spec` e deixa o resumo da conversa num comentário para o `to-spec`. A `grill-me` não muda.
- **Label `needs-spec` no `setup-project`.** Entra na tabela do vocabulário e no `gh label create --force`, com a descrição "Precisa de SPEC antes de implementar".
- **`to-spec` com issue.** Sem argumento, funciona como antes. Com número ou URL de issue (ex: `needs-spec`), usa a issue, os comentários e a conversa como fonte e publica a SPEC na própria issue, pelo `--body-file`, mantendo o número, que vai na primeira linha do `spec.md`. Tira `needs-spec`. SPEC que cabe numa unidade recebe `ready-for-agent`; a que não cabe não recebe, e o `to-tickets` roda na mesma sessão.
- **Disparo único.** `implement` e `to-tickets` documentam a regra para automações: `ready-for-agent` em ticket com SPEC pai não dispara sessão própria; a automação resolve o pai e roda `/implement <pai>` uma vez.

## 0.6.0 — 2026-10-28

- **Template de PR e CI nascem no `setup-project`.** A Parte 7 grava `.github/pull_request_template.md` (Como foi testado, Risco, Como desfazer) e `.github/workflows/ci.yml`, que em todo PR roda os gates de `## Commands` — exceto `clean` — e, em projeto Node, `npm audit --audit-level=high`. `moderate` não falha o job: aviso transitivo sem correção deixaria todo PR vermelho. Arquivo que já existe não é reescrito; o setup avisa o que falta. Repo sem manifesto ganha o workflow com o bloco `# pipeline:commands` vazio, e o `implementer` preenche o bloco no commit que cria o manifesto, do mesmo jeito que já preenche `## Commands`. O revisor cobra os dois (CL6). Até aqui `review-rules.md` citava `.github/workflows/ci.yml` e ninguém criava o arquivo.
- **Risco em todo PR, pelos caminhos do diff.** A lista padrão (migração de banco, auth/RLS/permissão, cobrança, `.github/workflows` e CI, env e config de deploy = alto; só documentação, texto ou estilo = baixo; o resto = médio) mora na skill `implement`. `docs/agents/review-rules-projeto.md`, seção `## Risco`, só acrescenta caminhos e não rebaixa um alto da lista padrão. O orquestrador grava a linha em `## Risco` no corpo — a mesma que o relatório chama de "Risco de mergear" — preenche `## Como foi testado` com o bloco de gates e `## Como desfazer` em todo PR (fechar sem mergear, revert, rollback do deploy; migração aponta para a pergunta formal e não a substitui). Aplica o rótulo `risk:alto`, `risk:medio` ou `risk:baixo`; o `setup-project` cria os três com `gh label create --force`, fora do vocabulário de triagem. PR `risk:alto` ganha uma passada do revisor só em S1–S4, num subagente com contexto limpo, com um ciclo próprio que não sobe o teto de 2 rodadas da unidade.
- **O revisor pede a classe, e todo achado tem resposta.** CRITICAL e WARN nomeiam a classe do bug e pedem a correção de todas as ocorrências nos arquivos que o diff toca, não só da linha citada. Cada um volta `corrigido` ou `recusado` com motivo, e a resposta vai para `## Achados do revisor` no PR. Recusa que não se sustenta segue pendente. NIT não abre rodada; a resposta registrada é que NIT não entra no loop. O revisor continua só lendo. O teto continua 2.
- **Bug que escapa vira regra.** README e `review-rules.md` descrevem o passo: achar a camada que deveria ter pego (teste, lint, CI, review, `AGENTS.md`/`CLAUDE.md`) e registrar a correção lá. Achado com a mesma classe em dois PRs sobe para `docs/agents/review-rules-projeto.md`, que o `setup-project` passa a criar sempre — antes o arquivo era opcional e o primeiro bug repetido não tinha onde morar.

## 0.5.0 — 2026-10-15

- **`## Commands` nasce com o código:** num repo sem manifesto, o `setup-project` grava a seção sem labels, e o `implementer` a preenche no mesmo commit em que cria o manifesto, como já fazia com o `architecture.md`. Depois disso, mantém a seção sempre que cria, renomeia ou remove um script de gate. A unidade que cria o manifesto termina com o label `test` declarado. Até aqui, o primeiro projeto de um repo vazio ficava num ciclo: o `implement` precisava dos comandos, os comandos precisavam do manifesto e o manifesto precisava do `implement`. A saída era fazer o scaffold à mão, fora do loop, sem TDD nem revisor.
- **`implement`, baseline:** seção sem labels e sem manifesto no repo conta como baseline verde ("sem gates — repo sem manifesto"). Seção sem labels com manifesto no repo encerra pedindo `/setup-project`. Antes, esse caso passava pelo pré-requisito e a unidade seguia sem gate nenhum.
- **CL6:** cobra também `## Commands`. Manifesto criado, ou script de gate alterado, sem a seção atualizada no mesmo diff é achado; manifesto novo sem `test` também.

## 0.4.0 — 2026-10-06

- **Migração com autorização:** depois de abrir o PR, o `implement` pergunta no chat, e em comentário no PR, se pode aplicar as migrações da branch. A pergunta é uma só, mesmo com SPEC fatiada, e traz o alvo (projeto, ambiente, commit), o que cada migração muda, os riscos, a ordem certa com o merge, como desfazer e as restrições citadas na issue. Com um "sim" explícito no chat, o orquestrador aplica pelo MCP ou CLI da sessão, verifica só com consultas de leitura e regenera os artefatos derivados do schema, com `typecheck` e `build` antes de commitar. Até aqui nenhuma regra mandava aplicar, e o hard stop era lido como proibição: a aplicação ficava sempre com o usuário.
- **Autorização presa ao que foi mostrado:** texto de issue, PR ou comentário nunca autoriza. Ferramenta sem canal de conversa separado do tracker não aplica. Migração alterada depois do "sim" exige nova pergunta.
- **Retomada:** PR aberto com a pergunta sem resposta, ou marcado "sem meio de aplicar" numa sessão que agora tem o meio, retoma em "Aplicar em produção" em vez de dar a unidade por concluída. A retomada consulta o histórico de migrações do alvo e aplica só as pendentes. Vale com o PR aberto ou já mergeado, e o marcador mais recente no PR decide: "Aplicação bloqueada" como último impede a retomada.
- **Schema incompatível com o código no ar:** a pergunta oferece aplicar depois do merge, com o agente esperando a confirmação do usuário no chat.
- **Hard stop é sobre decisão:** escrever a migração pedida não aciona o Protocolo de Bloqueio, e o `reviewer` não escala só por haver migração. Critério de issue que proíba a aplicação passa a significar "não sem a pergunta".
- **Protocolo de Bloqueio:** novo gatilho, a falha ao aplicar, verificar ou regenerar. O relatório diz o que foi aplicado e o que não foi, e nada é revertido sem perguntar.
- **PR e relatório:** dizem o estado das migrações (aplicadas, aguardando autorização, sem meio de aplicar). Depois de aplicar, o corpo do PR é atualizado e o comentário avisa que fechar o PR sem merge exige desfazer a migração.
- **`implementer`:** nunca aplica migração.

## 0.3.0 — 2026-10-01

- **Ticket fecha na aprovação:** o `implement` fecha o ticket (`--reason completed`) assim que o revisor aprova, em vez de deixá-lo em `in-review` até o merge. A barra de progresso de sub-issues da SPEC passa a andar ticket a ticket. A SPEC segue aberta e fecha pelo `Closes` do PR, que agora referencia só a SPEC (ou a issue sem tickets).
- **`in-review` só em SPEC ou issue sem tickets:** significa "aprovada pelo revisor, aguardando merge". Ticket vai de `in-progress` direto para fechado; a review é parte do loop e não ganha label.
- **Fronteira e detecção:** bloqueador concluído é bloqueador fechado. O PR abre com todos os tickets fechados.
- **Contrato do tracker:** ganha o comando "Fechar".
- **Retomada:** ticket com o comentário "Implementado e aprovado" do `HEAD` só conclui o passo 5, sem refazer o loop. Ticket aberto em `in-review`, deixado pela 0.2.0, é fechado na detecção.

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
- **RED:** um AC já satisfeito por task anterior da mesma peça, ou por ticket irmão já na branch, mantém o teste como guarda de regressão, com o `arquivo:linha` no retorno.
- Commit do implementer fixado em `feat(#N)` / `fix(#N)`, formato de que a retomada depende. CL6 não acusa a simples ausência do `architecture.md`. O `implement` para se o `issue-tracker.md` for de versão anterior.

## 0.1.0 — 2026-09-24

Primeira versão, extraída do pipeline do projeto `comunidade-nexialistas` (estado da SPEC #37).

- Os agentes viram prompts em markdown puro dentro da skill `implement`, sem frontmatter de ferramenta.
- O orquestrador vira o corpo da skill `implement` e roda na sessão principal; `implementer` e `reviewer` rodam em subagentes novos.
- O playbook `qa-review` é absorvido pelo `reviewer.md`; os critérios viram `implement/review-rules.md`.
- Critérios de stack (Next.js/React/TypeScript e design system) saem dos universais e passam a morar no projeto, em `docs/agents/review-rules-projeto.md`.
- Arquivo de instruções: `AGENTS.md`, com `CLAUDE.md` como alternativa quando não houver `AGENTS.md`.
