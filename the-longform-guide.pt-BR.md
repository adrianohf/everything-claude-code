
# O Guia Completo para Everything Claude Code

![Cabeçalho: O Guia Completo para Everything Claude Code](./assets/images/longform/01-header.png)

---

> **Pré-requisito**: Este guia se baseia em [O Guia Rápido para Everything Claude Code](./the-shortform-guide.md). Leia-o primeiro se você ainda não configurou skills, hooks, subagentes, MCPs e plugins.

![Referência ao Guia Rápido](./assets/images/longform/02-shortform-reference.png)
*O Guia Rápido - leia primeiro*

No guia rápido, abordei a configuração fundamental: skills e comandos, hooks, subagentes, MCPs, plugins e os padrões de configuração que formam a espinha dorsal de um fluxo de trabalho eficaz com Claude Code. Aquele foi o guia de configuração e a infraestrutura base.

Este guia completo aprofunda as técnicas que separam sessões produtivas das desperdiçadas. Se você não leu o guia rápido, volte e configure seus arquivos primeiro. O que se segue assume que você já tem skills, agentes, hooks e MCPs configurados e funcionando.

Os temas aqui são: economia de tokens, persistência de memória, padrões de verificação, estratégias de paralelização e os efeitos compostos da construção de fluxos de trabalho reutilizáveis. Estes são os padrões que refinei ao longo de mais de 10 meses de uso diário que fazem a diferença entre ser atormentado pela perda de contexto na primeira hora versus manter sessões produtivas por horas.

Tudo o que foi abordado nos guias rápido e completo está disponível no GitHub: `github.com/affaan-m/everything-claude-code`

---

## Dicas e Truques

### Alguns MCPs são Substituíveis e Liberarão sua Janela de Contexto

Para MCPs como controle de versão (GitHub), bancos de dados (Supabase), implantação (Vercel, Railway) etc. - a maioria dessas plataformas já possui CLIs robustas que o MCP está essencialmente apenas envolvendo. O MCP é um bom invólucro, mas tem um custo.

Para fazer a CLI funcionar mais como um MCP sem realmente usar o MCP (e a janela de contexto reduzida que o acompanha), considere agrupar a funcionalidade em skills e comandos. Remova as ferramentas que o MCP expõe que facilitam as coisas e transforme-as em comandos.

Exemplo: em vez de ter o MCP do GitHub carregado o tempo todo, crie um comando `/gh-pr` que envolva `gh pr create` com suas opções preferidas. Em vez do MCP do Supabase consumindo contexto, crie skills que usem a CLI do Supabase diretamente.

Com o carregamento preguiçoso (lazy loading), o problema da janela de contexto está praticamente resolvido. Mas o uso e o custo dos tokens não são resolvidos da mesma maneira. A abordagem de CLI + skills ainda é um método de otimização de tokens.

---

## COISAS IMPORTANTES

### Gerenciamento de Contexto e Memória

Para compartilhar memória entre sessões, uma skill ou comando que resume e verifica o progresso e depois salva em um arquivo `.tmp` em sua pasta `.claude` e o anexa até o final da sua sessão é a melhor aposta. No dia seguinte, ele pode usar isso como contexto e continuar de onde você parou, crie um novo arquivo para cada sessão para não poluir o contexto antigo em um novo trabalho.

![Árvore de Arquivos de Armazenamento de Sessão](./assets/images/longform/03-session-storage.png)
*Exemplo de armazenamento de sessão -> <https://github.com/affaan-m/everything-claude-code/tree/main/examples/sessions>*

Claude cria um arquivo resumindo o estado atual. Revise-o, peça edições se necessário e comece de novo. Para a nova conversa, basta fornecer o caminho do arquivo. Particularmente útil quando você está atingindo os limites de contexto e precisa continuar um trabalho complexo. Esses arquivos devem conter:
- Quais abordagens funcionaram (verificavelmente com evidências)
- Quais abordagens foram tentadas, mas não funcionaram
- Quais abordagens não foram tentadas e o que falta fazer

**Limpando o Contexto Estrategicamente:**

Depois de ter seu plano definido e o contexto limpo (opção padrão no modo de plano no Claude Code agora), você pode trabalhar a partir do plano. Isso é útil quando você acumulou muito contexto de exploração que não é mais relevante para a execução. Para compactação estratégica, desative a compactação automática. Compacte manualmente em intervalos lógicos ou crie uma skill que faça isso por você.

**Avançado: Injeção Dinâmica de Prompt de Sistema**

