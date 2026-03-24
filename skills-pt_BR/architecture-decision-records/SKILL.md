---
name: architecture-decision-records
description: Capture decisões arquiteturais tomadas durante sessões do Claude Code como ADRs (Architecture Decision Records) estruturados. Detecta automaticamente momentos de decisão, registra o contexto, as alternativas consideradas e a justificativa. Mantém um log de ADR para que futuros desenvolvedores entendam por que a base de código tem o formato que tem.
origin: ECC
---

# Registros de Decisões de Arquitetura (ADR)

Capture decisões arquiteturais conforme elas acontecem durante as sessões de codificação. Em vez de as decisões viverem apenas em threads do Slack, comentários de PR ou na memória de alguém, esta skill produz documentos ADR estruturados que vivem ao lado do código.

## Quando Ativar

- O usuário diz explicitamente "vamos registrar esta decisão" ou "faça um ADR disso"
- O usuário escolhe entre alternativas significativas (framework, biblioteca, padrão, banco de dados, design de API)
- O usuário diz "decidimos..." ou "o motivo de estarmos fazendo X em vez de Y é..."
- O usuário pergunta "por que escolhemos X?" (leia os ADRs existentes)
- Durante as fases de planejamento, quando as compensações (trade-offs) arquiteturais são discutidas

## Formato de ADR

Use o formato ADR leve proposto por Michael Nygard, adaptado para o desenvolvimento assistido por IA:

```markdown
# ADR-NNNN: [Título da Decisão]

**Data**: AAAA-MM-DD
**Status**: proposto | aceito | depreciado | substituído pelo ADR-NNNN
**Decisores**: [quem esteve envolvido]

## Contexto

Qual é o problema que estamos vendo que está motivando esta decisão ou mudança?

[2 a 5 sentenças descrevendo a situação, restrições e forças em jogo]

## Decisão

Qual é a mudança que estamos propondo e/ou fazendo?

[1 a 3 sentenças declarando a decisão claramente]

## Alternativas Consideradas

### Alternativa 1: [Nome]
- **Prós**: [benefícios]
- **Contras**: [desvantagens]
- **Por que não**: [razão específica pela qual isso foi rejeitado]

### Alternativa 2: [Nome]
- **Prós**: [benefícios]
- **Contras**: [desvantagens]
- **Por que não**: [razão específica pela qual isso foi rejeitado]

## Consequências

O que se torna mais fácil ou mais difícil de fazer por causa desta mudança?

### Positivas
- [benefício 1]
- [benefício 2]

### Negativas
- [compensação (trade-off) 1]
- [compensação (trade-off) 2]

### Riscos
- [risco e mitigação]
```

## Fluxo de Trabalho

### Capturando um novo ADR

Quando um momento de decisão é detectado:

1. **Inicializar (apenas na primeira vez)** — se `docs/adr/` não existir, peça confirmação ao usuário antes de criar o diretório, um `README.md` semeado com o cabeçalho da tabela de índice (veja o Formato do Índice ADR abaixo) e um `template.md` em branco para uso manual. Não crie arquivos sem consentimento explícito.
2. **Identificar a decisão** — extrair a escolha arquitetural central que está sendo feita.
3. **Reunir contexto** — qual problema motivou isso? Quais restrições existem?
4. **Documentar alternativas** — quais outras opções foram consideradas? Por que foram rejeitadas?
5. **Declarar consequências** — quais são as compensações (trade-offs)? O que se torna mais fácil/difícil?
6. **Atribuir um número** — analisar os ADRs existentes em `docs/adr/` e incrementar.
7. **Confirmar e escrever** — apresentar o rascunho do ADR ao usuário para revisão. Escrever em `docs/adr/NNNN-titulo-da-decisao.md` apenas após aprovação explícita. Se o usuário recusar, descarte o rascunho sem escrever nenhum arquivo.
8. **Atualizar o índice** — anexar ao `docs/adr/README.md`.

### Lendo ADRs Existentes

Quando um usuário pergunta "por que escolhemos X?":

1. Verificar se `docs/adr/` existe — se não, responder: "Nenhum ADR encontrado neste projeto. Gostaria de começar a registrar as decisões arquiteturais?"
2. Se existir, analisar o índice `docs/adr/README.md` em busca de entradas relevantes.
3. Ler os arquivos ADR correspondentes e apresentar as seções Contexto e Decisão.
4. Se nenhuma correspondência for encontrada, responder: "Nenhum ADR encontrado para essa decisão. Gostaria de registrar um agora?"

