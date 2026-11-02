---
name: prototype
description: >
  Abre a sessão do protótipo das telas e descobre sozinha em qual momento está pelo
  estado do repo. Protótipo inicial: `docs/agents/prototype.md` sem link — cria o
  artifact a partir do `docs/prd.md` da `origin/main`. Alteração: já tem link e veio
  uma issue com needs-prototype — altera o artifact existente com as decisões da issue.
  Nos dois casos a sessão fica aberta para os ajustes e termina com /close-prototype.
  Use: /prototype (protótipo inicial) ou /prototype 12 (issue needs-prototype).
disable-model-invocation: true
---

# Skill: prototype

O protótipo do projeto é um artifact do claude.ai, um só por projeto, com as telas. Não vira rota no código. Esta skill abre a sessão em que ele é criado ou alterado, e o `/close-prototype`, na mesma sessão, fecha a etapa.

## 1. Descobrir o momento

Rode `git fetch origin` e leia `docs/agents/prototype.md` da `origin/main` (`git show origin/main:docs/agents/prototype.md`), não da branch local, que pode estar velha.

- **Sem link** (ou sem o arquivo) → **protótipo inicial**. Se veio uma issue no argumento, avise que ela fica para depois: primeiro o projeto precisa do protótipo inicial.
- **Com link e uma issue `needs-prototype`** no argumento → **alteração**.
- **Com link e sem issue** → pergunte qual issue `needs-prototype`. Não crie um segundo artifact.

Diga ao usuário em uma linha qual momento detectou.

## 2. Protótipo inicial

1. Leia `docs/prd.md` da `origin/main` (`git show origin/main:docs/prd.md`). Se não existir, pare e avise: o protótipo inicial nasce do PRD, e o passo anterior é o `/to-prd`.
2. Procure um documento de design na `origin/main` (`docs/design.md`, ou `DESIGN.md` na raiz).
   - **Existe** → siga as especificações de design dele.
   - **Não existe** → wireframe minimalista neutro: tons de cinza, fonte do sistema, hierarquia só por tamanho e peso, imagens como caixa com X. Nada de cores, fontes ou identidade de marca.
3. Com ou sem design, o protótipo traz os fluxos da jornada do usuário e as especificações do PRD.
4. Crie o artifact sempre do tipo **Design** (`/design`), um só por projeto: pranchetas de celular (390×844) e desktop (1280×800), ligadas por links, com Play navegável.
5. Não invente dados reais (contato, endereço, preço, nomes de pessoas). Conteúdo de exemplo vai marcado como exemplo.
6. Passe ao usuário o nome e o link do artifact e fique aberta para os ajustes.

**Não mexa no repo durante os ajustes:** nenhum commit, branch ou PR. O nome e o link só entram em `docs/agents/prototype.md` pelo `/close-prototype`, depois da aprovação.

## 3. Alteração

Leia a issue com o comando "Ver" de `docs/agents/issue-tracker.md` (corpo e todos os comentários). Altere o artifact indicado em `docs/agents/prototype.md` com as decisões da issue, sem criar outro, e fique aberta para os ajustes.

## Nos dois momentos

- **Os ajustes vêm pelo chat desta sessão.** Comentário deixado direto no artifact não chega à sessão; se o usuário citar um, peça que mande o pedido aqui.
- **Nunca force push.** Se a branch da sessão foi apagada no merge, crie uma branch com nome novo a partir da `origin/main` atualizada.
- Quando o usuário aprovar o protótipo, o próximo passo é `/close-prototype` nesta mesma sessão (`/close-prototype <N>` na alteração).
