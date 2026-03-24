# Canary Watch — Monitoramento Pós-Implantação

## Quando Usar

- Após a implantação em produção ou staging
- Após a mesclagem de um PR de risco
- Quando você quiser verificar se uma correção realmente funcionou
- Monitoramento contínuo durante uma janela de lançamento
- Após atualizações de dependências

## Como Funciona

Monitora uma URL implantada em busca de regressões. É executado em um loop até ser interrompido ou até que a janela de observação expire.

### O que ele monitora

```
1. Status HTTP — a página está retornando 200?
2. Erros no Console — novos erros que não estavam lá antes?
3. Falhas de Rede — chamadas de API falhas, respostas 5xx?
4. Desempenho — regressão de LCP/CLS/INP em relação à linha de base?
5. Conteúdo — elementos principais desapareceram? (h1, nav, footer, CTA)
6. Saúde da API — os endpoints críticos estão respondendo dentro do SLA?
```

### Modos de Observação

**Verificação rápida** (padrão): única passagem, relata os resultados
```
/canary-watch https://myapp.com
```

**Observação sustentada**: verificar a cada N minutos por M horas
```
/canary-watch https://myapp.com --interval 5m --duration 2h
```

**Modo Diff**: comparar staging vs produção
```
/canary-watch --compare https://staging.myapp.com https://myapp.com
```

### Limiares de Alerta (Alert Thresholds)

```yaml
crítico:  # alerta imediato
  - Status HTTP != 200
  - Contagem de erros no console > 5 (apenas erros novos)
  - LCP > 4s
  - Endpoint da API retorna 5xx

aviso:   # sinalizar no relatório
  - LCP aumentou > 500ms em relação à linha de base
  - CLS > 0.1
  - Novos avisos (warnings) no console
  - Tempo de resposta > 2x a linha de base

info:      # apenas log
  - Pequena variação de desempenho
  - Novas solicitações de rede (scripts de terceiros adicionados?)
```

### Notificações

Quando um limiar crítico é ultrapassado:
- Notificação na área de trabalho (macOS/Linux)
- Opcional: webhook do Slack/Discord
- Registrar em `~/.claude/canary-watch.log`

## Saída

```markdown
## Relatório Canary — myapp.com — 2026-03-23 03:15 PST

### Status: SAUDÁVEL (HEALTHY) ✓

| Verificação | Resultado | Linha de Base | Delta |
|-------------|-----------|---------------|-------|
| HTTP | 200 ✓ | 200 | — |
| Erros no console | 0 ✓ | 0 | — |
| LCP | 1.8s ✓ | 1.6s | +200ms |
| CLS | 0.01 ✓ | 0.01 | — |
| API /health | 145ms ✓ | 120ms | +25ms |

### Nenhuma regressão detectada. A implantação está limpa.
```

## Integração

Combine com:
- `/browser-qa` para verificação pré-implantação
- Hooks: adicione como um hook PostToolUse em `git push` para verificar automaticamente após as implantações
- CI: execute no GitHub Actions após a etapa de implantação (deploy)