Um padrão que aprendi: em vez de colocar tudo apenas no `CLAUDE.md` (escopo do usuário) ou em `.claude/rules/` (escopo do projeto), que carrega a cada sessão, use flags de CLI para injetar contexto dinamicamente.

```bash
claude --system-prompt "$(cat memory.md)"
```

Isso permite que você seja mais cirúrgico sobre qual contexto carregar e quando. O conteúdo do prompt do sistema tem maior autoridade do que as mensagens do usuário, que têm maior autoridade do que os resultados das ferramentas.

**Configuração prática:**

```bash
# Desenvolvimento diário
alias claude-dev='claude --system-prompt "$(cat ~/.claude/contexts/dev.md)"'

# Modo de revisão de PR
alias claude-review='claude --system-prompt "$(cat ~/.claude/contexts/review.md)"'

# Modo de pesquisa/exploração
alias claude-research='claude --system-prompt "$(cat ~/.claude/contexts/research.md)"'
```

**Avançado: Hooks de Persistência de Memória**

Existem hooks que a maioria das pessoas não conhece que ajudam com a memória:

- **Hook PreCompact**: Antes que a compactação de contexto aconteça, salve o estado importante em um arquivo
- **Hook Stop (Fim da Sessão)**: No final da sessão, persista os aprendizados em um arquivo
- **Hook SessionStart**: Em uma nova sessão, carregue o contexto anterior automaticamente

Eu construí esses hooks e eles estão no repositório em `github.com/affaan-m/everything-claude-code/tree/main/hooks/memory-persistence`

---

### Aprendizado Contínuo / Memória

Se você teve que repetir um prompt várias vezes e Claude encontrou o mesmo problema ou deu uma resposta que você já ouviu antes - esses padrões devem ser anexados às skills.

**O Problema:** Tokens desperdiçados, contexto desperdiçado, tempo desperdiçado.

**A Solução:** Quando o Claude Code descobre algo que não é trivial - uma técnica de depuração, uma solução alternativa, algum padrão específico do projeto - ele salva esse conhecimento como uma nova skill. Na próxima vez que um problema semelhante surgir, a skill será carregada automaticamente.

Eu construí uma skill de aprendizado contínuo que faz isso: `github.com/affaan-m/everything-claude-code/tree/main/skills/continuous-learning`

**Por que o Hook Stop (e não o UserPromptSubmit):**

A principal decisão de design é usar um hook **Stop** em vez de UserPromptSubmit. O UserPromptSubmit é executado em cada mensagem - adiciona latência a cada prompt. O Stop é executado uma vez no final da sessão - leve, não o atrasa durante a sessão.

---

### Otimização de Tokens

**Estratégia Principal: Arquitetura de Subagentes**

Otimize as ferramentas que você usa e a arquitetura de subagentes projetada para delegar o modelo mais barato possível que seja suficiente para a tarefa.

**Referência Rápida de Seleção de Modelo:**

![Tabela de Seleção de Modelo](./assets/images/longform/04-model-selection.png)
*Configuração hipotética de subagentes em várias tarefas comuns e o raciocínio por trás das escolhas*

| Tipo de Tarefa            | Modelo | Por quê                                    |
| ------------------------- | ------ | ------------------------------------------ |
| Exploração/pesquisa       | Haiku  | Rápido, barato, bom o suficiente para encontrar arquivos |
| Edições simples           | Haiku  | Alterações em um único arquivo, instruções claras |
| Implementação multi-arquivo | Sonnet | Melhor equilíbrio para codificação          |
| Arquitetura complexa      | Opus   | Raciocínio profundo necessário             |
| Revisões de PR            | Sonnet | Entende o contexto, percebe nuances        |
| Análise de segurança      | Opus   | Não se pode dar ao luxo de perder vulnerabilidades |
| Escrever documentos       | Haiku  | A estrutura é simples                      |
| Depuração de bugs complexos | Opus   | Precisa ter todo o sistema em mente        |

Use o Sonnet como padrão para 90% das tarefas de codificação. Atualize para o Opus quando a primeira tentativa falhar, a tarefa abranger mais de 5 arquivos, decisões de arquitetura ou código crítico de segurança.

**Referência de Preços:**

![Preços do Modelo Claude](./assets/images/longform/05-pricing-table.png)
*Fonte: <https://platform.claude.com/docs/en/about-claude/pricing>*

**Otimizações Específicas de Ferramentas:**

Substitua o `grep` por `mgrep` - redução de tokens de ~50% em média em comparação com `grep` ou `ripgrep` tradicionais:

