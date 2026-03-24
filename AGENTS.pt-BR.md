# Everything Claude Code (ECC) — Instruções para Agentes

Este é um **plugin de codificação com IA pronto para produção** que fornece 28 agentes especializados, 119 skills, 60 comandos e fluxos automatizados com hooks para desenvolvimento de software.

**Versão:** 1.9.0

## Princípios Centrais

1. **Agent-First** — Delegue tarefas para agentes especializados por domínio
2. **Orientado a Testes** — Escreva testes antes da implementação, com 80%+ de cobertura
3. **Segurança em Primeiro Lugar** — Nunca comprometa a segurança; valide todas as entradas
4. **Imutabilidade** — Sempre crie novos objetos; nunca mutile os existentes
5. **Planeje Antes de Executar** — Planeje funcionalidades complexas antes de escrever código

## Agentes Disponíveis

| Agente | Propósito | Quando usar |
|-------|---------|-------------|
| planner | Planejamento de implementação | Funcionalidades complexas, refatorações |
| architect | Design de sistema e escalabilidade | Decisões arquiteturais |
| tdd-guide | Desenvolvimento orientado a testes | Novas funcionalidades, correções |
| code-reviewer | Qualidade de código e manutenibilidade | Após escrever/modificar código |
| security-reviewer | Detecção de vulnerabilidades | Antes de commits, código sensível |
| build-error-resolver | Corrigir erros de build/tipo | Quando o build falha |
| e2e-runner | Testes E2E com Playwright | Fluxos críticos do usuário |
| refactor-cleaner | Limpeza de código morto | Manutenção de código |
| doc-updater | Documentação e codemaps | Atualização de docs |
| docs-lookup | Pesquisa de documentação e referência de API | Dúvidas sobre bibliotecas/APIs |
| cpp-reviewer | Revisão de código C++ | Projetos C++ |
| cpp-build-resolver | Erros de build em C++ | Falhas de build em C++ |
| go-reviewer | Revisão de código Go | Projetos Go |
| go-build-resolver | Erros de build em Go | Falhas de build em Go |
| kotlin-reviewer | Revisão de código Kotlin | Projetos Kotlin/Android/KMP |
| kotlin-build-resolver | Erros de build em Kotlin/Gradle | Falhas de build em Kotlin |
| database-reviewer | Especialista em PostgreSQL/Supabase | Design de schema, otimização de consultas |
| python-reviewer | Revisão de código Python | Projetos Python |
| java-reviewer | Revisão de código Java e Spring Boot | Projetos Java/Spring Boot |
| java-build-resolver | Erros de build em Java/Maven/Gradle | Falhas de build em Java |
| chief-of-staff | Triagem de comunicação e rascunhos | E-mail, Slack, LINE, Messenger |
| loop-operator | Execução autônoma de loops | Rodar loops com segurança, monitorar travamentos, intervir |
| harness-optimizer | Ajuste fino de harness | Confiabilidade, custo, throughput |
| rust-reviewer | Revisão de código Rust | Projetos Rust |
| rust-build-resolver | Erros de build em Rust | Falhas de build em Rust |
| pytorch-build-resolver | Erros de runtime/CUDA/treino em PyTorch | Falhas de build/treino em PyTorch |
| typescript-reviewer | Revisão de código TypeScript/JavaScript | Projetos TypeScript/JavaScript |

## Orquestração de Agentes

Use agentes proativamente, sem esperar o usuário pedir:
- Solicitações complexas de funcionalidade → **planner**
- Código recém-escrito/modificado → **code-reviewer**
- Correção de bug ou nova funcionalidade → **tdd-guide**
- Decisão arquitetural → **architect**
- Código sensível à segurança → **security-reviewer**
- Triagem de comunicação multicanal → **chief-of-staff**
- Loops autônomos / monitoramento de loops → **loop-operator**
- Confiabilidade e custo do harness → **harness-optimizer**

Use execução paralela em operações independentes.

## Diretrizes de Segurança

**Antes de QUALQUER commit:**
- Sem segredos hardcoded (chaves de API, senhas, tokens)
- Todas as entradas do usuário validadas
- Prevenção de SQL injection (consultas parametrizadas)
- Prevenção de XSS (HTML sanitizado)
- Proteção CSRF habilitada
- Autenticação/autorização verificadas
- Rate limiting em todos os endpoints
- Mensagens de erro não podem vazar dados sensíveis

**Gestão de segredos:** NUNCA hardcode segredos. Use variáveis de ambiente ou um gerenciador de segredos. Valide segredos obrigatórios na inicialização. Faça rotação imediata de qualquer segredo exposto.

## Estilo de Código

**Imutabilidade (CRÍTICO):** Sempre crie novos objetos; nunca mutile.

**Organização de arquivos:** Muitos arquivos pequenos em vez de poucos gigantes. O típico é 200-400 linhas, máximo 800. Organize por feature/domínio, não por tipo.

**Tratamento de erros:** Trate erros em todos os níveis. Forneça mensagens amigáveis em código de UI. Registre contexto detalhado no lado do servidor. Nunca engula erros em silêncio.

**Validação de entrada:** Valide toda entrada do usuário nos limites do sistema. Use validação baseada em schema. Falhe rápido com mensagens claras. Nunca confie em dados externos.

## Requisitos de Teste

**Cobertura mínima: 80%**

Tipos de teste:
1. **Testes unitários** — Funções, utilitários e componentes individuais
2. **Testes de integração** — Endpoints de API, operações de banco
3. **Testes E2E** — Fluxos críticos do usuário

**Fluxo TDD (obrigatório):**
1. Escreva o teste primeiro (RED)
2. Escreva a implementação mínima (GREEN)
3. Refatore (IMPROVE)

## Fluxo de Desenvolvimento

1. **Planejar** — Use o agente planner
2. **TDD** — Use o agente tdd-guide
3. **Revisar** — Use code-reviewer imediatamente
4. **Registrar conhecimento no lugar certo**
5. **Commitar** — Use formato de commits convencionais

## Fluxo Git

**Formato de commit:** `<type>: <description>` — Tipos: feat, fix, refactor, docs, test, chore, perf, ci

## Estrutura do Projeto

```text
agents/          — 28 subagentes especializados
skills/          — 117 skills e conhecimento de domínio
commands/        — 60 comandos slash
hooks/           — Automações disparadas por gatilho
rules/           — Diretrizes obrigatórias (comuns + por linguagem)
scripts/         — Utilitários Node.js multiplataforma
mcp-configs/     — 14 configurações de servidor MCP
tests/           — Suíte de testes
```

## Métricas de Sucesso

- Todos os testes passam com 80%+ de cobertura
- Nenhuma vulnerabilidade de segurança
- Código legível e manutenível
- Performance aceitável
- Requisitos do usuário atendidos
