# Regras de Review

Todo critério aqui é **verificável por leitura**. O revisor não executa nada — lint, typecheck, build e a suite de testes já rodaram no gate do `implementer`, e o que estas regras cobrem é justamente o que essas ferramentas não pegam.

## Regra de admissão de critério

Um critério só pertence a este arquivo se o revisor puder julgá-lo **lendo o diff e o código**. Duas classes ficam de fora, por construção:

1. **Critério que exige executar algo** não é do revisor. Ele não executa nada. Se algo precisa rodar, é label de `## Commands` — e o resultado chega colado no retorno do worker que o executou — ou é passo da CI. Um critério que manda o revisor rodar um comando só pode produzir duas coisas: achado inventado ou omissão silenciosa.
2. **Prática que só o autor pode atestar** não é critério de review. "Escreveu o teste antes do código" não é observável num diff — o diff mostra o resultado, não a ordem em que foi digitado. Isso é regra do worker que faz, e mora no `implementer.md`.

Numa equipe real ninguém pede ao revisor que rode `npm audit`, e ninguém audita se o autor escreveu o teste primeiro. As duas regras estão no papel errado. Este arquivo já afirmava "todo critério aqui é verificável por leitura" e se contradizia em dois pontos; a afirmação agora vale.

## Os eixos

Cinco eixos universais, nesta ordem de severidade: **S** segurança, **T** testes, **CL** limpeza, **C** complexidade, **SM** code smells.

Critérios de stack, de framework ou de design system não moram aqui — valem só para um projeto, e moram em `docs/agents/review-rules.md` do próprio projeto, quando ele os tem.

Reporte só achados com evidência concreta — `arquivo:linha` mais o trecho que sustenta. Suspeita sem evidência não é achado.

Escopo: **o que o diff mudou**. Dívida pré-existente que o diff apenas tangenciou é NIT, nunca bloqueio deste ticket.

**Marque risco alto.** Para cada CRITICAL, diga se ele é de risco alto — segurança ou perda de dado. É a única classificação que decide o destino do achado se ele sobreviver a duas rodadas de correção.

---

# Segurança (S)

## S1 — Nenhum segredo hardcoded

**Regra:** nenhuma chave de API, senha, token, connection string, secret, ou credencial pode aparecer literal no código-fonte.

**Como checar:** busque por padrões como strings longas com caracteres aleatórios, palavras-chave `password`, `secret`, `token`, `key`, `api_key`, `auth` atribuídas a strings literais.

Busque sobre os arquivos tocados pelo diff:

- `(password|secret|token|api[_-]?key|auth)\s*[:=]\s*['\"]` — atribuição literal
- `(sk|pk|ghp|xox|eyJ)[-_A-Za-z0-9]{16,}` — prefixos de chave conhecidos
- literais com 32+ caracteres alfanuméricos sem espaço

Toda credencial deve vir de variável de ambiente. Atenção extra às variáveis que o framework expõe ao cliente por prefixo (`NEXT_PUBLIC_*`, `VITE_*`, `PUBLIC_*` e equivalentes): o valor é embutido no bundle entregue ao navegador e é público por construção — segredo com esse prefixo é achado CRITICAL mesmo vindo de env.

**O que fazer se falhar:** mova para variável de ambiente. Documente no README quais variáveis são necessárias. Invalide o segredo exposto imediatamente.

---

## S2 — Todo input externo é validado antes de ser usado

**Regra:** qualquer dado vindo de fora do sistema (body de requisição, query params, headers, arquivos, eventos externos) deve ser validado quanto a tipo, formato e limites antes de ser processado ou persistido.

**Por quê:** input não validado é a origem de injection attacks, crashes por tipo inesperado, e corrupção de dados.

**Como checar:** trace cada ponto de entrada do sistema. Verifique se há validação explícita (schema, parser, assertions) antes de qualquer uso do dado.

**O que fazer se falhar:** adicione validação no ponto de entrada. A validação deve rejeitar explicitamente o que não está no contrato esperado — não apenas ignorar campos extras.

---

## S3 — Autorização verificada no servidor

