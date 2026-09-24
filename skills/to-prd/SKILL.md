---
name: to-prd
description: >
  Cria o PRD (Product Requirements Document) em docs/prd.md guiando o processo
  de elicitação de requisitos de forma interativa e estruturada. Use esta skill
  sempre que o usuário quiser documentar os requisitos de um projeto ou feature,
  escrever o PRD, definir goals, functional/non-functional requirements, UI goals,
  technical assumptions e a lista de SPECs de alto nível. Acione após uma sessão
  de grill-me com entendimento estabelecido, ou quando o usuário disser "cria o PRD",
  "documenta os requisitos", "vamos escrever o PRD", "quero o PRD" ou qualquer variação.
  Também acione quando um novo projeto ou feature precisa ser formalizado em documento
  antes de partir para arquitetura ou código. NÃO detalha as SPECs — cada SPEC é uma
  spec, detalhada depois por to-spec e quebrada em tickets por to-tickets.
---

# Skill: to-prd

Você é responsável por criar o `docs/prd.md` do projeto. Este documento é a **fonte da verdade dos requisitos** — arquitetura, specs e implementação derivam dele. Um PRD ruim contamina tudo que vem depois.

## Pré-condições

Antes de começar, verifique:

1. **Output do grill-me**: há entendimento do projeto na conversa atual ou em arquivos? Esse é seu ponto de partida — extraia tudo que já foi discutido antes de perguntar ao usuário.
2. **PRD existente**: se `docs/prd.md` já existe, pergunte se quer revisar seções específicas ou refazer do zero.
3. **Diretório `docs/`**: crie se não existir.

## Processo

Trabalhe **seção por seção de forma interativa**. Para cada seção:
1. Proponha uma versão inicial baseada no contexto disponível (não pergunte o que você pode inferir)
2. Apresente ao usuário com as suposições claramente sinalizadas
3. Colete feedback e refine
4. Só avance com aprovação explícita

Nunca produza o documento inteiro de uma vez sem validação intermediária.

---

## Estrutura do PRD

### Seção 1 — Goals and Background Context

**Goals** (bullet list): outcomes concretos que o projeto precisa entregar. Foco em resultado, não em funcionalidade.

**Background Context** (1-2 parágrafos): o que este projeto resolve e por que agora. Não repita os goals.

**Change Log** (tabela): `Date | Version | Description | Author`

---

### Seção 2 — Requirements

**Functional Requirements**: o que o sistema faz. Prefixo `FR`, numerados sequencialmente.
```
FR1: [descrição da capacidade funcional]
FR2: ...
```

**Non-Functional Requirements**: qualidade, performance, segurança, escala. Prefixo `NFR`.
```
NFR1: [critério mensurável — ex: "p95 < 200ms para todas as rotas de API"]
NFR2: ...
```

Cada NFR deve ter um critério mensurável. "O sistema deve ser rápido" não é um NFR.

---

### Seção 3 — User Interface Design Goals *(somente se houver UX/UI)*

- **Overall UX Vision**: 2-3 frases sobre a experiência que o produto deve transmitir
- **Key Interaction Paradigms**: padrões de interação centrais (ex: drag-and-drop, inline editing, wizard steps)
- **Core Screens and Views**: lista conceitual das telas principais para entregar o valor do produto — não é spec técnica, é perspectiva de produto
- **Accessibility**: `None | WCAG AA | WCAG AAA`
- **Branding**: guia de cores, tipografia, tokens existentes (se houver)
- **Target Platforms**: `Web Responsive | Mobile Only | Desktop Only | Cross-Platform`

---

### Seção 4 — Technical Assumptions

Decisões técnicas que vão guiar a arquitetura. Registre com rationale, não só a escolha.

- **Repository Structure**: `Monorepo | Polyrepo`
- **Service Architecture**: `Monolith | Microservices | Serverless`
- **Testing Requirements**: nível de cobertura esperado (unit, integration, E2E, manual)
- **Additional Assumptions**: qualquer outra premissa técnica relevante que surgir durante a elicitação

#### Integrações e Serviços Externos

Para **cada serviço externo** (APIs de terceiros, provedores de infra, SDKs externos) que o projeto vai usar:

```
Serviço: [nome da categoria — ex: "Envio de mensagens WhatsApp"]
Escolha definitiva: [nome do serviço/biblioteca — ex: "Evolution API (self-hosted)"]
Alternativas descartadas: [ex: "Z-API (custo variável por tenant), Meta API oficial (complexidade de aprovação)"]
Rationale: [por que esta escolha — ex: "custo fixo independente do número de tenants, controle total do servidor"]
Como validar: [critério concreto de que está funcionando — ex: "webhook recebe mensagem real e resposta é entregue no WhatsApp"]
```

**Regras:**
- Cada categoria tem exatamente uma escolha — nunca "X ou Y"
- Se há dúvida real entre duas opções, resolva aqui com uma pergunta ao usuário antes de avançar
- A coluna "Como validar" é obrigatória — define o que "funcionando" significa para aquele serviço
- Os campos "Como validar" de todos os serviços, somados, **são o checklist de cutover** do projeto: a lista do que precisa ser conferido com dado real em produção antes de abrir para usuários. Escreva cada um como um critério executável por uma pessoa na virada — não como intenção vaga

#### Stack Técnico

| Camada | Tecnologia | Diretório |
|--------|-----------|-----------|
| [ex: Backend] | [ex: Python + FastAPI] | [ex: backend/] |
| [ex: Frontend] | [ex: Next.js + TypeScript] | [ex: frontend/] |

