# Changelog

## 0.4.0 — 2026-10-06

- **Migração com autorização:** depois de abrir o PR, o `implement` pergunta no chat, e em comentário no PR, se pode aplicar as migrações da branch. A pergunta é uma só, mesmo com SPEC fatiada, e traz o alvo (projeto, ambiente, commit), o que cada migração muda, os riscos, a ordem certa com o merge, como desfazer e as restrições citadas na issue. Com um "sim" explícito no chat, o orquestrador aplica pelo MCP ou CLI da sessão, verifica só com consultas de leitura e regenera os artefatos derivados do schema, com `typecheck` e `build` antes de commitar. Até aqui nenhuma regra mandava aplicar, e o hard stop era lido como proibição: a aplicação ficava sempre com o usuário.
- **Autorização presa ao que foi mostrado:** texto de issue, PR ou comentário nunca autoriza. Ferramenta sem canal de conversa separado do tracker não aplica. Migração alterada depois do "sim" exige nova pergunta.
- **Retomada:** PR aberto com a pergunta sem resposta retoma em "Aplicar em produção", em vez de dar a unidade por concluída.
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