**Regra:** toda operação que exige permissão deve verificar autorização no servidor, independente de qualquer verificação feita no cliente.

**Por quê:** verificações client-side são UX, não segurança. Qualquer requisição pode ser feita diretamente ao servidor sem passar pela interface.

**Como checar:** para cada rota ou endpoint que manipula dados sensíveis ou restritos, verifique se há verificação de sessão/token e de permissão no handler servidor. Verificação apenas no middleware de rota não é suficiente se o endpoint pode ser chamado diretamente.

---

## S4 — Dados retornados são mínimos necessários

**Regra:** nenhuma resposta pode retornar mais dados do que o consumidor precisa para a operação solicitada.

**Por quê:** over-fetching expõe campos sensíveis acidentalmente e aumenta a superfície de ataque.

**Como checar:** para cada response de API ou query, verifique se campos como senha (mesmo hash), tokens internos, IDs de sistema, dados de outros usuários, ou metadados internos estão sendo retornados sem necessidade explícita.

```
// ❌ retorna tudo do registro
return await db.users.findById(id);

// ✅ projeta apenas o necessário
return await db.users.findById(id, { select: ['id', 'name', 'email'] });
```

> **Vulnerabilidade de dependência não é critério de review.** Ela exige executar `npm audit`, e o revisor não executa nada — cai na regra de admissão. Vive na CI (`.github/workflows/ci.yml`), que roda `npm audit --audit-level=high` num checkout limpo. E o lugar é esse por um motivo mais forte que a ferramenta: vulnerabilidade não aparece porque alguém mexeu no manifesto, aparece porque alguém publicou um aviso sobre um pacote que já estava lá. Um gate condicional ao diff nunca pegaria o caso comum.

---

# Testes (T)

## T1 — Toda lógica nova tem teste que a protege

**Regra:** cada unidade de **lógica de negócio** introduzida ou alterada pelo diff tem ao menos um teste, no próprio diff, que falharia se aquela lógica fosse revertida.

Conta como lógica de negócio: condicional, cálculo, transformação de dados, parsing, validação, tratamento de erro, decisão de roteamento, chamada de I/O com contrato próprio.

**Não** conta (isento de T1): JSX puramente apresentacional sem condicional, tipos e interfaces TypeScript, constantes e tokens de design, arquivos de configuração, mocks e fixtures, migrations e schemas gerados, re-exports.

**Como checar** — por leitura do diff, sem executar nada:

1. Liste cada trecho de lógica de negócio adicionado ou modificado (arquivo:linha).
2. Para cada um, localize no diff o teste correspondente e o assert específico que o cobre.
3. Aplique o teste de reversão mental: *se eu invertesse esta condição / apagasse este branch, algum assert do diff quebraria?* Se não, o trecho está descoberto.

**Achado T1 =** trecho de lógica sem assert que o proteja. Reporte `arquivo:linha` da lógica **e** o motivo (`nenhum teste no diff` ou `teste existe mas não falharia na reversão`).

**Não conte linhas.** Ratio linha-a-linha entre código e teste produz falso positivo em código majoritariamente JSX e não é medida de proteção.

---

## T2 — Testes cobrem comportamento, não implementação

**Regra:** nenhum teste pode verificar detalhes internos de implementação — nomes de funções privadas, estrutura interna de objetos, sequência de chamadas internas.

**Por quê:** testes de implementação quebram quando você refatora sem mudar comportamento. São falsos positivos e freios ao refactoring legítimo.

**Como checar:** leia o nome e o assert de cada teste. Se o teste quebraria ao renomear uma função interna sem mudar o comportamento externo, é um teste de implementação.

```
// ❌ testa implementação
expect(userService._hashPassword).toHaveBeenCalled();

// ✅ testa comportamento
expect(await login(wrongPassword)).toEqual({ error: 'invalid_credentials' });
```

---

## T3 — Cada teste tem exatamente um motivo para falhar

**Regra:** nenhum teste pode verificar múltiplos comportamentos independentes em um único caso.

**Por quê:** testes com múltiplos asserts verificando coisas diferentes mascaram qual comportamento quebrou e tornam o diagnóstico mais lento.

