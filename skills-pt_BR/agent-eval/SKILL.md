---
name: agent-eval-pt-BR
description: Comparação lado a lado de agentes de codificação (Claude Code, Aider, Codex, etc.) em tarefas personalizadas com métricas de taxa de aprovação, custo, tempo e consistência.
origin: ECC
tools: Read, Write, Edit, Bash, Grep, Glob
---

# SKILL: Avaliação de Agente (Tradução)

Uma ferramenta CLI leve para comparar agentes de codificação lado a lado em tarefas reproduzíveis. Toda comparação "qual agente de codificação é o melhor?" é baseada em impressões — esta ferramenta sistematiza isso.

## Quando Ativar

- Comparando agentes de codificação (Claude Code, Aider, Codex, etc.) em sua própria base de código
- Medindo o desempenho do agente antes de adotar uma nova ferramenta ou modelo
- Executando verificações de regressão quando um agente atualiza seu modelo ou ferramentas
- Produzindo decisões de seleção de agentes baseadas em dados para uma equipe

## Instalação

> **Nota:** Instale o agent-eval a partir de seu repositório após revisar o código-fonte.

## Conceitos Principais

### Definições de Tarefas em YAML

Defina tarefas declarativamente. Cada tarefa especifica o que fazer, quais arquivos tocar e como julgar o sucesso:

```yaml
name: add-retry-logic
description: Add exponential backoff retry to the HTTP client
repo: ./my-project
files:
  - src/http_client.py
prompt: |
  Add retry logic with exponential backoff to all HTTP requests.
  Max 3 retries. Initial delay 1s, max delay 30s.
judge:
  - type: pytest
    command: pytest tests/test_http_client.py -v
  - type: grep
    pattern: "exponential_backoff|retry"
    files: src/http_client.py
commit: "abc1234"  # pin to specific commit for reproducibility
```

### Isolamento com Git Worktree

Cada execução de agente obtém seu próprio git worktree — não é necessário Docker. Isso fornece isolamento de reprodutibilidade para que os agentes não possam interferir uns com os outros ou corromper o repositório base.

### Métricas Coletadas

| Métrica | O Que Mede |
|---|---|
| Taxa de aprovação | O agente produziu código que passa no juiz? |
| Custo | Gasto de API por tarefa (quando disponível) |
| Tempo | Segundos de relógio para conclusão |
| Consistência | Taxa de aprovação em execuções repetidas (ex: 3/3 = 100%) |

## Fluxo de Trabalho

### 1. Definir Tarefas

Crie um diretório `tasks/` com arquivos YAML, um por tarefa:

```bash
mkdir tasks
# Write task definitions (see template above)
```

### 2. Executar Agentes

Execute agentes contra suas tarefas:

```bash
agent-eval run --task tasks/add-retry-logic.yaml --agent claude-code --agent aider --runs 3
```

Cada execução:
1. Cria um novo git worktree a partir do commit especificado
2. Entrega o prompt ao agente
3. Executa os critérios do juiz
4. Registra aprovação/falha, custo e tempo

### 3. Comparar Resultados

Gere um relatório de comparação:

```bash
agent-eval report --format table
```

```
Task: add-retry-logic (3 runs each)
┌──────────────┬───────────┬────────┬────────┬─────────────┐
│ Agent        │ Pass Rate │ Cost   │ Time   │ Consistency │
├──────────────┼───────────┼────────┼────────┼─────────────┤
│ claude-code  │ 3/3       │ $0.12  │ 45s    │ 100%        │
│ aider        │ 2/3       │ $0.08  │ 38s    │  67%        │
└──────────────┴───────────┴────────┴────────┴─────────────┘
```

## Tipos de Juiz

### Baseado em Código (determinístico)

```yaml
judge:
  - type: pytest
    command: pytest tests/ -v
  - type: command
    command: npm run build
```

### Baseado em Padrão

```yaml
judge:
  - type: grep
    pattern: "class.*Retry"
    files: src/**/*.py
```

### Baseado em Modelo (LLM-como-juiz)

```yaml
judge:
  - type: llm
    prompt: |
      Does this implementation correctly handle exponential backoff?
      Check for: max retries, increasing delays, jitter.
```

## Melhores Práticas

- **Comece com 3-5 tarefas** que representem sua carga de trabalho real, não exemplos de brinquedo
- **Execute pelo menos 3 tentativas** por agente para capturar a variação — agentes não são determinísticos
- **Fixe o commit** em seu YAML de tarefa para que os resultados sejam reproduzíveis ao longo de dias/semanas
- **Inclua pelo menos um juiz determinístico** (testes, build) por tarefa — juízes LLM adicionam ruído
- **Acompanhe o custo junto com a taxa de aprovação** — um agente de 95% com 10x o custo pode não ser a escolha certa
- **Versione suas definições de tarefa** — elas são acessórios de teste, trate-as como código

## Links

- Repositório: [github.com/joaquinhuigomez/agent-eval](https://github.com/joaquinhuigomez/agent-eval)