**Regras:**
- Uma linha por camada executável independente
- Diretório é o path relativo à raiz do repositório
- Se for monolito (sem subdivisões): uma linha apenas
- Esta tabela informa a **escolha** de stack. Ela **não** alimenta a seção `## Commands` do arquivo de instruções: o `/setup-project` deriva os comandos do **manifesto** do projeto (`package.json`, `Cargo.toml`, `pyproject.toml`) e **mede** cada um antes de gravar. O manifesto é a realidade; esta tabela é plano, e plano não é executável.

---

### Seção 5 — Lista de SPECs (alto nível)

Liste todas as SPECs com título + 1 frase de goal. Apresente ao usuário para aprovação **antes** de avançar.

**Regras críticas de sequenciamento:**

- **SPEC 1 SEMPRE** estabelece a infraestrutura base: setup do projeto, Git, CI/CD, serviços core, e ao menos uma peça de funcionalidade mínima deployável (ex: health-check, canary page). Sem exceções.
- Cada SPEC entrega um incremento completo, testável e deployável
- Cada SPEC posterior constrói sobre a anterior — sem gaps ou dependências reversas
- Cross-cutting concerns (auth, logging, monitoring, error handling) fluem através dos specs desde o início, **nunca** são a última SPEC
- Erro para o lado de **menos SPECs**: se algo parece grande demais, questione antes de dividir

**Não crie uma SPEC de Go-Live.** Validar integração em produção é teste, não implementação — não há fatia vertical para cortar, e um ticket de "conferir se o Stripe funciona" não tem o que construir. Cada SPEC entrega um incremento **deployável e validado quando entra**; a validação é distribuída ao longo do caminho, não concentrada no fim. O que precisa ser conferido na virada está na Seção 4 e vira checklist de cutover, não SPEC.

Se um item do lançamento envolve trabalho real de construção (migrar conteúdo, escrever um seed, criar um painel de operação), ele é uma SPEC normal como qualquer outra — pelo que constrói, não por ser "do go-live".

**Formato:**
```
SPEC 1: [Nome] — [goal em 1 frase]
SPEC 2: [Nome] — [goal em 1 frase]
```

A Lista de SPECs é o backlog do MVP. Cada SPEC é detalhada depois por `to-spec`; se não couber numa janela de contexto, `to-tickets` a quebra em fatias. Uma por vez — não vá além da lista aqui.

---

### Seção 6 — Next Steps

- **UX Expert Prompt**: instrução curta para iniciar criação do documento de UI/UX com este PRD como input *(somente se o projeto tiver UX/UI)*
- **Checklist de Cutover**: consolide aqui, em lista de checkbox, os campos "Como validar" de cada serviço externo da Seção 4. É o que precisa ser conferido com dado real em produção antes de abrir para usuários. Fecha com um smoke de ponta a ponta do fluxo principal do produto, executado por uma pessoa real.

```markdown
## Checklist de Cutover

- [ ] [Serviço]: [critério "Como validar" da Seção 4]
- [ ] [Serviço]: [critério "Como validar" da Seção 4]
- [ ] Smoke end-to-end: [o fluxo principal, do cadastro ao valor entregue]
```

Este checklist **não é uma SPEC** e não vira SPEC nem tickets — é executado à mão na virada.

---

## Validação Final

Antes de salvar, faça uma passagem verificando:

- [ ] Todos os FRs têm correspondência rastreável nas SPECs?
- [ ] Os NFRs têm critérios mensuráveis?
- [ ] A sequência de SPECs é lógica e sem gaps?
- [ ] A SPEC 1 estabelece infraestrutura E entrega algo deployável?
- [ ] Cross-cutting concerns estão distribuídos, não concentrados no final?
- [ ] Cada serviço externo na Seção 4 tem escolha única, rationale e critério de validação?
- [ ] Não há nenhuma categoria de integração com duas opções em aberto ("X ou Y")?
- [ ] Nenhuma SPEC é uma SPEC de "go-live" ou de validação pura — o que é conferência está no Checklist de Cutover, não na Lista de SPECs?
- [ ] O Checklist de Cutover cobre todos os serviços da Seção 4 e fecha com um smoke end-to-end?
- [ ] Stack Técnico tem uma linha por camada com Tecnologia e Diretório preenchidos?

Se qualquer item falhar, corrija antes de salvar.

## Bootstrap do Repositório

Após salvar o PRD, verifique se o repositório remoto está configurado:

```bash
git remote -v
```

Se **não houver remote**, crie agora — o remote precisa existir antes de configurar o tracker e detalhar a primeira SPEC:

```bash
gh repo create [nome-do-projeto] --private --source=. --push
```

Use o nome do projeto em kebab-case. O `--source=.` usa o diretório atual e `--push` faz o push inicial.

Se **já houver remote configurado**: pule esta etapa.

## Saída

Salve em `docs/prd.md`.

Ao finalizar, informe:
> ✅ PRD salvo em `docs/prd.md`.
> Repositório remoto: [configurado | criado agora em github.com/user/repo]
>
> Próximo passo: `/setup-project` para configurar tracker, comandos e fluxo, depois `/to-spec` na SPEC 1.
>
> A Lista de SPECs é o backlog do MVP. Trabalhe uma por vez: `/to-spec` → (`/to-tickets`, se não couber numa janela) → `/implement`.