### Estrutura de Diretório ADR

```
docs/
└── adr/
    ├── README.md              ← índice de todos os ADRs
    ├── 0001-use-nextjs.md
    ├── 0002-postgres-over-mongo.md
    ├── 0003-rest-over-graphql.md
    └── template.md            ← modelo em branco para uso manual
```

### Formato do Índice ADR

```markdown
# Registros de Decisões de Arquitetura

| ADR | Título | Status | Data |
|-----|-------|--------|------|
| [0001](0001-use-nextjs.md) | Usar Next.js como framework de frontend | aceito | 2026-01-15 |
| [0002](0002-postgres-over-mongo.md) | PostgreSQL em vez de MongoDB para armazenamento de dados principal | aceito | 2026-01-20 |
| [0003](0003-rest-over-graphql.md) | API REST em vez de GraphQL | aceito | 2026-02-01 |
```

## Sinais de Detecção de Decisão

Fique atento a estes padrões na conversa que indicam uma decisão arquitetural:

**Sinais explícitos**
- "Vamos usar X"
- "Devemos usar X em vez de Y"
- "A compensação (trade-off) vale a pena porque..."
- "Registre isso como um ADR"

**Sinais implícitos** (sugerir o registro de um ADR — não crie automaticamente sem confirmação do usuário)
- Comparar dois frameworks ou bibliotecas e chegar a uma conclusão.
- Fazer uma escolha de design de esquema de banco de dados com justificativa declarada.
- Escolher entre padrões arquiteturais (monólito vs microserviços, REST vs GraphQL).
- Decidir sobre a estratégia de autenticação/autorização.
- Selecionar a infraestrutura de implantação após avaliar alternativas.

## O que faz um bom ADR

### Fazer (Do)
- **Seja específico** — "Usar Prisma ORM" em vez de "usar um ORM".
- **Registre o porquê** — a justificativa importa mais do que o quê.
- **Inclua alternativas rejeitadas** — futuros desenvolvedores precisam saber o que foi considerado.
- **Declare as consequências honestamente** — toda decisão tem compensações (trade-offs).
- **Mantenha-o curto** — um ADR deve ser lido em 2 minutos.
- **Use o tempo presente** — "Usamos X" em vez de "Usaremos X".

### Não Fazer (Don't)
- Registrar decisões triviais — escolhas de nomenclatura de variáveis ou formatação não precisam de ADRs.
- Escrever ensaios — se a seção de contexto exceder 10 linhas, está longa demais.
- Omitir alternativas — "nós apenas escolhemos" não é uma justificativa válida.
- Preencher retroativamente sem marcar — se estiver registrando uma decisão passada, anote a data original.
- Deixar os ADRs ficarem obsoletos — decisões substituídas devem referenciar sua substituição.

## Ciclo de Vida do ADR

```
proposto → aceito → [depreciado | substituído pelo ADR-NNNN]
```

- **proposto**: a decisão está em discussão, ainda não confirmada.
- **aceito**: a decisão está em vigor e sendo seguida.
- **depreciado**: a decisão não é mais relevante (ex: recurso removido).
- **substituído**: um ADR mais novo substitui este (sempre vincule a substituição).

## Categorias de Decisões que Vale a Pena Registrar

| Categoria | Exemplos |
|----------|---------|
| **Escolhas de tecnologia** | Framework, linguagem, banco de dados, provedor de nuvem |
| **Padrões de arquitetura** | Monólito vs microserviços, orientado a eventos, CQRS |
| **Design de API** | REST vs GraphQL, estratégia de versionamento, mecanismo de autenticação |
| **Modelagem de dados** | Design de esquema, decisões de normalização, estratégia de cache |
| **Infraestrutura** | Modelo de implantação, pipeline de CI/CD, stack de monitoramento |
| **Segurança** | Estratégia de autenticação, abordagem de criptografia, gerenciamento de segredos |
| **Testes** | Framework de testes, metas de cobertura, equilíbrio E2E vs integração |
| **Processo** | Estratégia de ramificação (branching), processo de revisão, cadência de lançamento |

## Integração com Outras Skills

- **Agente de Planejamento (Planner)**: quando o planejador propuser mudanças na arquitetura, sugira a criação de um ADR.
- **Agente de Revisão de Código (Code Reviewer)**: sinalizar PRs que introduzem mudanças arquiteturais sem um ADR correspondente.
