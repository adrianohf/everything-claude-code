
# O Guia Rápido para Everything Claude Code

![Cabeçalho: Vencedor do Hackathon da Anthropic - Dicas e Truques para o Claude Code](./assets/images/shortform/00-header.png)

---

**Sou um usuário ávido do Claude Code desde o lançamento experimental em fevereiro e ganhei o hackathon Anthropic x Forum Ventures com [zenith.chat](https://zenith.chat) ao lado de [@DRodriguezFX](https://x.com/DRodriguezFX) - usando completamente o Claude Code.**

Aqui está minha configuração completa após 10 meses de uso diário: skills, hooks, subagentes, MCPs, plugins e o que realmente funciona.

---

## Skills e Comandos

As skills operam como regras, restritas a certos escopos e fluxos de trabalho. São atalhos para prompts quando você precisa executar um fluxo de trabalho específico.

Depois de uma longa sessão de codificação com o Opus 4.5, você quer limpar código morto e arquivos .md soltos? Execute `/refactor-clean`. Precisa de testes? `/tdd`, `/e2e`, `/test-coverage`. As skills também podem incluir codemaps - uma maneira para o Claude navegar rapidamente em sua base de código sem queimar contexto na exploração.

![Terminal mostrando comandos encadeados](./assets/images/shortform/02-chaining-commands.jpeg)
*Encadeando comandos*

Comandos são skills executadas via comandos de barra. Eles se sobrepõem, mas são armazenados de forma diferente:

- **Skills**: `~/.claude/skills/` - definições de fluxo de trabalho mais amplas
- **Comandos**: `~/.claude/commands/` - prompts executáveis rápidos

```bash
# Estrutura de exemplo de skill
~/.claude/skills/
  pmx-guidelines.md      # Padrões específicos do projeto
  coding-standards.md    # Melhores práticas da linguagem
  tdd-workflow/          # Skill de vários arquivos com README.md
  security-review/       # Skill baseada em checklist
```

---

## Hooks

Hooks são automações baseadas em gatilhos que disparam em eventos específicos. Ao contrário das skills, eles são restritos a chamadas de ferramentas e eventos de ciclo de vida.

**Tipos de Hooks:**

1. **PreToolUse** - Antes da execução de uma ferramenta (validação, lembretes)
2. **PostToolUse** - Após a conclusão de uma ferramenta (formatação, ciclos de feedback)
3. **UserPromptSubmit** - Quando você envia uma mensagem
4. **Stop** - Quando o Claude termina de responder
5. **PreCompact** - Antes da compactação do contexto
6. **Notification** - Solicitações de permissão

**Exemplo: lembrete do tmux antes de comandos de longa execução**

```json
{
  "PreToolUse": [
    {
      "matcher": "tool == "Bash" && tool_input.command matches "(npm|pnpm|yarn|cargo|pytest)"",
      "hooks": [
        {
          "type": "command",
          "command": "if [ -z "$TMUX" ]; then echo '[Hook] Considere o tmux para persistência da sessão' >&2; fi"
        }
      ]
    }
  ]
}
```

![Feedback do hook PostToolUse](./assets/images/shortform/03-posttooluse-hook.png)
*Exemplo do feedback que você recebe no Claude Code, ao executar um hook PostToolUse*

**Dica profissional:** Use o plugin `hookify` para criar hooks conversacionalmente em vez de escrever JSON manualmente. Execute `/hookify` e descreva o que você quer.

---

## Subagentes

Subagentes são processos aos quais seu orquestrador (Claude principal) pode delegar tarefas com escopos limitados. Eles podem ser executados em segundo ou primeiro plano, liberando contexto para o agente principal.

Os subagentes funcionam bem com as skills - um subagente capaz de executar um subconjunto de suas skills pode receber tarefas e usar essas skills autonomamente. Eles também podem ser colocados em sandbox com permissões de ferramentas específicas.

```bash
# Estrutura de exemplo de subagente
~/.claude/agents/
  planner.md           # Planejamento de implementação de recursos
  architect.md         # Decisões de design de sistema
  tdd-guide.md         # Desenvolvimento orientado a testes
  code-reviewer.md     # Revisão de qualidade/segurança
  security-reviewer.md # Análise de vulnerabilidades
  build-error-resolver.md
  e2e-runner.md
  refactor-cleaner.md
```

Configure as ferramentas permitidas, MCPs e permissões por subagente para um escopo adequado.

---

## Regras e Memória

Sua pasta `.rules` contém arquivos `.md` com as melhores práticas que o Claude deve SEMPRE seguir. Duas abordagens:

1. **Único CLAUDE.md** - Tudo em um arquivo (nível de usuário ou projeto)
2. **Pasta de regras** - Arquivos `.md` modulares agrupados por preocupação

```bash
~/.claude/rules/
  security.md      # Sem segredos codificados, valide entradas
  coding-style.md  # Imutabilidade, organização de arquivos
  testing.md       # Fluxo de trabalho TDD, cobertura de 80%
  git-workflow.md  # Formato de commit, processo de PR
  agents.md        # Quando delegar para subagentes
  performance.md   # Seleção de modelo, gerenciamento de contexto
```

**Regras de exemplo:**

- Sem emojis na base de código
- Abster-se de tons roxos no frontend
- Sempre teste o código antes da implantação
- Priorize código modular em vez de mega-arquivos
- Nunca comite console.logs

---

## MCPs (Protocolo de Contexto de Modelo)

Os MCPs conectam o Claude a serviços externos diretamente. Não é um substituto para APIs - é um invólucro orientado por prompt em torno delas, permitindo mais flexibilidade na navegação de informações.

**Exemplo:** O MCP do Supabase permite que o Claude puxe dados específicos, execute SQL diretamente no upstream sem copiar e colar. O mesmo para bancos de dados, plataformas de implantação, etc.

![MCP do Supabase listando tabelas](./assets/images/shortform/04-supabase-mcp.jpeg)
*Exemplo do MCP do Supabase listando as tabelas dentro do esquema público*

**Chrome no Claude:** é um plugin MCP integrado que permite ao Claude controlar autonomamente seu navegador - clicando para ver como as coisas funcionam.

**CRÍTICO: Gerenciamento da Janela de Contexto**

Seja exigente com os MCPs. Eu mantenho todos os MCPs na configuração do usuário, mas **desabilito tudo o que não é usado**. Navegue até `/plugins` e role para baixo ou execute `/mcp`.

![Interface /plugins](./assets/images/shortform/05-plugins-interface.jpeg)
*Usando /plugins para navegar até os MCPs para ver quais estão atualmente instalados e seu status*

Sua janela de contexto de 200k antes da compactação pode ser de apenas 70k com muitas ferramentas habilitadas. O desempenho se degrada significativamente.

**Regra geral:** Tenha 20-30 MCPs na configuração, mas mantenha menos de 10 habilitados / menos de 80 ferramentas ativas.

```bash
# Verifique os MCPs habilitados
/mcp

# Desabilite os não utilizados em ~/.claude.json em projects.disabledMcpServers
```

---

## Plugins

Os plugins empacotam ferramentas para fácil instalação em vez de configuração manual tediosa. Um plugin pode ser uma skill + MCP combinados, ou hooks/ferramentas agrupados.

**Instalando plugins:**

```bash
# Adicione um marketplace
# plugin mgrep por @mixedbread-ai
claude plugin marketplace add https://github.com/mixedbread-ai/mgrep

# Abra o Claude, execute /plugins, encontre o novo marketplace, instale a partir daí
```

![Aba de Marketplaces mostrando mgrep](./assets/images/shortform/06-marketplaces-mgrep.jpeg)
*Exibindo o marketplace Mixedbread-Grep recém-instalado*

**Plugins LSP** são particularmente úteis se você executa o Claude Code fora dos editores com frequência. O Protocolo de Servidor de Linguagem dá ao Claude verificação de tipo em tempo real, ir para a definição e conclusões inteligentes sem precisar de um IDE aberto.

```bash
# Exemplo de plugins habilitados
typescript-lsp@claude-plugins-official  # Inteligência TypeScript
pyright-lsp@claude-plugins-official     # Verificação de tipo Python
hookify@claude-plugins-official         # Crie hooks conversacionalmente
mgrep@Mixedbread-Grep                   # Melhor pesquisa que o ripgrep
```

O mesmo aviso que para os MCPs - observe sua janela de contexto.

---

## Dicas e Truques

### Atalhos de Teclado

- `Ctrl+U` - Excluir linha inteira (mais rápido que spam de backspace)
- `!` - Prefixo rápido de comando bash
- `@` - Procurar arquivos
- `/` - Iniciar comandos de barra
- `Shift+Enter` - Entrada de várias linhas
- `Tab` - Alternar exibição de pensamento
- `Esc Esc` - Interromper o Claude / restaurar código

### Fluxos de Trabalho Paralelos

- **Fork** (`/fork`) - Bifurque conversas para fazer tarefas não sobrepostas em paralelo em vez de enviar mensagens em fila
- **Git Worktrees** - Para Claudes paralelos sobrepostos sem conflitos. Cada worktree é um checkout independente

```bash
git worktree add ../feature-branch feature-branch
# Agora execute instâncias separadas do Claude em cada worktree
```

### tmux para Comandos de Longa Execução

Transmita e observe logs/processos bash que o Claude executa:

<https://github.com/user-attachments/assets/shortform/07-tmux-video.mp4>

```bash
tmux new -s dev
# Claude executa comandos aqui, você pode desanexar e reanexar
tmux attach -t dev
```

### mgrep > grep

`mgrep` é uma melhoria significativa em relação ao ripgrep/grep. Instale via marketplace de plugins e, em seguida, use a skill `/mgrep`. Funciona tanto com pesquisa local quanto com pesquisa na web.

```bash
mgrep "function handleSubmit"  # Pesquisa local
mgrep --web "Alterações do roteador de aplicativos Next.js 15"  # Pesquisa na web
```

### Outros Comandos Úteis

- `/rewind` - Voltar para um estado anterior
- `/statusline` - Personalize com branch, % de contexto, todos
- `/checkpoints` - Pontos de desfazer no nível do arquivo
- `/compact` - Acionar manualmente a compactação do contexto

### CI/CD de Ações do GitHub

Configure a revisão de código em seus PRs com o GitHub Actions. O Claude pode revisar PRs automaticamente quando configurado.

![Bot do Claude aprovando um PR](./assets/images/shortform/08-github-pr-review.jpeg)
*Claude aprovando um PR de correção de bug*

### Sandboxing

Use o modo sandbox para operações arriscadas - o Claude é executado em um ambiente restrito sem afetar seu sistema real.

---

## Sobre Editores

Sua escolha de editor afeta significativamente o fluxo de trabalho do Claude Code. Embora o Claude Code funcione de qualquer terminal, combiná-lo com um editor capaz desbloqueia o rastreamento de arquivos em tempo real, a navegação rápida e a execução integrada de comandos.

### Zed (Minha Preferência)

Eu uso o [Zed](https://zed.dev) - escrito em Rust, então é genuinamente rápido. Abre instantaneamente, lida com bases de código massivas sem suar a camisa e mal toca nos recursos do sistema.

**Por que Zed + Claude Code é uma ótima combinação:**

- **Velocidade** - O desempenho baseado em Rust significa que não há atraso quando o Claude está editando arquivos rapidamente. Seu editor acompanha
- **Integração do Painel do Agente** - A integração do Claude no Zed permite que você acompanhe as alterações de arquivo em tempo real enquanto o Claude edita. Pule entre os arquivos que o Claude referencia sem sair do editor
- **Paleta de Comandos CMD+Shift+R** - Acesso rápido a todos os seus comandos de barra personalizados, depuradores, scripts de compilação em uma interface pesquisável
- **Uso Mínimo de Recursos** - Não competirá com o Claude por RAM/CPU durante operações pesadas. Importante ao executar o Opus
- **Modo Vim** - Atalhos de teclado completos do vim, se for a sua praia

![Editor Zed com comandos personalizados](./assets/images/shortform/09-zed-editor.jpeg)
*Editor Zed com menu suspenso de comandos personalizados usando CMD+Shift+R. O modo de seguimento é mostrado como o alvo no canto inferior direito.*

**Dicas Agnósticas de Editor:**

1. **Divida sua tela** - Terminal com o Claude Code de um lado, editor do outro
2. **Ctrl + G** - abra rapidamente o arquivo em que o Claude está trabalhando atualmente no Zed
3. **Salvar automaticamente** - Habilite o salvamento automático para que as leituras de arquivo do Claude estejam sempre atualizadas
4. **Integração com o Git** - Use os recursos do git do editor para revisar as alterações do Claude antes de comitar
5. **Observadores de arquivos** - A maioria dos editores recarrega automaticamente os arquivos alterados, verifique se isso está habilitado

### VSCode / Cursor

Esta também é uma escolha viável e funciona bem com o Claude Code. Você pode usá-lo em formato de terminal, com sincronização automática com seu editor usando `\ide` habilitando a funcionalidade LSP (um tanto redundante com os plugins agora). Ou você pode optar pela extensão que é mais integrada ao Editor e tem uma interface de usuário correspondente.

![Extensão do Claude Code para VS Code](./assets/images/shortform/10-vscode-extension.jpeg)
*A extensão do VS Code fornece uma interface gráfica nativa para o Claude Code, integrada diretamente ao seu IDE.*

---

## Minha Configuração

### Plugins

**Instalados:** (Geralmente, tenho apenas 4-5 deles habilitados por vez)

```markdown
ralph-wiggum@claude-code-plugins       # Automação de loop
frontend-design@claude-code-plugins    # Padrões de UI/UX
commit-commands@claude-code-plugins    # Fluxo de trabalho Git
security-guidance@claude-code-plugins  # Verificações de segurança
pr-review-toolkit@claude-code-plugins  # Automação de PR
typescript-lsp@claude-plugins-official # Inteligência TS
hookify@claude-plugins-official        # Criação de hook
code-simplifier@claude-plugins-official
feature-dev@claude-code-plugins
explanatory-output-style@claude-code-plugins
code-review@claude-code-plugins
context7@claude-plugins-official       # Documentação ao vivo
pyright-lsp@claude-plugins-official    # Tipos Python
mgrep@Mixedbread-Grep                  # Melhor pesquisa
```

### Servidores MCP

**Configurados (Nível de Usuário):**

```json
{
  "github": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"] },
  "firecrawl": { "command": "npx", "args": ["-y", "firecrawl-mcp"] },
  "supabase": {
    "command": "npx",
    "args": ["-y", "@supabase/mcp-server-supabase@latest", "--project-ref=SEU_REF"]
  },
  "memory": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-memory"] },
  "sequential-thinking": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-sequential-thinking"]
  },
  "vercel": { "type": "http", "url": "https://mcp.vercel.com" },
  "railway": { "command": "npx", "args": ["-y", "@railway/mcp-server"] },
  "cloudflare-docs": { "type": "http", "url": "https://docs.mcp.cloudflare.com/mcp" },
  "cloudflare-workers-bindings": {
    "type": "http",
    "url": "https://bindings.mcp.cloudflare.com/mcp"
  },
  "clickhouse": { "type": "http", "url": "https://mcp.clickhouse.cloud/mcp" },
  "AbletonMCP": { "command": "uvx", "args": ["ableton-mcp"] },
  "magic": { "command": "npx", "args": ["-y", "@magicuidesign/mcp@latest"] }
}
```

Esta é a chave - tenho 14 MCPs configurados, mas apenas ~5-6 habilitados por projeto. Mantém a janela de contexto saudável.

### Hooks Chave

```json
{
  "PreToolUse": [
    { "matcher": "npm|pnpm|yarn|cargo|pytest", "hooks": ["lembrete tmux"] },
    { "matcher": "Write && .md file", "hooks": ["bloquear a menos que README/CLAUDE"] },
    { "matcher": "git push", "hooks": ["abrir editor para revisão"] }
  ],
  "PostToolUse": [
    { "matcher": "Edit && .ts/.tsx/.js/.jsx", "hooks": ["prettier --write"] },
    { "matcher": "Edit && .ts/.tsx", "hooks": ["tsc --noEmit"] },
    { "matcher": "Edit", "hooks": ["aviso de grep console.log"] }
  ],
  "Stop": [
    { "matcher": "*", "hooks": ["verificar arquivos modificados para console.log"] }
  ]
}
```

### Linha de Status Personalizada

Mostra usuário, diretório, branch do git com indicador de sujeira, % restante de contexto, modelo, hora e contagem de tarefas:

![Linha de status personalizada](./assets/images/shortform/11-statusline.jpeg)
*Exemplo de linha de status no meu diretório raiz do Mac*

```
affoon:~ ctx:65% Opus 4.5 19:52
▌▌ modo de plano ativado (shift+tab para alternar)
```

### Estrutura de Regras

```
~/.claude/rules/
  security.md      # Verificações de segurança obrigatórias
  coding-style.md  # Imutabilidade, limites de tamanho de arquivo
  testing.md       # TDD, cobertura de 80%
  git-workflow.md  # Commits convencionais
  agents.md        # Regras de delegação de subagente
  patterns.md      # Formatos de resposta da API
  performance.md   # Seleção de modelo (Haiku vs Sonnet vs Opus)
  hooks.md         # Documentação de hook
```

### Subagentes

```
~/.claude/agents/
  planner.md           # Desmembrar recursos
  architect.md         # Design de sistema
  tdd-guide.md         # Escreva testes primeiro
  code-reviewer.md     # Revisão de qualidade
  security-reviewer.md # Varredura de vulnerabilidades
  build-error-resolver.md
  e2e-runner.md        # Testes do Playwright
  refactor-cleaner.md  # Remoção de código morto
  doc-updater.md       # Manter documentos sincronizados
```

---

## Principais Conclusões

1. **Não complique demais** - trate a configuração como um ajuste fino, não como arquitetura
2. **A janela de contexto é preciosa** - desabilite MCPs e plugins não utilizados
3. **Execução paralela** - bifurque conversas, use git worktrees
4. **Automatize o repetitivo** - hooks para formatação, linting, lembretes
5. **Defina o escopo de seus subagentes** - ferramentas limitadas = execução focada

---

## Referências

- [Referência de Plugins](https://code.claude.com/docs/en/plugins-reference)
- [Documentação de Hooks](https://code.claude.com/docs/en/hooks)
- [Checkpointing](https://code.claude.com/docs/en/checkpointing)
- [Modo Interativo](https://code.claude.com/docs/en/interactive-mode)
- [Sistema de Memória](https://code.claude.com/docs/en/memory)
- [Subagentes](https://code.claude.com/docs/en/sub-agents)
- [Visão Geral do MCP](https://code.claude.com/docs/en/mcp-overview)

---

**Nota:** Este é um subconjunto de detalhes. Consulte o [Guia Completo](./the-longform-guide.md) para padrões avançados.

---

*Ganhei o hackathon Anthropic x Forum Ventures em Nova York construindo [zenith.chat](https://zenith.chat) com [@DRodriguezFX](https://x.com/DRodriguezFX)*