**Como checar:** se um teste falhar, a mensagem de falha deve identificar exatamente qual comportamento quebrou sem ambiguidade.

**Exceção permitida:** múltiplos asserts são permitidos quando verificam facetas do mesmo comportamento coeso (ex: um objeto retornado com seus campos relacionados).

---

## T4 — Nome do teste descreve o comportamento esperado

**Regra:** o nome de todo teste deve ser legível como uma frase que descreve o que o sistema faz em determinada condição.

**Formato recomendado:** `[contexto] — [ação] — [resultado esperado]`

```
// ❌
test('checkAuth')
test('returns false')
test('user test 3')

// ✅
test('usuário sem token — acessa rota protegida — recebe 401')
test('item fora de estoque — tentativa de compra — retorna erro com motivo')
```

---

> **O RED do TDD não é critério de review.** "Execute os testes antes de escrever o código de produção" é uma prática do autor: o diff mostra o resultado, não a ordem em que foi digitado, e nenhuma leitura distingue um teste escrito antes de um escrito depois. Cai na regra de admissão, e mora no ciclo TDD do `implementer.md`. O que **é** verificável por leitura, e continua aqui, é o teste que não protege nada — T1, pelo teste de reversão mental.

---

## T5 — Sem testes desabilitados sem justificativa

**Regra:** nenhum teste pode estar skipado (`skip`, `xtest`, `xit`, `.todo`, `pending`) sem um comentário explicando por quê e um item de tech debt registrado.

**Por quê:** testes skipados são buracos silenciosos na cobertura. Se não pode passar agora, documente o motivo e registre o débito.

---

# Limpeza (CL)

## CL1 — Sem código morto

**Regra:** nenhuma função, variável, import, export, classe, ou bloco de código pode existir no código-fonte sem ser referenciado e executado em algum caminho real do sistema.

**Como checar** — por leitura, sem executar o linter (lint e typecheck já rodaram no gate do `implementer`; aqui o alvo é o que eles não pegam): para cada símbolo **exportado** que o diff adiciona, busque um import ou uso dele em outro arquivo. Zero referências fora do próprio arquivo = código morto. Para símbolos locais, leia o corpo do arquivo e confirme que há ao menos um caminho de execução que os alcança.

**O que fazer se falhar:** delete. Se "pode ser útil no futuro", não pertence ao código agora — pertence ao backlog.

---

## CL2 — Sem código comentado

**Regra:** nenhum bloco de código pode estar comentado no código-fonte. Comentários explicam o *porquê* de uma decisão não-óbvia — nunca preservam código removido.

**Por quê:** código comentado é lixo com contexto perdido. Ninguém sabe se ainda é válido, se foi substituído, ou se pode ser deletado. O histórico de git existe para isso.

```
// ❌
// function oldCalculation(x) {
//   return x * 1.15;
// }

// ✅ (se a decisão não for óbvia)
// Usamos 1.15 porque inclui IOF — ver contrato com fornecedor X
const rate = 1.15;
```

**O que fazer se falhar:** delete o código comentado. Se precisar de referência histórica, use `git log`.

---

## CL3 — Sem abstrações especulativas

**Regra:** nenhuma abstração (interface, classe base, factory, adapter, configuração parametrizável) pode existir para suportar casos de uso que não existem ainda como requisito explícito na unidade de trabalho.

**Por quê:** YAGNI — You Ain't Gonna Need It. Abstrações prematuras aumentam complexidade sem entregar valor. Quando o caso de uso real aparecer, a abstração especulativa geralmente está errada de qualquer forma.

**Como checar:** para cada abstração encontrada, pergunte: existe um requisito na unidade de trabalho (ticket, SPEC ou o que foi acordado no grill) que justifica esta generalização hoje? Se não, é especulativa.

```
// ❌ especulativo — só tem um provider hoje
interface PaymentProvider { ... }
class StripeProvider implements PaymentProvider { ... }
class PaypalProvider implements PaymentProvider { ... } // não existe requisito

// ✅ direto — quando o segundo provider aparecer, extrai a interface
function chargeWithStripe(amount, card) { ... }
```

---

## CL4 — Nomes expressivos em todos os identificadores

