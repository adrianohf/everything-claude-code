# Registro de Mudanças

## 1.9.0 - 2026-03-20

### Destaques

- Arquitetura de instalação seletiva com pipeline guiado por manifesto e armazenamento de estado em SQLite.
- Cobertura de linguagens expandida para mais de 10 ecossistemas, com 6 novos agentes e regras específicas por linguagem.
- Confiabilidade do observer reforçada com controle de memória, correções de sandbox e proteção de loop em 5 camadas.
- Base para skills autoevolutivas com evolução de skills e adaptadores de sessão.

### Novos Agentes

- `typescript-reviewer` — Especialista em revisão de código TypeScript/JavaScript (#647)
- `pytorch-build-resolver` — Resolução de erros de runtime, CUDA e treinamento em PyTorch (#549)
- `java-build-resolver` — Resolução de erros de build Maven/Gradle (#538)
- `java-reviewer` — Revisão de código Java e Spring Boot (#528)
- `kotlin-reviewer` — Revisão de código Kotlin/Android/KMP (#309)
- `kotlin-build-resolver` — Erros de build em Kotlin/Gradle (#309)
- `rust-reviewer` — Revisão de código Rust (#523)
- `rust-build-resolver` — Resolução de erros de build em Rust (#523)
- `docs-lookup` — Pesquisa de documentação e referência de API (#529)

### Novas Skills

- `pytorch-patterns` — Fluxos de deep learning com PyTorch (#550)
- `documentation-lookup` — Pesquisa de referência de API e documentação de bibliotecas (#529)
- `bun-runtime` — Padrões de runtime com Bun (#529)
- `nextjs-turbopack` — Fluxos de trabalho com Turbopack no Next.js (#529)
- `mcp-server-patterns` — Padrões de design para servidores MCP (#531)
- `data-scraper-agent` — Coleta pública de dados com IA (#503)
- `team-builder` — Skill de composição de equipe (#501)
- `ai-regression-testing` — Fluxos de teste de regressão com IA (#433)
- `claude-devfleet` — Orquestração multiagente (#505)
- `blueprint` — Planejamento de construção multi-sessão
- `everything-claude-code` — Skill autorreferente do ECC (#335)
- `prompt-optimizer` — Skill de otimização de prompts (#418)

### Infraestrutura e correções

- Arquitetura de instalação seletiva com resolução por manifesto
- Armazenamento de estado em SQLite com CLI de consulta
- Adaptadores de sessão para registro estruturado
- Base de evolução de skills para skills autoaperfeiçoáveis
- Diversas correções em CI, observer, hooks e compatibilidade com Windows

### Traduções

- Tradução para coreano (ko-KR)
- Sincronização da documentação em chinês (zh-CN)

## 1.8.0 - 2026-03-04

### Destaques

- Release focada em harness, com confiabilidade, disciplina de evals e operação autônoma de loops.
- O runtime de hooks agora suporta controle por perfil e desativação direcionada de hooks.
- NanoClaw v2 adiciona roteamento de modelos, hot-load de skills, branching, busca, compactação, exportação e métricas.

### Núcleo

- Novos comandos: `/harness-audit`, `/loop-start`, `/loop-status`, `/quality-gate`, `/model-route`
- Novas skills para construção de harness, engenharia agêntica e loops contínuos
- Novos agentes: `harness-optimizer` e `loop-operator`
