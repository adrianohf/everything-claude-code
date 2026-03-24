# Benchmark — Linha de Base de Desempenho e Detecção de Regressão

## Quando Usar

- Antes e depois de um PR para medir o impacto no desempenho
- Configuração de linhas de base (baselines) de desempenho para um projeto
- Quando os usuários relatam que "parece lento"
- Antes de um lançamento — garanta que você atenda às metas de desempenho
- Comparando sua stack com alternativas

## Como Funciona

### Modo 1: Desempenho da Página

Mede métricas reais do navegador via MCP do navegador:

```
1. Navegar para cada URL de destino
2. Medir Core Web Vitals:
   - LCP (Largest Contentful Paint) — meta < 2.5s
   - CLS (Cumulative Layout Shift) — meta < 0.1
   - INP (Interaction to Next Paint) — meta < 200ms
   - FCP (First Contentful Paint) — meta < 1.8s
   - TTFB (Time to First Byte) — meta < 800ms
3. Medir tamanhos de recursos:
   - Peso total da página (meta < 1MB)
   - Tamanho do bundle JS (meta < 200KB compactado)
   - Tamanho do CSS
   - Peso da imagem
   - Peso de scripts de terceiros
4. Contar solicitações de rede
5. Verificar recursos que bloqueiam a renderização
```

### Modo 2: Desempenho da API

Realiza benchmarks de endpoints de API:

```
1. Acessar cada endpoint 100 vezes
2. Medir: latência p50, p95, p99
3. Rastrear: tamanho da resposta, códigos de status
4. Testar sob carga: 10 solicitações simultâneas
5. Comparar com as metas de SLA
```

### Modo 3: Desempenho de Build

Mede o ciclo de feedback de desenvolvimento:

```
1. Tempo de build a frio (cold build)
2. Tempo de recarregamento a quente (HMR - hot reload)
3. Duração da suíte de testes
4. Tempo de verificação do TypeScript
5. Tempo de lint
6. Tempo de build do Docker
```

### Modo 4: Comparação Antes/Depois

Executar antes e depois de uma alteração para medir o impacto:

```
/benchmark baseline    # salva as métricas atuais
# ... fazer alterações ...
/benchmark compare     # compara com a linha de base
```

Saída:
```
| Métrica | Antes | Depois | Delta | Veredito |
|---------|-------|--------|-------|----------|
| LCP | 1.2s | 1.4s | +200ms | ⚠ AVISO |
| Bundle | 180KB | 175KB | -5KB | ✓ MELHOR |
| Build | 12s | 14s | +2s | ⚠ AVISO |
```

## Saída

Armazena as linhas de base em `.ecc/benchmarks/` como JSON. Rastreado pelo Git para que a equipe compartilhe as linhas de base.

## Integração

- CI: execute `/benchmark compare` em cada PR
- Combine com `/canary-watch` para monitoramento pós-implantação
- Combine com `/browser-qa` para um checklist completo pré-envio