**Regra:** nenhuma variável, função, classe, ou arquivo pode ter nome genérico que não descreve o que representa ou faz.

**Nomes proibidos sem qualificador:** `data`, `info`, `temp`, `result`, `obj`, `val`, `item`, `thing`, `stuff`, `misc`, `util` (como nome de arquivo ou módulo inteiro), `helper` (idem), `manager`, `handler` (sem qualificador do domínio).

**Como checar:** leia cada identificador isoladamente. Sem o contexto do código ao redor, você consegue dizer o que ele representa? Se não, o nome está errado.

```
// ❌
const data = await fetchUser(id);
function processInfo(obj) { ... }

// ✅
const user = await fetchUser(id);
function formatUserDisplayName(user: User) { ... }
```

---

## CL5 — Todo TODO/FIXME tem item de tech debt registrado

**Regra:** nenhum comentário `TODO`, `FIXME`, `HACK` ou `XXX` pode existir no código sem uma issue correspondente no tracker.

**Formato obrigatório:**
```ts
// TODO(#42): [descrição curta do que falta]
```

**Por quê:** TODO sem número de issue desaparece. A issue é a garantia de que o débito foi conscientemente aceito, não esquecido.

**O que fazer se falhar:** abra a issue e referencie o número, ou delete o TODO se não for mais relevante. No caminho pós-grill, que roda sem tracker, um TODO novo é achado — se merece TODO, merece issue.

---

## CL6 — Mudança estrutural acompanhada no `architecture.md`

**Regra:** se o diff criou módulo novo, criou camada nova, moveu um seam de teste, mudou uma fronteira que o documento lista (ex: servidor/cliente) ou mudou um invariante declarado, `docs/agents/architecture.md` tem que aparecer no mesmo diff.

**Por quê:** o `architecture.md` é lido por `to-spec`, `to-tickets`, `implementer` e pelo próprio revisor — é o que evita quatro etapas relerem o repo inteiro. Documento vivo sem fiscal apodrece sozinho: o dono é o `implementer`, e quem cobra é você. Sem isso, a posse existe no papel e ninguém a exerce.

**Como checar** — por leitura do diff, sem executar nada: liste os arquivos criados em diretório de módulo ou em diretório novo, e os que cruzaram uma fronteira listada no `architecture.md`. Para cada um, procure `docs/agents/architecture.md` na lista de arquivos do diff. Ausente = achado.

**Severidade: WARN.** O conserto é cirúrgico — uma linha na tabela "Onde cada coisa mora" — e não trava o loop.

**Não é achado:** arquivo novo que é só mais uma instância de um padrão já documentado (uma rota, uma página); teste novo num seam que já existe; edição de arquivo existente que não muda camada, seam nem invariante.

---

# Complexidade (C)

## C1 — Tamanho de função

**Regra:** nenhuma função ou método pode ter mais de 20 linhas de lógica (excluindo linhas em branco e comentários).

**Por quê:** funções longas fazem mais de uma coisa. Se não cabe em 20 linhas, tem responsabilidade demais.

**Como checar** — por leitura do diff, sem executar nada: apenas funções **adicionadas ou modificadas pelo diff** estão em escopo; função pré-existente que o diff só tangenciou não é achado desta review. Conte as linhas de corpo da função no arquivo final (o lado `+` do diff, mais o contexto não alterado da mesma função), excluindo linhas em branco, comentários, assinatura e chave de fechamento.

Só reporte quando a violação for inequívoca — 24 linhas contra o limite de 20 é achado; 21 contra 20 é ruído de contagem, não reporte. Funções que violam devem ser extraídas em funções menores com nomes expressivos.

**Exceção permitida:** funções de orquestração pura (que só chamam outras funções em sequência, sem lógica condicional própria) podem ter até 30 linhas. Documente com comentário: `// orchestration — sem lógica de negócio`.

---

## C2 — Tamanho de arquivo

**Regra:** nenhum arquivo de produção pode ter mais de 300 linhas.

**Por quê:** arquivo grande é sintoma de múltiplas responsabilidades no mesmo lugar.

