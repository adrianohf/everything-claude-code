---
name: claude-devfleet
description: Orquestre tarefas de codificação multiagente via Claude DevFleet — planeje projetos, envie agentes paralelos em worktrees isolados, monitore o progresso e leia relatórios estruturados.
origin: community
---

# Orquestração Multiagente Claude DevFleet

## Quando Usar

Use esta skill quando precisar enviar vários agentes Claude Code para trabalhar em tarefas de codificação em paralelo. Cada agente roda em um git worktree isolado com ferramentas completas.

Requer uma instância do Claude DevFleet rodando e conectada via MCP:
```bash
claude mcp add devfleet --transport http http://localhost:18801/mcp
```

## Como Funciona

```
Usuário → "Construa uma API REST com autenticação e testes"
  ↓
plan_project(prompt) → project_id + DAG de missões
  ↓
Mostrar plano ao usuário → obter aprovação
  ↓
dispatch_mission(M1) → Agente 1 inicia no worktree
  ↓
M1 concluída → auto-merge → auto-dispatch M2 (depende de M1)
  ↓
M2 concluída → auto-merge
  ↓
get_report(M2) → arquivos_alterados, o_que_foi_feito, erros, próximos_passos
  ↓
Relatar ao usuário
```

### Ferramentas

| Ferramenta | Propósito |
|------|---------|
| `plan_project(prompt)` | A IA divide uma descrição em um projeto com missões encadeadas |
| `create_project(name, path?, description?)` | Cria um projeto manualmente, retorna `project_id` |
| `create_mission(project_id, title, prompt, depends_on?, auto_dispatch?)` | Adiciona uma missão. `depends_on` é uma lista de strings de ID de missão (ex: `["abc-123"]`). Defina `auto_dispatch=true` para iniciar automaticamente quando as dependências forem atendidas. |
| `dispatch_mission(mission_id, model?, max_turns?)` | Inicia um agente em uma missão |
| `cancel_mission(mission_id)` | Para um agente em execução |
| `wait_for_mission(mission_id, timeout_seconds?)` | Bloqueia até que uma missão seja concluída (veja nota abaixo) |
| `get_mission_status(mission_id)` | Verifica o progresso da missão sem bloquear |
| `get_report(mission_id)` | Lê o relatório estruturado (arquivos alterados, testados, erros, próximos passos) |
| `get_dashboard()` | Visão geral do sistema: agentes rodando, estatísticas, atividade recente |
| `list_projects()` | Navega por todos os projetos |
| `list_missions(project_id, status?)` | Lista as missões de um projeto |

> **Nota sobre `wait_for_mission`:** Isso bloqueia a conversa por até `timeout_seconds` (padrão 600). Para missões de longa duração, prefira fazer consultas (polling) com `get_mission_status` a cada 30–60 segundos, para que o usuário veja as atualizações de progresso.

### Fluxo de Trabalho: Planejar → Enviar → Monitorar → Relatar

1. **Planejar**: Chamar `plan_project(prompt="...")` → retorna `project_id` + lista de missões com cadeias de `depends_on` e `auto_dispatch=true`.
2. **Mostrar plano**: Apresentar os títulos das missões, tipos e cadeia de dependências ao usuário.
3. **Enviar**: Chamar `dispatch_mission(mission_id=<primeiro_id_da_missao>)` na missão raiz (`depends_on` vazio). As missões restantes são enviadas automaticamente conforme suas dependências são concluídas (porque `plan_project` define `auto_dispatch=true` nelas).
4. **Monitorar**: Chamar `get_mission_status(mission_id=...)` ou `get_dashboard()` para verificar o progresso.
5. **Relatar**: Chamar `get_report(mission_id=...)` quando as missões forem concluídas. Compartilhar os destaques com o usuário.

### Concorrência

O DevFleet executa até 3 agentes simultâneos por padrão (configurável via `DEVFLEET_MAX_AGENTS`). Quando todos os slots estão ocupados, as missões com `auto_dispatch=true` entram na fila do monitor de missões e são enviadas automaticamente conforme os slots são liberados. Verifique `get_dashboard()` para ver o uso atual dos slots.

## Exemplos

### Automático total: planejar e lançar

1. `plan_project(prompt="...")` → mostra o plano com missões e dependências.
2. Envia a primeira missão (aquela com `depends_on` vazio).
3. As missões restantes são enviadas automaticamente conforme as dependências são resolvidas (elas possuem `auto_dispatch=true`).
4. Relata de volta com o ID do projeto e a contagem de missões para que o usuário saiba o que foi lançado.
5. Faz consultas com `get_mission_status` ou `get_dashboard()` periodicamente até que todas as missões atinjam um estado terminal (`completed`, `failed` ou `cancelled`).
6. `get_report(mission_id=...)` para cada missão terminada — resume os sucessos e aponta as falhas com erros e próximos passos.

### Manual: controle passo a passo

1. `create_project(name="Meu Projeto")` → retorna `project_id`.
2. `create_mission(project_id=project_id, title="...", prompt="...", auto_dispatch=true)` para a primeira missão (raiz) → captura o `root_mission_id`.
   `create_mission(project_id=project_id, title="...", prompt="...", auto_dispatch=true, depends_on=["<root_mission_id>"])` para cada tarefa subsequente.
3. `dispatch_mission(mission_id=...)` na primeira missão para iniciar a cadeia.
4. `get_report(mission_id=...)` quando terminar.

### Sequencial com revisão

1. `create_project(name="...")` → obtém `project_id`.
2. `create_mission(project_id=project_id, title="Implementar funcionalidade", prompt="...")` → obtém `impl_mission_id`.
3. `dispatch_mission(mission_id=impl_mission_id)`, então consulta com `get_mission_status` até concluir.
4. `get_report(mission_id=impl_mission_id)` para revisar os resultados.
5. `create_mission(project_id=project_id, title="Revisão", prompt="...", depends_on=[impl_mission_id], auto_dispatch=true)` — inicia automaticamente, já que a dependência já foi atendida.

## Diretrizes

- Sempre confirme o plano com o usuário antes de enviar, a menos que ele tenha dito para prosseguir.
- Inclua os títulos e IDs das missões ao relatar o status.
- Se uma missão falhar, leia o relatório dela antes de tentar novamente.
- Verifique `get_dashboard()` para disponibilidade de slots de agente antes de enviar em massa.
- As dependências das missões formam um DAG — não crie dependências circulares.
- Cada agente roda em um git worktree isolado e faz o merge automático na conclusão. Se ocorrer um conflito de merge, as alterações permanecem na branch do worktree do agente para resolução manual.
- Ao criar missões manualmente, sempre defina `auto_dispatch=true` se quiser que elas sejam acionadas automaticamente quando as dependências forem concluídas. Sem essa flag, as missões permanecem no status `draft`.
