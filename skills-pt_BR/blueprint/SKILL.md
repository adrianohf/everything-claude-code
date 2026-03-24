---
name: blueprint
description: >-
  Transforme um objetivo de uma linha em um plano de construção passo a passo para
  projetos de engenharia multissessão e multiagente. Cada etapa possui um
  resumo de contexto independente para que um novo agente possa executá-lo do zero.
  Inclui porta de revisão adversária, grafo de dependência, detecção de etapas paralelas,
  catálogo de anti-padrões e protocolo de mutação de plano.
  ACIONAR quando: o usuário solicitar um plano, blueprint ou roteiro (roadmap) para uma
  tarefa complexa de vários PRs, ou descrever um trabalho que precise de várias sessões.
  NÃO ACIONAR quando: a tarefa puder ser concluída em um único PR ou em menos
  de 3 chamadas de ferramenta, ou se o usuário disser "apenas faça".
origin: community
---

# Blueprint — Gerador de Plano de Construção

Transforme um objetivo de uma linha em um plano de construção passo a passo que qualquer agente de codificação possa executar do zero.

## Quando Usar

- Dividir uma funcionalidade grande em vários PRs com ordem de dependência clara
- Planejar uma refatoração ou migração que abranja várias sessões
- Coordenar fluxos de trabalho paralelos entre subagentes
- Qualquer tarefa em que a perda de contexto entre sessões causaria retrabalho

**Não use** para tarefas que possam ser concluídas em um único PR, em menos de 3 chamadas de ferramenta ou quando o usuário disser "apenas faça".

## Como Funciona

O Blueprint executa um pipeline de 5 fases:

1. **Pesquisa** — Verificações prévias (git, gh auth, remoto, branch padrão), em seguida, lê a estrutura do projeto, planos existentes e arquivos de memória para reunir contexto.
2. **Design** — Divide o objetivo em etapas do tamanho de um PR (geralmente de 3 a 12). Atribui arestas de dependência, ordenação paralela/serial, nível do modelo (mais forte vs padrão) e estratégia de rollback por etapa.
3. **Rascunho** — Escreve um arquivo de plano Markdown independente em `plans/`. Cada etapa inclui um resumo de contexto, lista de tarefas, comandos de verificação e critérios de saída — para que um novo agente possa executar qualquer etapa sem ler as anteriores.
4. **Revisão** — Delega a revisão adversária a um subagente de modelo mais forte (ex: Opus) contra um checklist e um catálogo de anti-padrões. Corrige todas as descobertas críticas antes de finalizar.
5. **Registro** — Salva o plano, atualiza o índice de memória e apresenta a contagem de etapas e o resumo de paralelismo ao usuário.

O Blueprint detecta a disponibilidade de git/gh automaticamente. Com git + GitHub CLI, ele gera planos completos de fluxo de trabalho de branch/PR/CI. Sem eles, ele alterna para o modo direto (edição no local, sem branches).

## Exemplos

### Uso básico

```
/blueprint myapp "migrar banco de dados para PostgreSQL"
```

Produz `plans/myapp-migrar-banco-de-dados-para-postgresql.md` com etapas como:
- Etapa 1: Adicionar driver PostgreSQL e configuração de conexão
- Etapa 2: Criar scripts de migração para cada tabela
- Etapa 3: Atualizar a camada de repositório para usar o novo driver
- Etapa 4: Adicionar testes de integração contra o PostgreSQL
- Etapa 5: Remover o código e a configuração do banco de dados antigo

### Projeto multiagente

```
/blueprint chatbot "extrair provedores de LLM para um sistema de plugins"
```

Produz um plano com etapas paralelas onde for possível (ex: "implementar plugin Anthropic" e "implementar plugin OpenAI" rodam em paralelo após a conclusão da etapa da interface do plugin), atribuições de nível de modelo (mais forte para a etapa de design da interface, padrão para a implementação) e invariantes verificados após cada etapa (ex: "todos os testes existentes passam", "nenhum import de provedor no core").

## Principais Recursos

- **Execução do zero (Cold-start)** — Cada etapa inclui um resumo de contexto independente. Nenhum contexto anterior é necessário.
- **Porta de revisão adversária** — Cada plano é revisado por um subagente de modelo mais forte contra um checklist que cobre completude, correção de dependência e detecção de anti-padrões.
- **Fluxo de trabalho de Branch/PR/CI** — Integrado em cada etapa. Degrada suavemente para o modo direto quando git/gh está ausente.
- **Detecção de etapas paralelas** — O grafo de dependência identifica etapas sem arquivos compartilhados ou dependências de saída.
- **Protocolo de mutação de plano** — As etapas podem ser divididas, inseridas, puladas, reordenadas ou abandonadas com protocolos formais e trilha de auditoria.
- **Risco zero em tempo de execução** — Skill de Markdown puro. Todo o repositório contém apenas arquivos `.md` — sem hooks, sem scripts de shell, sem código executável, sem `package.json`, sem etapa de build. Nada é executado na instalação ou invocação além do carregador de skills de Markdown nativo do Claude Code.

## Instalação

Esta skill vem com o Everything Claude Code. Nenhuma instalação separada é necessária quando o ECC está instalado.

### Instalação completa do ECC

Se você estiver trabalhando a partir do checkout do repositório ECC, verifique se a skill está presente com:

```bash
test -f skills/blueprint/SKILL.md
```

Para atualizar mais tarde, revise o diff do ECC antes de atualizar:

```bash
cd /path/to/everything-claude-code
git fetch origin main
git log --oneline HEAD..origin/main       # revise os novos commits antes de atualizar
git checkout <sha-completo-revisado>      # fixe em um commit revisado específico
```

### Instalação autônoma (Vendored)

Se você estiver distribuindo apenas esta skill fora da instalação completa do ECC, copie o arquivo revisado do repositório ECC para `~/.claude/skills/blueprint/SKILL.md`. Cópias distribuídas não têm um remoto git, portanto, atualize-as copiando novamente o arquivo de um commit revisado do ECC em vez de executar `git pull`.

## Requisitos

- Claude Code (para o comando slash `/blueprint`)
- Git + GitHub CLI (opcional — permite fluxo de trabalho completo de branch/PR/CI; o Blueprint detecta a ausência e alterna automaticamente para o modo direto)

## Fonte

Inspirado por antbotlab/blueprint — projeto upstream e design de referência.
