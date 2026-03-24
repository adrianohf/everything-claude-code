# Contribuindo para o Everything Claude Code

Obrigado por querer contribuir! Este repositório é um recurso comunitário para usuários do Claude Code.

## Índice

- [O Que Estamos Buscando](#o-que-estamos-buscando)
- [Início Rápido](#início-rápido)
- [Contribuindo com Skills](#contribuindo-com-skills)
- [Contribuindo com Agentes](#contribuindo-com-agentes)
- [Contribuindo com Hooks](#contribuindo-com-hooks)
- [Contribuindo com Comandos](#contribuindo-com-comandos)
- [MCP e Documentação (ex: Context7)](#mcp-e-documentação-ex-context7)
- [Multiplataforma e Traduções](#multiplataforma-e-traduções)
- [Processo de Pull Request](#processo-de-pull-request)

---

## O Que Estamos Buscando

### Agentes
Novos agentes que lidam bem com tarefas específicas:
- Revisores específicos de linguagem (Python, Go, Rust)
- Especialistas em frameworks (Django, Rails, Laravel, Spring)
- Especialistas em DevOps (Kubernetes, Terraform, CI/CD)
- Especialistas de domínio (pipelines de ML, engenharia de dados, mobile)

### Skills
Definições de fluxo de trabalho e conhecimento de domínio:
- Melhores práticas de linguagem
- Padrões de frameworks
- Estratégias de testes
- Guias de arquitetura

### Hooks
Automações úteis:
- Hooks de lint/formatação
- Verificações de segurança
- Hooks de validação
- Hooks de notificação

### Comandos
Comandos slash que invocam fluxos de trabalho úteis:
- Comandos de implantação
- Comandos de teste
- Comandos de geração de código

---

## Início Rápido

```bash
# 1. Fork e clone
gh repo fork affaan-m/everything-claude-code --clone
cd everything-claude-code

# 2. Criar uma branch
git checkout -b feat/minha-contribuicao

# 3. Adicionar sua contribuição

# 4. Testar suas mudanças
node tests/run-all.js

# 5. Commitar
git commit -m "feat: adicionar minha contribuição"

# 6. Abrir PR
gh pr create
```

## Contribuindo com Skills

Skills são definições de fluxo de trabalho e conhecimento de domínio para Claude Code.

Cada skill deve incluir:

1. **Quando Usar**
2. **Como Funciona**
3. **Exemplos**

## Contribuindo com Agentes

Agentes são arquivos Markdown com frontmatter YAML descrevendo nome, propósito, ferramentas e modelo.

## Contribuindo com Hooks

Hooks automatizam verificações e ações baseadas em eventos.

- Mantenha compatibilidade multiplataforma
- Evite ações destrutivas automáticas
- Adicione testes quando houver lógica relevante

## Contribuindo com Comandos

Comandos são prompts executáveis via slash commands.

- Dê nome curto e descritivo
- Especifique entradas esperadas
- Deixe claro o resultado esperado

## MCP e Documentação (ex: Context7)

Contribuições relacionadas a MCP devem documentar caso de uso, dependências externas, configuração necessária e considerações de segurança.

## Multiplataforma e Traduções

- Teste em Windows/macOS/Linux quando aplicável
- Evite suposições específicas de shell
- Mantenha caminhos portáveis
- Se adicionar docs, considere impacto em traduções

## Processo de Pull Request

Antes de abrir o PR:

- Rode os testes relevantes
- Revise mudanças de docs
- Verifique se não expôs segredos
- Confirme que a contribuição segue os padrões do projeto