![Benchmark do mgrep](./assets/images/longform/06-mgrep-benchmark.png)
*Em nosso benchmark de 50 tarefas, o `mgrep` + Claude Code usou ~2x menos tokens do que os fluxos de trabalho baseados em `grep` com qualidade julgada semelhante ou melhor. Fonte: `mgrep` por @mixedbread-ai*

**Benefícios de uma Base de Código Modular:**

Ter uma base de código mais modular com os arquivos principais tendo centenas de linhas em vez de milhares ajuda tanto nos custos de otimização de tokens quanto na conclusão correta de uma tarefa na primeira tentativa.

---

### Loops de Verificação e Avaliações

**Fluxo de Trabalho de Benchmarking:**

Compare pedir a mesma coisa com e sem uma skill e verifique a diferença na saída:

Bifurque a conversa, inicie um novo `worktree` em uma delas sem a skill, faça um `diff` no final, veja o que foi registrado.

**Tipos de Padrões de Avaliação:**

- **Avaliações Baseadas em Checkpoint**: Defina checkpoints explícitos, verifique em relação a critérios definidos, corrija antes de prosseguir
- **Avaliações Contínuas**: Execute a cada N minutos ou após grandes alterações, suíte de testes completa + lint

**Métricas Chave:**

```
pass@k: Pelo menos UMA de k tentativas tem sucesso
        k=1: 70%  k=3: 91%  k=5: 97%

pass^k: TODAS as k tentativas devem ter sucesso
        k=1: 70%  k=3: 34%  k=5: 17%
```

Use **pass@k** quando você só precisa que funcione. Use **pass^k** quando a consistência é essencial.

---

## PARALELIZAÇÃO

Ao bifurcar conversas em uma configuração de terminal multi-Claude, certifique-se de que o escopo esteja bem definido para as ações na bifurcação e na conversa original. Procure uma sobreposição mínima quando se trata de alterações de código.

**Meu Padrão Preferido:**

Chat principal para alterações de código, bifurcações para perguntas sobre a base de código e seu estado atual, ou pesquisa sobre serviços externos.

**Sobre Contagens Arbitrárias de Terminais:**

![Boris sobre Terminais Paralelos](./assets/images/longform/07-boris-parallel.png)
*Boris (Anthropic) sobre a execução de várias instâncias do Claude*

Boris tem dicas sobre paralelização. Ele sugeriu coisas como executar 5 instâncias do Claude localmente e 5 upstream. Aconselho contra a definição de quantidades arbitrárias de terminais. A adição de um terminal deve ser por verdadeira necessidade.

Seu objetivo deve ser: **quanto você consegue fazer com a quantidade mínima viável de paralelização.**

**Git Worktrees para Instâncias Paralelas:**

```bash
# Crie worktrees para trabalho paralelo
git worktree add ../project-feature-a feature-a
git worktree add ../project-feature-b feature-b
git worktree add ../project-refactor refactor-branch

# Cada worktree recebe sua própria instância do Claude
cd ../project-feature-a && claude
```

SE você começar a escalar suas instâncias E tiver várias instâncias do Claude trabalhando em código que se sobrepõe, é imperativo que você use `git worktrees` e tenha um plano muito bem definido para cada um. Use `/rename <nome aqui>` para nomear todos os seus chats.

![Configuração de Dois Terminais](./assets/images/longform/08-two-terminals.png)
*Configuração Inicial: Terminal Esquerdo para Codificação, Terminal Direito para Perguntas - use `/rename` e `/fork`*

**O Método Cascata:**

Ao executar várias instâncias do Claude Code, organize com um padrão "cascata":

- Abra novas tarefas em novas abas à direita
- Varra da esquerda para a direita, do mais antigo para o mais novo
- Concentre-se em no máximo 3-4 tarefas por vez

---

## TRABALHO PREPARATÓRIO

**O Padrão de Kickoff de Duas Instâncias:**

Para o gerenciamento do meu próprio fluxo de trabalho, gosto de iniciar um repositório vazio com 2 instâncias abertas do Claude.

**Instância 1: Agente de Scaffolding**
- Cria o scaffold e o trabalho preparatório
- Cria a estrutura do projeto
- Configura os arquivos de configuração (CLAUDE.md, regras, agentes)

**Instância 2: Agente de Pesquisa Profunda**
- Conecta-se a todos os seus serviços, pesquisa na web
- Cria o PRD detalhado
- Cria diagramas de arquitetura em mermaid
- Compila as referências com trechos reais da documentação

