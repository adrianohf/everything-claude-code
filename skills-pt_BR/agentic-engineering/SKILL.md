---
name: agentic-engineering
description: Opere como um engenheiro agêntico usando execução focada em avaliação, decomposição e roteamento de modelo ciente de custos.
origin: ECC
---

# Engenharia Agêntica

Use esta skill para fluxos de trabalho de engenharia onde agentes de IA realizam a maior parte do trabalho de implementação e os humanos aplicam controles de qualidade e risco.

## Princípios Operacionais

1. Defina os critérios de conclusão antes da execução.
2. Decomponha o trabalho em unidades do tamanho de um agente.
3. Roteie os níveis do modelo pela complexidade da tarefa.
4. Meça com avaliações e verificações de regressão.

## Loop Focado em Avaliação

1. Defina a avaliação de capacidade e a avaliação de regressão.
2. Execute a linha de base e capture as assinaturas de falha.
3. Execute a implementação.
4. Execute novamente as avaliações e compare as diferenças.

## Decomposição de Tarefas

Aplique a regra da unidade de 15 minutos:
- cada unidade deve ser verificável independentemente
- cada unidade deve ter um único risco dominante
- cada unidade deve expor uma condição de conclusão clara

## Roteamento de Modelo

- Haiku: classificação, transformações de boilerplate, edições restritas
- Sonnet: implementação e refatorações
- Opus: arquitetura, análise de causa raiz, invariantes de vários arquivos

## Estratégia de Sessão

- Continue a sessão para unidades fortemente acopladas.
- Inicie uma nova sessão após as principais transições de fase.
- Compacte após a conclusão do marco, não durante a depuração ativa.

## Foco da Revisão para Código Gerado por IA

Priorize:
- invariantes e casos extremos
- limites de erro
- premissas de segurança e autenticação
- acoplamento oculto e risco de implementação

Não desperdice ciclos de revisão em divergências apenas de estilo quando a formatação/lint automatizada já impõe o estilo.

## Disciplina de Custos

Acompanhe por tarefa:
- modelo
- estimativa de tokens
- novas tentativas
- tempo de relógio de parede
- sucesso/falha

Eleve o nível do modelo apenas quando o nível inferior falhar com uma lacuna de raciocínio clara.
