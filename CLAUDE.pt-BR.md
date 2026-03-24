# CLAUDE.md

Este arquivo fornece orientações ao Claude Code (`claude.ai/code`) ao trabalhar com código neste repositório.

## Visão Geral do Projeto

Este é um **plugin para Claude Code**: uma coleção de agentes, skills, hooks, comandos, regras e configurações MCP prontos para produção. O projeto fornece fluxos de trabalho testados em batalha para desenvolvimento de software com Claude Code.

## Executando Testes

```bash
# Executar todos os testes
node tests/run-all.js

# Executar arquivos de teste individuais
node tests/lib/utils.test.js
node tests/lib/package-manager.test.js
node tests/hooks/hooks.test.js
```

## Arquitetura

O projeto está organizado em vários componentes centrais:

- **agents/** - Subagentes especializados para delegação (`planner`, `code-reviewer`, `tdd-guide` etc.)
- **skills/** - Definições de fluxo de trabalho e conhecimento de domínio (padrões de código, padrões, testes)
- **commands/** - Comandos slash invocados por usuários (`/tdd`, `/plan`, `/e2e` etc.)
- **hooks/** - Automações baseadas em gatilhos (persistência de sessão, hooks pré/pós-ferramenta)
- **rules/** - Diretrizes que devem ser sempre seguidas (segurança, estilo de código, requisitos de teste)
- **mcp-configs/** - Configurações de servidores MCP para integrações externas
- **scripts/** - Utilitários Node.js multiplataforma para hooks e setup
- **tests/** - Suíte de testes para scripts e utilitários

## Comandos Principais

- `/tdd` - Fluxo de desenvolvimento orientado a testes
- `/plan` - Planejamento de implementação
- `/e2e` - Gerar e executar testes E2E
- `/code-review` - Revisão de qualidade
- `/build-fix` - Corrigir erros de build
- `/learn` - Extrair padrões de sessões
- `/skill-create` - Gerar skills a partir do histórico do git

## Notas de Desenvolvimento

- Detecção de gerenciador de pacotes: npm, pnpm, yarn, bun (configurável via variável de ambiente `CLAUDE_PACKAGE_MANAGER` ou config do projeto)
- Multiplataforma: suporte a Windows, macOS e Linux via scripts Node.js
- Formato de agente: Markdown com frontmatter YAML (`name`, `description`, `tools`, `model`)
- Formato de skill: Markdown com seções claras para quando usar, como funciona e exemplos
- Local de skills: curadas em `skills/`; geradas/importadas em `~/.claude/skills/`. Veja `docs/SKILL-PLACEMENT-POLICY.md`
- Formato de hook: JSON com condições `matcher` e hooks de comando/notificação

## Contribuindo

Siga os formatos em `CONTRIBUTING.md`:
- Agentes: Markdown com frontmatter (`name`, `description`, `tools`, `model`)
- Skills: seções claras (Quando Usar, Como Funciona, Exemplos)
- Comandos: Markdown com frontmatter de descrição
- Hooks: JSON com `matcher` e array de hooks

Nomeação de arquivos: minúsculas com hífens (por exemplo, `python-reviewer.md`, `tdd-workflow.md`)
