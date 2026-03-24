---
name: autonomous-loops
description: "Padrões e arquiteturas para loops autônomos do Claude Code — desde pipelines sequenciais simples até sistemas DAG multiagentes orientados por RFC."
origin: ECC
---

# Skill de Loops Autônomos

> Nota de compatibilidade (v1.8.0): `autonomous-loops` é mantida por uma versão.
> O nome canônico da skill agora é `continuous-agent-loop`. Novas orientações sobre loops
> devem ser criadas lá, enquanto esta skill permanece disponível para evitar a quebra
> de fluxos de trabalho existentes.

Padrões, arquiteturas e implementações de referência para executar o Claude Code de forma autônoma em loops. Abrange tudo, desde pipelines simples `claude -p` até orquestração DAG multiagente completa orientada por RFC.

## Quando Usar

- Configuração de fluxos de trabalho de desenvolvimento autônomos que rodam sem intervenção humana
- Escolha da arquitetura de loop correta para o seu problema (simples vs complexa)
- Construção de pipelines de desenvolvimento contínuo no estilo CI/CD
- Execução de agentes paralelos com coordenação de mesclagem (merge)
- Implementação de persistência de contexto entre as iterações do loop
- Adição de portões de qualidade (quality gates) e passagens de limpeza a fluxos de trabalho autônomos

## Espectro de Padrões de Loop

Do mais simples ao mais sofisticado:

