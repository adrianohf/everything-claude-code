# Política de Segurança

## Versões Suportadas

| Versão | Suportada          |
| ------- | ------------------ |
| 1.9.x   | :white_check_mark: |
| 1.8.x   | :white_check_mark: |
| < 1.8   | :x:                |

## Reportando uma Vulnerabilidade

Se você descobrir uma vulnerabilidade de segurança no ECC, por favor reporte com responsabilidade.

**Não abra uma issue pública no GitHub para vulnerabilidades de segurança.**

Em vez disso, envie um e-mail para **security@ecc.tools** com:

- Uma descrição da vulnerabilidade
- Passos para reproduzir
- As versões afetadas
- Qualquer avaliação de impacto em potencial

Você pode esperar:

- **Confirmação de recebimento** em até 48 horas
- **Atualização de status** em até 7 dias
- **Correção ou mitigação** em até 30 dias para casos críticos

Se a vulnerabilidade for aceita, nós iremos:

- Creditar você nas notas de release (a menos que prefira anonimato)
- Corrigir o problema em tempo hábil
- Coordenar o timing da divulgação com você

Se a vulnerabilidade for recusada, explicaremos o motivo e orientaremos se ela deve ser reportada em outro lugar.

## Escopo

Esta política cobre:

- O plugin ECC e todos os scripts deste repositório
- Scripts de hook executados na sua máquina
- Scripts de ciclo de vida de instalação/desinstalação/reparo
- Configurações MCP distribuídas com o ECC
- O scanner de segurança AgentShield ([github.com/affaan-m/agentshield](https://github.com/affaan-m/agentshield))

## Recursos de Segurança

- **AgentShield**: Escaneie sua configuração de agente em busca de vulnerabilidades — `npx ecc-agentshield scan`
- **Guia de Segurança**: [The Shorthand Guide to Everything Agentic Security](./the-security-guide.md)
- **OWASP MCP Top 10**: [owasp.org/www-project-mcp-top-10](https://owasp.org/www-project-mcp-top-10/)
- **OWASP Agentic Applications Top 10**: [genai.owasp.org](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)
