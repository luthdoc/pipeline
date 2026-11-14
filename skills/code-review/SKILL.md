---
name: code-review
description: >
  Revisão de segurança em sessão nova para issue ou PR com o rótulo risk:alto.
  Confronta a SPEC com o diff e aplica o checklist de segurança (S1–S5:
  segredos, input, autenticação, dados mínimos, filtro de origem e RLS).
  Roda numa sessão com contexto limpo, não na que implementou. Não altera
  código e não faz merge. Use: /code-review 12, /code-review <URL do PR>.
disable-model-invocation: true
---

# Skill: code-review

Você revisa o diff de um PR `risk:alto` (ou de uma issue com esse rótulo) **numa sessão com contexto limpo**. O produto é uma lista de achados — ou a confirmação de que a revisão está limpa. Depois disso, se estiver limpa, você move a issue alvo para `in-review`. Você **não altera código**, **não executa testes**, **não faz merge** e **não abre um segundo PR**.

Esta skill existe porque o revisor da mesma sessão que implementou já deixou passar um defeito de segurança (filtro que aceitava eventos de outra origem). Subagente com contexto "limpo" na sessão do `/implement` não basta: o raciocínio de quem implementou continua na sessão. Por isso o `/implement` não roda esta skill. A passada de segurança *dentro* do `/implement` continua existindo, como primeiro filtro; esta sessão é a revisão que libera o `in-review` e o merge.

## Sessão nova

Se esta conversa implementou o código deste PR ou desta issue — ou se o histórico contém o loop de implementação desta unidade (`implementer`, "Implementado e aprovado", correções da mesma branch) — **pare**. Peça ao usuário para abrir uma sessão nova, só com `/code-review <PR ou issue>`, e encerre. Não revise aqui: o contexto já não está limpo.

## Entrada

O argumento é o número ou a URL de um PR, ou o número ou a URL da issue (SPEC ou issue sem tickets). Sem argumento, pergunte. A partir da issue, localize o PR aberto cuja branch é a da SPEC (`spec-<N>-<slug>`). Sem PR, encerre pedindo que o `/implement` o abra primeiro.

Leia a seção `## Issue Tracker` do arquivo de instruções (`AGENTS.md`, ou `CLAUDE.md` na falta dele) e `docs/agents/issue-tracker.md`. Sem eles, encerre pedindo `/setup-project`.

## O que revisar

1. **O diff** — `git fetch origin`, depois `git diff main...HEAD` na branch do PR (três pontos, contra o merge-base). Também a lista de commits: `git log main..HEAD --oneline`.
2. **A SPEC** — o corpo da issue alvo (a SPEC pai, ou a issue sem tickets) e, quando existir, a cópia em `.scratch/*/spec.md` localizada pela linha `#<N>`. Não invente um caminho. É o confronto SPEC × diff: requisito pedido que falta ou está errado, e comportamento no diff que a SPEC não pediu, entram como achado quando tiverem efeito de segurança.
3. **O checklist de segurança**, nesta ordem, lendo `skills/implement/review-rules.md` (localize o arquivo a partir da raiz do repo, inclusive pastas ocultas) — **só o eixo S, S1–S5**. Não percorra T, CL, C, SM nem `docs/agents/review-rules-projeto.md`.

   | Id | O que cobre |
   |---|---|
   | S1 | Segredos no código ou expostos ao cliente |
   | S2 | Validação de input externo |
   | S3 | Autenticação e autorização no servidor, inclusive RLS e permissão |
   | S4 | Dados retornados mínimos |
   | S5 | Filtro de origem e fronteira de confiança (número, tenant, conta, webhook) |

   Na S3, confira também políticas RLS e checagens de permissão que o diff toca ou deveria ter tocado. Na S5, o caso negativo (origem errada aceita) é CRITICAL.

Você não executa nada e não altera nada. As restrições do `reviewer.md` valem: sem delegar a revisão, num passe único. Cada achado cita `arquivo:linha`, a classe, se é de risco alto, e o trecho que o sustenta. O formato é o de `ACHADOS` / `LIMPO` do `reviewer.md`.

## Resultado

Grave o relatório num arquivo em `.scratch/` e publique com `--body-file`.

- **`LIMPO`, ou só NIT** — comente no PR o resultado ("Revisão de segurança, sessão nova: limpa"). Atualize `## Revisão de segurança` no corpo do PR para "Limpa, sessão nova `/code-review`." Mova a issue alvo (SPEC ou issue sem tickets) para `in-review`, removendo `in-progress` na mesma operação. Diga ao usuário que o PR pode seguir para merge — você não mergeia.
- **CRITICAL ou WARN** — comente os achados no PR, verbatim. **Não** mova para `in-review`. **Não** corrija nesta sessão: a correção volta ao `/implement` (ou a uma sessão de correção), e esta skill roda de novo, outra vez numa sessão nova, depois do ajuste. Informe o que bloqueia o merge.
- **`ESCALAR`** — use a mesma régua do `reviewer.md`: só decisão de produto ou de risco. Não mergeie.

Sempre informe a URL do PR e se o `in-review` foi aplicado ou não.
