# Browser QA — Testes Visuais e de Interação Automatizados

## Quando Usar

- Após implantar uma funcionalidade em ambiente de staging/preview
- Quando você precisar verificar o comportamento da UI em várias páginas
- Antes do lançamento — confirme se layouts, formulários e interações realmente funcionam
- Ao revisar PRs que tocam no código do frontend
- Auditorias de acessibilidade e testes responsivos

## Como Funciona

Usa o MCP de automação de navegador (claude-in-chrome, Playwright ou Puppeteer) para interagir com páginas reais como um usuário real.

### Fase 1: Teste de Fumaça (Smoke Test)
```
1. Navegar para a URL de destino
2. Verificar erros no console (filtrar ruídos: analytics, terceiros)
3. Verificar se não há 4xx/5xx nas solicitações de rede
4. Captura de tela da parte visível (above-the-fold) no viewport de desktop + mobile
5. Verificar Core Web Vitals: LCP < 2.5s, CLS < 0.1, INP < 200ms
```

### Fase 2: Teste de Interação
```
1. Clicar em cada link de navegação — verificar se não há links quebrados
2. Enviar formulários com dados válidos — verificar o estado de sucesso
3. Enviar formulários com dados inválidos — verificar o estado de erro
4. Testar o fluxo de autenticação: login → página protegida → logout
5. Testar jornadas críticas do usuário (checkout, onboarding, pesquisa)
```

### Fase 3: Regressão Visual
```
1. Captura de tela das páginas principais em 3 breakpoints (375px, 768px, 1440px)
2. Comparar com as capturas de tela da linha de base (se armazenadas)
3. Sinalizar mudanças de layout > 5px, elementos ausentes, estouro de conteúdo (overflow)
4. Verificar o modo escuro, se aplicável
```

### Fase 4: Acessibilidade
```
1. Executar o axe-core ou equivalente em cada página
2. Sinalizar violações WCAG AA (contraste, rótulos, ordem de foco)
3. Verificar se a navegação pelo teclado funciona de ponta a ponta
4. Verificar marcos (landmarks) do leitor de tela
```

## Formato de Saída

```markdown
## Relatório de QA — [URL] — [timestamp]

### Teste de Fumaça
- Erros no console: 0 críticos, 2 avisos (ruído de analytics)
- Rede: todos 200/304, sem falhas
- Core Web Vitals: LCP 1.2s ✓, CLS 0.02 ✓, INP 89ms ✓

### Interações
- [✓] Links de navegação: 12/12 funcionando
- [✗] Formulário de contato: falta o estado de erro para e-mail inválido
- [✓] Fluxo de autenticação: login/logout funcionando

### Visual
- [✗] A seção hero transborda (overflow) no viewport de 375px
- [✓] Modo escuro: todas as páginas consistentes

### Acessibilidade
- 2 violações AA: falta texto alternativo na imagem hero, baixo contraste nos links do rodapé

### Veredito: LANÇAR COM CORREÇÕES (2 problemas, 0 bloqueadores)
```

## Integração

Funciona com qualquer MCP de navegador:
- Ferramentas `mChild__claude-in-chrome__*` (preferencial — usa seu Chrome real)
- Playwright via `mcp__browserbase__*`
- Scripts diretos do Puppeteer

Combine com `/canary-watch` para monitoramento pós-implantação.