**Como checar** — por leitura, sem executar nada: só arquivos criados pelo diff, ou cujo diff **cruzou** o limite (estava abaixo, passou), são achado. Arquivo que já estava acima de 300 linhas antes do diff é dívida pré-existente — registre como NIT, não como bloqueio deste ticket. Se ultrapassar, extraia módulos por responsabilidade.

**Exceção permitida:** arquivos de configuração, schemas de banco de dados, e arquivos gerados automaticamente estão isentos.

---

## C3 — Profundidade de aninhamento

**Regra:** nenhum bloco de código pode ter mais de 3 níveis de aninhamento (`if`, `for`, `while`, `try`, closures, etc.).

**Por quê:** aninhamento profundo é complexidade ciclomática acumulada — difícil de ler, difícil de testar, fácil de quebrar.

**Como checar:** inspecione visualmente. Se ultrapassar 3 níveis, use early return, extração de função, ou inversão de condicional.

```
// ❌ 4 níveis
if (a) {
  for (b) {
    if (c) {
      try { ... }   // nível 4
    }
  }
}

// ✅ early return + extração
if (!a) return;
for (b) {
  processItem(c);
}
```

---

## C4 — Complexidade ciclomática por função

**Regra:** nenhuma função pode ter complexidade ciclomática acima de 5.

**Como calcular:** comece com 1. Some +1 para cada: `if`, `else if`, `else`, `for`, `while`, `do`, `case`, `&&`, `||`, `??`, `?.` com lógica condicional, `catch`.

**Por quê:** complexidade ciclomática acima de 5 aumenta exponencialmente o número de caminhos de teste necessários para cobertura real.

**O que fazer se falhar:** extraia blocos condicionais em funções com nomes que descrevem a decisão.

---

## C5 — Profundidade de abstração

**Regra:** no máximo 2 camadas de abstração entre o ponto de entrada (rota, controller, handler) e a lógica de negócio real.

**Por quê:** abstrações em excesso escondem o que o código realmente faz. Toda camada que não adiciona comportamento adiciona apenas complexidade de navegação.

**Exemplo permitido:**
```
handler → service → repository
```

**Violação:**
```
handler → service → adapter → transformer → util → helper
```

**Como checar:** trace o caminho de chamada de qualquer endpoint ou função pública. Se precisar de mais de 2 saltos para chegar à lógica real, a abstração é especulativa.

---

## C6 — Parâmetros de função

**Regra:** nenhuma função pode ter mais de 3 parâmetros posicionais.

**Por quê:** mais de 3 parâmetros indica que a função está fazendo coisas demais ou que os dados deveriam ser agrupados em um objeto com semântica própria.

**O que fazer se falhar:** agrupe parâmetros relacionados em um objeto tipado com nome expressivo.

```
// ❌
function createUser(name, email, role, tenantId, plan) {}

// ✅
function createUser(user: CreateUserInput) {}
```

---

## C1 + C6 em conjunto — padrão obrigatório

Ao extrair uma função auxiliar para satisfazer C1 (reduzir linhas), é tentador passar os parâmetros avulsos para a nova função — o que viola C6. **O padrão correto é criar o dataclass no ponto de chamada, não esconder os parâmetros no helper.**

```ts
// ❌ extrai helper mas viola C6 (4 params posicionais)
function buildContext(session: Session, cohortId: string, slug: string, locale: string) {}

// ✅ define o tipo e passa como objeto único
type RenderContext = {
  session: Session;
  cohortId: string;
  slug: string;
  locale: string;
};

function buildContext(ctx: RenderContext) {}   // 1 parâmetro — C6 ✅
                                               // ≤ 20 linhas — C1 ✅
```

**Regra derivada:** funções auxiliares privadas criadas para satisfazer C1 **não são isentas** de C6. A solução é sempre o objeto tipado, nunca o repasse de parâmetros avulsos.

---

# Code smells (SM)

O eixo **C** mede tamanho: 20 linhas, 300 linhas, 3 níveis, complexidade 5, 3 parâmetros. Dá para escrever macarronada inteira respeitando todas. Este eixo mede **estrutura** — como as responsabilidades estão distribuídas, não quantas linhas ocupam.