| Padrão | Complexidade | Melhor Para |
|---------|-----------|----------|
| [Pipeline Sequencial](#1-pipeline-sequencial-claude--p) | Baixa | Passos de desenvolvimento diários, fluxos de trabalho roteirizados |
| [NanoClaw REPL](#2-nanoclaw-repl) | Baixa | Sessões persistentes interativas |
| [Loop Agêntico Infinito](#3-loop-agentico-infinito) | Média | Geração de conteúdo paralela, trabalho orientado por especificações |
| [Loop de PR Contínuo do Claude](#4-loop-de-pr-continuo-do-claude) | Média | Projetos iterativos de vários dias com portões de CI |
| [Padrão De-Sloppify (Limpeza)](#5-o-padrao-de-sloppify) | Adicional | Limpeza de qualidade após qualquer etapa de Implementação |
| [Ralphinho / Orquestração DAG Orientada por RFC](#6-ralphinho--orquestracao-dag-orientada-por-rfc) | Alta | Grandes funcionalidades, trabalho paralelo em várias unidades com fila de mesclagem |

---

## 1. Pipeline Sequencial (`claude -p`)

**O loop mais simples.** Divida o desenvolvimento diário em uma sequência de chamadas `claude -p` não interativas. Cada chamada é uma etapa focada com um prompt claro.

### Insight Principal

> Se você não consegue entender um loop como este, significa que você nem consegue conduzir o LLM para corrigir seu código no modo interativo.

A flag `claude -p` executa o Claude Code de forma não interativa com um prompt e sai quando termina. Encadeie chamadas para construir um pipeline:

```bash
#!/bin/bash
# daily-dev.sh — Pipeline sequencial para uma branch de funcionalidade

set -e

# Passo 1: Implementar a funcionalidade
claude -p "Leia a especificação em docs/auth-spec.md. Implemente o login OAuth2 em src/auth/. Escreva os testes primeiro (TDD). NÃO crie novos arquivos de documentação."

# Passo 2: De-sloppify (passagem de limpeza)
claude -p "Revise todos os arquivos alterados pelo commit anterior. Remova quaisquer testes de tipo desnecessários, verificações excessivamente defensivas ou testes de recursos da linguagem (ex: testar se os genéricos do TypeScript funcionam). Mantenha os testes de lógica de negócio reais. Execute a suíte de testes após a limpeza."

# Passo 3: Verificar
claude -p "Execute o build completo, lint, verificação de tipo e a suíte de testes. Corrija quaisquer falhas. Não adicione novas funcionalidades."

# Passo 4: Commit
claude -p "Crie um commit convencional para todas as alterações preparadas. Use 'feat: add OAuth2 login flow' como mensagem."
```

### Princípios de Design Fundamentais

1. **Cada etapa é isolada** — Uma janela de contexto nova por chamada `claude -p` significa que não há vazamento de contexto entre as etapas.
2. **A ordem importa** — As etapas são executadas sequencialmente. Cada uma constrói sobre o estado do sistema de arquivos deixado pela anterior.
3. **Instruções negativas são perigosas** — Não diga "não teste sistemas de tipo". Em vez disso, adicione uma etapa de limpeza separada (veja [Padrão De-Sloppify](#5-o-padrao-de-sloppify)).
4. **Os códigos de saída se propagam** — `set -e` interrompe o pipeline em caso de falha.

### Variações

**Com roteamento de modelo:**
```bash
# Pesquisa com Opus (raciocínio profundo)
claude -p --model opus "Analise a arquitetura da base de código e escreva um plano para adicionar cache..."

# Implementação com Sonnet (rápido, capaz)
claude -p "Implemente a camada de cache de acordo com o plano em docs/caching-plan.md..."

# Revisão com Opus (minuciosa)
claude -p --model opus "Revise todas as alterações em busca de problemas de segurança, condições de corrida e casos extremos..."
```

**Com contexto de ambiente:**
```bash
# Passe o contexto via arquivos, não pelo comprimento do prompt
echo "Áreas de foco: módulo de autenticação, limitação de taxa da API" > .claude-context.md
claude -p "Leia .claude-context.md para prioridades. Trabalhe nelas em ordem."
rm .claude-context.md
```

**Com restrições `--allowedTools`:**
```bash
# Passagem de análise apenas de leitura
claude -p --allowedTools "Read,Grep,Glob" "Realize uma auditoria nesta base de código em busca de vulnerabilidades de segurança..."

# Passagem de implementação apenas de escrita
claude -p --allowedTools "Read,Write,Edit,Bash" "Implemente as correções de security-audit.md..."
```

---

## 2. NanoClaw REPL

**Loop persistente integrado do ECC.** Um REPL ciente da sessão que chama `claude -p` sincronicamente com o histórico completo da conversa.

```bash
# Iniciar a sessão padrão
node scripts/claw.js

# Sessão nomeada com contexto de skill
CLAW_SESSION=meu-projeto CLAW_SKILLS=tdd-workflow,security-review node scripts/claw.js
```

### Como Funciona

1. Carrega o histórico da conversa de `~/.claude/claw/{session}.md`
2. Cada mensagem do usuário é enviada para `claude -p` com o histórico completo como contexto
3. As respostas são anexadas ao arquivo da sessão (Markdown como banco de dados)
4. As sessões persistem entre as reinicializações

### Quando usar NanoClaw vs Pipeline Sequencial

| Caso de Uso | NanoClaw | Pipeline Sequencial |
|----------|----------|-------------------|
| Exploração interativa | Sim | Não |
| Automação roteirizada | Não | Sim |
| Persistência de sessão | Integrada | Manual |
| Acúmulo de contexto | Cresce por turno | Novo em cada etapa |
| Integração CI/CD | Ruim | Excelente |

Veja a documentação do comando `/claw` para detalhes completos.

---

## 3. Loop Agêntico Infinito

**Um sistema de dois prompts** que orquestra subagentes paralelos para geração baseada em especificações. Desenvolvido por disler (crédito: @disler).

### Arquitetura: Sistema de Dois Prompts

```
PROMPT 1 (Orquestrador)              PROMPT 2 (Subagentes)
┌─────────────────────┐             ┌──────────────────────┐
│ Analisar arq. espec  │             │ Receber contexto total│
│ Escanear dir. saída  │  implanta   │ Ler número atribuído  │
│ Planejar iteração    │────────────│ Seguir espec. à risca │
│ Atribuir dirs criat. │  N agentes  │ Gerar saída única     │
│ Gerenciar ondas      │             │ Salvar no dir. saída  │
└─────────────────────┘             └──────────────────────┘
```

### O Padrão

1. **Análise da Espec** — O orquestrador lê um arquivo de especificação (Markdown) definindo o que gerar.
2. **Reconhecimento de Diretório** — Escaneia a saída existente para encontrar o número de iteração mais alto.
3. **Implantação Paralela** — Lança N subagentes, cada um com:
   - A especificação completa
   - Uma direção criativa única
   - Um número de iteração específico (sem conflitos)
   - Um snapshot das iterações existentes (para garantir a unicidade)
4. **Gerenciamento de Ondas** — Para o modo infinito, implanta ondas de 3 a 5 agentes até que o contexto se esgote.

### Implementação via Comandos do Claude Code

Crie `.claude/commands/infinite.md`:

```markdown
Analise os seguintes argumentos de $ARGUMENTS:
1. spec_file — caminho para o markdown de especificação
2. output_dir — onde as iterações são salvas
3. count — inteiro 1-N ou "infinite"

FASE 1: Ler e entender profundamente a especificação.
FASE 2: Listar output_dir, encontrar o número de iteração mais alto. Começar em N+1.
FASE 3: Planejar direções criativas — cada agente recebe um tema/abordagem DIFERENTE.
FASE 4: Implantar subagentes em paralelo (ferramenta Task). Cada um recebe:
  - Texto completo da especificação
  - Snapshot do diretório atual
  - Seu número de iteração atribuído
  - Sua direção criativa única
FASE 5 (modo infinito): Loop em ondas de 3-5 até que o contexto esteja baixo.
```

**Invocar:**
```bash
/project:infinite specs/component-spec.md src/ 5
/project:infinite specs/component-spec.md src/ infinite
```

### Estratégia de Lote (Batching)

| Contagem | Estratégia |
|-------|----------|
| 1-5 | Todos os agentes simultaneamente |
| 6-20 | Lotes de 5 |
| infinite | Ondas de 3-5, sofisticação progressiva |

### Insight Principal: Unicidade via Atribuição

Não confie nos agentes para se diferenciarem sozinhos. O orquestrador **atribui** a cada agente uma direção criativa específica e um número de iteração. Isso evita conceitos duplicados entre agentes paralelos.

---

## 4. Loop de PR Contínuo do Claude

**Um script shell de nível de produção** que executa o Claude Code em um loop contínuo, criando PRs, aguardando o CI e mesclando automaticamente. Criado por AnandChowdhary (crédito: @AnandChowdhary).

### Loop Central

```
┌─────────────────────────────────────────────────────┐
│  ITERAÇÃO CONTÍNUA DO CLAUDE                        │
│                                                     │
│  1. Criar branch (continuous-claude/iteration-N)    │
│  2. Rodar claude -p com prompt aprimorado           │
│  3. (Opcional) Passagem de revisor — claude -p sep. │
│  4. Commitar mudanças (Claude gera a mensagem)      │
│  5. Push + criar PR (gh pr create)                  │
│  6. Aguardar checks de CI (poll gh pr checks)       │
│  7. Falha no CI? → Passagem de auto-correção        │
│  8. Mesclar PR (squash/merge/rebase)                │
│  9. Retornar para a main → repetir                  │
│                                                     │
│  Limitar por: --max-runs N | --max-cost $X          │
│            --max-duration 2h | sinal de conclusão   │
└─────────────────────────────────────────────────────┘
```

### Instalação

> **Aviso:** Instale o continuous-claude a partir do seu repositório após revisar o código. Não direcione scripts externos diretamente para o bash.

### Uso

```bash
# Básico: 10 iterações
continuous-claude --prompt "Adicione testes unitários para todas as funções não testadas" --max-runs 10

# Limitado por custo
continuous-claude --prompt "Corrija todos os erros do linter" --max-cost 5.00

# Limitado por tempo
continuous-claude --prompt "Melhore a cobertura de testes" --max-duration 8h

# Com passagem de revisão de código
continuous-claude \
  --prompt "Adicione funcionalidade de autenticação" \
  --max-runs 10 \
  --review-prompt "Execute npm test && npm run lint, corrija quaisquer falhas"

# Paralelo via worktrees
continuous-claude --prompt "Adicione testes" --max-runs 5 --worktree tests-worker &
continuous-claude --prompt "Refatore o código" --max-runs 5 --worktree refactor-worker &
wait
```

### Contexto Entre Iterações: SHARED_TASK_NOTES.md

A inovação crítica: um arquivo `SHARED_TASK_NOTES.md` que persiste entre as iterações:

```markdown
## Progresso
- [x] Adicionados testes para o módulo de autenticação (iteração 1)
- [x] Corrigido caso extremo na renovação do token (iteração 2)
- [ ] Ainda necessário: testes de limitação de taxa, testes de error boundary

## Próximos Passos
- Focar no módulo de limitação de taxa em seguida
- A configuração de mock em tests/helpers.ts pode ser reutilizada
```

O Claude lê este arquivo no início da iteração e o atualiza no final. Isso preenche a lacuna de contexto entre as invocações independentes de `claude -p`.

### Recuperação de Falha no CI

Quando os checks de PR falham, o Continuous Claude automaticamente:
1. Busca o ID da execução que falhou via `gh run list`
2. Inicia um novo `claude -p` com o contexto da correção do CI
3. O Claude inspeciona os logs via `gh run view`, corrige o código, commita e faz o push
4. Aguarda novamente pelos checks (até o limite de tentativas de `--ci-retry-max`)

### Sinal de Conclusão

O Claude pode sinalizar "Terminei" emitindo uma frase mágica:

```bash
continuous-claude \
  --prompt "Corrija todos os bugs no rastreador de problemas" \
  --completion-signal "CONTINUOUS_CLAUDE_PROJECT_COMPLETE" \
  --completion-threshold 3  # Para após 3 sinais consecutivos
```

Três iterações consecutivas sinalizando a conclusão interrompem o loop, evitando execuções desperdiçadas em trabalho finalizado.

### Configuração Principal

| Flag | Propósito |
|------|---------|
| `--max-runs N` | Parar após N iterações bem-sucedidas |
| `--max-cost $X` | Parar após gastar $X |
| `--max-duration 2h` | Parar após o tempo decorrido |
| `--merge-strategy squash` | squash, merge ou rebase |
| `--worktree <nome>` | Execução paralela via worktrees do git |
| `--disable-commits` | Modo de simulação (sem operações de git) |
| `--review-prompt "..."` | Adicionar passagem de revisor por iteração |
| `--ci-retry-max N` | Auto-corrigir falhas de CI (padrão: 1) |

---

## 5. O Padrão De-Sloppify

**Um padrão adicional para qualquer loop.** Adicione uma etapa dedicada de limpeza/refatoração após cada etapa de Implementação.

### O Problema

Quando você pede para um LLM implementar com TDD, ele leva o "escrever testes" muito ao pé da letra:
- Testes que verificam se o sistema de tipos do TypeScript funciona (testando `typeof x === 'string'`)
- Verificações de runtime excessivamente defensivas para coisas que o sistema de tipos já garante
- Testes para o comportamento do framework em vez da lógica de negócio
- Tratamento de erros excessivo que obscurece o código real

### Por que não usar Instruções Negativas?

Adicionar "não teste sistemas de tipos" ou "não adicione verificações desnecessárias" ao prompt do Implementador tem efeitos colaterais:
- O modelo torna-se hesitante sobre TODOS os testes
- Ele pula testes legítimos de casos extremos
- A qualidade degrada de forma imprevisível

### A Solução: Passagem Separada

Em vez de restringir o Implementador, deixe-o ser minucioso. Em seguida, adicione um agente de limpeza focado:

```bash
# Passo 1: Implementar (deixe-o ser minucioso)
claude -p "Implemente a funcionalidade com TDD completo. Seja minucioso nos testes."

# Passo 2: De-sloppify (contexto separado, limpeza focada)
claude -p "Revise todas as alterações na árvore de trabalho. Remova:
- Testes que verificam o comportamento da linguagem/framework em vez da lógica de negócio
- Verificações de tipo redundantes que o sistema de tipos já impõe
- Tratamento de erros excessivamente defensivo para estados impossíveis
- Instruções console.log
- Código comentado

Mantenha todos os testes de lógica de negócio. Execute a suíte de testes após a limpeza para garantir que nada quebre."
```

### Em um Contexto de Loop

```bash
for feature in "${features[@]}"; do
  # Implementar
  claude -p "Implemente $feature com TDD."

  # De-sloppify
  claude -p "Passagem de limpeza: revise as alterações, remova o lixo de teste/código, execute os testes."

  # Verificar
  claude -p "Execute build + lint + testes. Corrija quaisquer falhas."

  # Commit
  claude -p "Commit com mensagem: feat: add $feature"
done
```

### Insight Principal

> Em vez de adicionar instruções negativas que têm efeitos colaterais na qualidade, adicione uma passagem de limpeza (de-sloppify) separada. Dois agentes focados superam um agente restrito.

---

## 6. Ralphinho / Orquestração DAG Orientada por RFC

**O padrão mais sofisticado.** Um pipeline multiagente orientado por RFC que decompõe uma especificação em um DAG de dependências, executa cada unidade através de um pipeline de qualidade em camadas e as entrega via uma fila de mesclagem (merge queue) orientada por agentes. Criado por enitrat (crédito: @enitrat).

### Visão Geral da Arquitetura

```
Documento RFC/PRD
       │
       ▼
  DECOMPOSIÇÃO (IA)
  Dividir RFC em unidades de trabalho com DAG de dependências
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  LOOP RALPH (até 3 passagens)                        │
│                                                      │
│  Para cada camada do DAG (sequencial, por dep.):     │
│                                                      │
│  ┌── Pipelines de Qualidade (paralelos por unid.) ─┐  │
│  │  Cada unidade em seu próprio worktree:          │  │
│  │  Pesquisa → Plano → Implementar → Testar → Rev.│  │
│  │  (profundidade varia por nível de complexidade) │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌── Fila de Mesclagem (Merge Queue) ─────────────┐  │
│  │  Rebase na main → Rodar testes → Unir ou expul.│  │
│  │  Unidades expulsas reentram com cont. conflito  │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### Decomposição de RFC

A IA lê o RFC e produz as unidades de trabalho:

```typescript
interface WorkUnit {
  id: string;              // identificador em kebab-case
  name: string;            // Nome legível por humanos
  rfcSections: string[];   // Quais seções do RFC isso aborda
  description: string;     // Descrição detalhada
  deps: string[];          // Dependências (outros IDs de unidade)
  acceptance: string[];    // Critérios de aceitação concretos
  tier: "trivial" | "small" | "medium" | "large";
}
```

**Regras de Decomposição:**
- Preferir menos unidades, mais coesas (minimizar o risco de mesclagem)
- Minimizar a sobreposição de arquivos entre unidades (evitar conflitos)
- Manter os testes JUNTO com a implementação (nunca separe "implementar X" + "testar X")
- Dependências apenas onde existe dependência de código real

O DAG de dependências determina a ordem de execução:
```
Camada 0: [unidade-a, unidade-b]     ← sem deps, rodar em paralelo
Camada 1: [unidade-c]               ← depende da unidade-a
Camada 2: [unidade-d, unidade-e]     ← dependem da unidade-c
```

### Níveis de Complexidade

Níveis diferentes recebem profundidades de pipeline diferentes:

| Nível | Estágios do Pipeline |
|------|----------------|
| **trivial** | implementar → testar |
| **small** | implementar → testar → revisão de código |
| **medium** | pesquisa → plano → implementar → testar → revisão de PRD + revisão de código → correção de revisão |
| **large** | pesquisa → plano → implementar → testar → revisão de PRD + revisão de código → correção de revisão → revisão final |

Isso evita operações caras em mudanças simples, ao mesmo tempo em que garante que as mudanças arquiteturais recebam um escrutínio minucioso.

### Janelas de Contexto Separadas (Eliminação do Viés do Autor)

Cada estágio é executado em seu próprio processo de agente com sua própria janela de contexto:

| Estágio | Modelo | Propósito |
|-------|-------|---------|
| Pesquisa | Sonnet | Ler base de código + RFC, produzir doc de contexto |
| Plano | Opus | Projetar as etapas de implementação |
| Implementar | Codex | Escrever o código seguindo o plano |
| Testar | Sonnet | Executar build + suíte de testes |
| Revisão PRD | Sonnet | Verificação de conformidade com a especificação |
| Revisão Código | Opus | Verificação de qualidade + segurança |
| Corr. Revisão | Codex | Abordar os problemas da revisão |
| Revisão Final | Opus | Portão de qualidade (apenas nível grande) |

**Design crítico:** O revisor nunca escreveu o código que revisa. Isso elimina o viés do autor — a fonte mais comum de problemas ignorados na autorrevisão.

### Fila de Mesclagem com Expulsão (Eviction)

Após a conclusão dos pipelines de qualidade, as unidades entram na fila de mesclagem:

```
Branch da unidade
    │
    ├─ Rebase na main
    │   └─ Conflito? → EXPULSAR (capturar contexto do conflito)
    │
    ├─ Rodar build + testes
    │   └─ Falha? → EXPULSAR (capturar saída do teste)
    │
    └─ Sucesso → Fast-forward main, push, deletar branch
```

**Inteligência de Sobreposição de Arquivos:**
- Unidades que não se sobrepõem são mescladas especulativamente em paralelo.
- Unidades que se sobrepõem são mescladas uma a uma, fazendo rebase a cada vez.

**Recuperação de Expulsão:**
Quando expulsa, o contexto completo é capturado (arquivos em conflito, diffs, saída de teste) e enviado de volta ao implementador na próxima passagem Ralph:

```markdown
## CONFLITO DE MESCLAGEM — RESOLVA ANTES DA PRÓXIMA TENTATIVA

Sua implementação anterior entrou em conflito com outra unidade que foi mesclada primeiro.
Reestruture suas alterações para evitar os arquivos/linhas em conflito abaixo.

{contexto completo da expulsão com diffs}
```

### Fluxo de Dados entre Estágios

```
pesquisa.contextFilePath ──────────────────→ plano
plano.implementationSteps ──────────────────→ implementar
implementar.{filesCreated, whatWasDone} ─────→ testar, revisões
testar.failingSummary ───────────────────────→ revisões, implementar (próx. pass)
revisões.{feedback, issues} ────────────────→ corr. revisão → implementar (próx. pass)
revisão-final.reasoning ────────────────────→ implementar (próx. pass)
evictionContext ───────────────────────────→ implementar (após conflito mesclagem)
```

### Isolamento por Worktree

Cada unidade é executada em um worktree isolado (usa jj/Jujutsu, não git):
```
/tmp/workflow-wt-{id-da-unidade}/
```

Os estágios do pipeline para a mesma unidade **compartilham** um worktree, preservando o estado (arquivos de contexto, arquivos de plano, alterações de código) ao longo de pesquisa → plano → implementar → testar → revisão.

### Princípios de Design Fundamentais

1. **Execução determinística** — A decomposição inicial define o paralelismo e a ordenação.
2. **Revisão humana em pontos de alavancagem** — O plano de trabalho é o ponto de intervenção de maior alavancagem.
3. **Separação de preocupações** — Cada estágio em uma janela de contexto separada com um agente separado.
4. **Recuperação de conflitos com contexto** — O contexto completo de expulsão permite re-execuções inteligentes, não tentativas cegas.
5. **Profundidade orientada por nível** — Mudanças triviais pulam a pesquisa/revisão; mudanças grandes recebem escrutínio máximo.
6. **Fluxos de trabalho retomáveis** — Estado completo persistido no SQLite; retome de qualquer ponto.

### Quando usar Ralphinho vs Padrões Mais Simples

| Sinal | Use Ralphinho | Use Padrão Mais Simples |
|--------|--------------|-------------------|
| Várias unidades de trabalho interdependentes | Sim | Não |
| Necessidade de implementação paralela | Sim | Não |
| Conflitos de mesclagem prováveis | Sim | Não (sequencial serve) |
| Mudança em arquivo único | No | Sim (pipeline sequencial) |
| Projeto de vários dias | Sim | Talvez (continuous-claude) |
| Especificação/RFC já escrita | Sim | Talvez |
| Iteração rápida em uma coisa | No | Sim (NanoClaw ou pipeline) |

---

## Escolhendo o Padrão Correto

### Matriz de Decisão

```
A tarefa é uma mudança focada e única?
├─ Sim → Pipeline Sequencial ou NanoClaw
└─ Não → Existe uma especificação/RFC escrita?
         ├─ Sim → Você precisa de implementação paralela?
         │        ├─ Sim → Ralphinho (orquestração DAG)
         │        └─ Não → Continuous Claude (loop iterativo de PR)
         └─ Não → Você precisa de muitas variações da mesma coisa?
                  ├─ Sim → Loop Agêntico Infinito (geração orientada por espec.)
                  └─ Não → Pipeline Sequencial com de-sloppify
```

### Combinando Padrões

Estes padrões combinam bem:

1. **Pipeline Sequencial + De-Sloppify** — A combinação mais comum. Cada etapa de implementação recebe uma passagem de limpeza.

2. **Continuous Claude + De-Sloppify** — Adicione `--review-prompt` com uma diretiva de de-sloppify a cada iteração.

3. **Qualquer loop + Verificação** — Use o comando `/verify` do ECC ou a skill `verification-loop` como um portão antes dos commits.

4. **Abordagem em camadas do Ralphinho em loops simples** — Mesmo em um pipeline sequencial, você pode direcionar tarefas simples para o Haiku e tarefas complexas para o Opus:
   ```bash
   # Correção de formatação simples
   claude -p --model haiku "Corrija a ordem dos imports em src/utils.ts"

   # Mudança arquitetural complexa
   claude -p --model opus "Refatore o módulo de autenticação para usar o padrão strategy"
   ```

---

## Anti-padrões

### Erros Comuns

1. **Loops infinitos sem condições de saída** — Sempre tenha um max-runs, max-cost, max-duration ou sinal de conclusão.

2. **Nenhuma ponte de contexto entre iterações** — Cada chamada `claude -p` começa do zero. Use `SHARED_TASK_NOTES.md` ou o estado do sistema de arquivos para fazer a ponte de contexto.

3. **Repetir a mesma falha** — Se uma iteração falhar, não apenas tente novamente. Capture o contexto do erro e envie-o para a próxima tentativa.

4. **Instruções negativas em vez de passagens de limpeza** — Não diga "não faça X". Adicione uma passagem separada que remova X.

5. **Todos os agentes em uma única janela de contexto** — Para fluxos de trabalho complexos, separe as preocupações em processos de agentes diferentes. O revisor nunca deve ser o autor.

6. **Ignorar a sobreposição de arquivos no trabalho paralelo** — Se dois agentes paralelos puderem editar o mesmo arquivo, você precisa de uma estratégia de mesclagem (entrega sequencial, rebase ou resolução de conflitos).

---

## Referências

| Projeto | Autor | Link |
|---------|--------|------|
| Ralphinho | enitrat | crédito: @enitrat |
| Loop Agêntico Infinito | disler | crédito: @disler |
| Continuous Claude | AnandChowdhary | crédito: @AnandChowdhary |
| NanoClaw | ECC | comando `/claw` neste repositório |
| Loop de Verificação | ECC | `skills/verification-loop/` neste repositório |