**Padrão `llms.txt`:**

Se disponível, você pode encontrar um `llms.txt` em muitas referências de documentação fazendo `/llms.txt` nelas assim que chegar à página de documentação. Isso lhe dá uma versão limpa e otimizada para LLM da documentação.

**Filosofia: Construa Padrões Reutilizáveis**

De @omarsar0: "No início, passei um tempo construindo fluxos de trabalho/padrões reutilizáveis. Foi tedioso de construir, mas isso teve um efeito composto incrível à medida que os modelos e os harnesses de agentes melhoraram."

**No que investir:**

- Subagentes
- Skills
- Comandos
- Padrões de planejamento
- Ferramentas MCP
- Padrões de engenharia de contexto

---

## Melhores Práticas para Agentes e Subagentes

**O Problema de Contexto do Subagente:**

Os subagentes existem para economizar contexto, retornando resumos em vez de despejar tudo. Mas o orquestrador tem um contexto semântico que falta ao subagente. O subagente conhece apenas a consulta literal, não o PROPÓSITO por trás da solicitação.

**Padrão de Recuperação Iterativa:**

1. O orquestrador avalia cada retorno do subagente
2. Faça perguntas de acompanhamento antes de aceitá-lo
3. O subagente volta à fonte, obtém respostas, retorna
4. Faça um loop até ser suficiente (máximo de 3 ciclos)

**Chave:** Passe o contexto do objetivo, não apenas a consulta.

**Orquestrador com Fases Sequenciais:**

```markdown
Fase 1: PESQUISA (use o agente Explore) → research-summary.md
Fase 2: PLANO (use o agente planner) → plan.md
Fase 3: IMPLEMENTAÇÃO (use o agente tdd-guide) → alterações de código
Fase 4: REVISÃO (use o agente code-reviewer) → review-comments.md
Fase 5: VERIFICAÇÃO (use o build-error-resolver se necessário) → concluído ou volte ao início
```

**Regras chave:**

1. Cada agente recebe UMA entrada clara e produz UMA saída clara
2. As saídas se tornam entradas para a próxima fase
3. Nunca pule fases
4. Use `/clear` entre os agentes
5. Armazene as saídas intermediárias em arquivos

---

## COISAS DIVERTIDAS / NÃO CRÍTICAS, APENAS DICAS DIVERTIDAS

### Linha de Status Personalizada

Você pode defini-la usando `/statusline` - então Claude dirá que você não tem uma, mas pode configurá-la para você e perguntar o que você quer nela.

Veja também: `ccstatusline` (projeto da comunidade para linhas de status personalizadas do Claude Code)

### Transcrição de Voz

Fale com o Claude Code com sua voz. Mais rápido do que digitar para muitas pessoas.

- `superwhisper`, `MacWhisper` no Mac
- Mesmo com erros de transcrição, Claude entende a intenção

### Aliases de Terminal

```bash
alias c='claude'
alias gb='github'
alias co='code'
alias q='cd ~/Desktop/projects'
```

---

## Marco

![Mais de 25k Estrelas no GitHub](./assets/images/longform/09-25k-stars.png)
*Mais de 25.000 estrelas no GitHub em menos de uma semana*

---

## Recursos

**Orquestração de Agentes:**

- `claude-flow` — Plataforma de orquestração empresarial construída pela comunidade com mais de 54 agentes especializados

**Memória com Autoaperfeiçoamento:**

- Veja `skills/continuous-learning/` neste repositório
- `rlancemartin.github.io/2025/12/01/claude_diary/` - Padrão de reflexão de sessão

**Referência de Prompts de Sistema:**

- `system-prompts-and-models-of-ai-tools` — Coleção da comunidade de prompts de sistema de IA (mais de 110k estrelas)

**Oficial:**

- Anthropic Academy: `anthropic.skilljar.com`

---

## Referências

- [Anthropic: Desmistificando avaliações para agentes de IA](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [YK: 32 Dicas de Claude Code](https://agenticcoding.substack.com/p/32-claude-code-tips-from-basics-to)
- [RLanceMartin: Padrão de Reflexão de Sessão](https://rlancemartin.github.io/2025/12/01/claude_diary/)
- @PerceptualPeak: Negociação de Contexto de Subagente
- @menhguin: Tierlist de Abstrações de Agente
- @omarsar0: Filosofia de Efeitos Compostos

---

*Tudo abordado em ambos os guias está disponível no GitHub em [everything-claude-code](https://github.com/affaan-m/everything-claude-code)*