Evidência de que o buraco é real: a seção "C1 + C6 em conjunto", acima, já documenta o remédio para Data Clumps (agrupar campos que viajam juntos num tipo) — mas ele só dispara quando a função passa de 3 parâmetros. Três campos viajando juntos por todo o código nunca disparam nada. O conserto estava documentado; a detecção, não.

## Disciplina deste eixo — leia antes de reportar

1. **Todo smell é julgamento, nunca violação dura.** Escreva "possível Feature Envy", não "viola Feature Envy". Um smell é um convite a olhar, não um veredito.
2. **Padrão documentado do repo vence o smell.** Se `docs/agents/architecture.md` ou o documento de design do projeto estabelecem o padrão que você ia apontar, não é achado — é a arquitetura, e questioná-la é assunto de outra conversa, não desta review.
3. **Pule o que a ferramenta já pega.** `lint` e `typecheck` rodaram no gate. Não reporte o que eles reportariam.
4. **Severidade: NIT, todos os dez.** Inclusive SM1. NIT vai para o corpo do PR e morre lá — **NIT não vira ticket**, não há limiar nem contador. O motivo é evitar churn automático em cima de julgamento: um achado que é uma opinião defensável não deve gerar trabalho sozinho.

> **Consequência aceita:** sem ferramenta de detecção de duplicação, SM1 só é pego quando você repara — e, como NIT, não é consertado automaticamente. É a opção mais barata, e é reversível.

## SM1 — Código duplicado

Mesma estrutura de lógica aparecendo em dois ou mais lugares do diff, ou no diff e num arquivo que ele toca. Não é texto idêntico — é a **mesma decisão** escrita duas vezes: quando uma mudar, alguém vai esquecer a outra. Conserto: extrair e chamar dos dois lados.

## SM2 — Feature Envy

Função que usa mais os dados de outro objeto/módulo do que os próprios. Sinal: uma cadeia de `outro.a`, `outro.b`, `outro.c` dentro de um método que quase não toca em si mesmo. Conserto: mover o método para onde os dados moram.

## SM3 — Message Chains

`a.b().c().d()` — o chamador precisa conhecer a estrutura interna de três objetos para fazer uma coisa. Qualquer um deles muda e o chamador quebra. Conserto: pedir o resultado direto a quem sabe produzi-lo.

## SM4 — Middle Man

Classe ou módulo cuja maioria dos métodos só delega para outro. A camada não adiciona comportamento, só navegação. Conserto: chamar o destino direto. (Parente do C5, mas C5 conta saltos e este julga se o salto faz algo.)

## SM5 — Data Clumps

O mesmo grupo de campos viajando junto em vários lugares — parâmetros, propriedades, retornos. Se três valores sempre aparecem juntos, eles são um conceito que ainda não tem nome. Conserto: dar o nome, criar o tipo.

## SM6 — Primitive Obsession

`string` para um id, um e-mail, um slug e um estado de turma — todos o mesmo tipo, nenhuma validação, e nada impede passar um no lugar do outro. Conserto: tipo com semântica própria, validado numa fronteira só.

## SM7 — Repeated Switches

O mesmo `switch`/cadeia de `if` sobre o mesmo valor, repetido em lugares diferentes. Adicionar um caso novo exige achar todos. Conserto: uma tabela, um mapa, ou polimorfismo — um lugar só que conhece os casos.

## SM8 — Shotgun Surgery

Uma mudança conceitual pequena espalhando edições por muitos arquivos. Se o diff toca oito arquivos para fazer uma coisa só, o comportamento está fragmentado. Conserto: juntar o que muda junto.

## SM9 — Divergent Change

O inverso do SM8: um arquivo que é editado por motivos não relacionados entre si. Se o mesmo módulo muda quando o layout muda **e** quando a regra de negócio muda, são duas responsabilidades no mesmo lugar. Conserto: separar por eixo de mudança.

## SM10 — Refused Bequest

Subclasse ou implementação que herda/implementa uma interface e ignora, lança, ou deixa vazia boa parte do contrato. Sinal de que a relação de herança está errada. Conserto: composição no lugar de herança, ou uma interface mais estreita.
