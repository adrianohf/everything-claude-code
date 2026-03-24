---
name: ai-first-engineering
description: Modelo de operação de engenharia para equipes onde agentes de IA geram uma grande parte da saída de implementação.
origin: ECC
---

# Engenharia com Foco em IA

Use esta habilidade ao projetar processos, revisões e arquitetura para equipes que entregam software com auxílio de código gerado por IA.

## Mudanças no Processo

1. A qualidade do planejamento importa mais do que a velocidade de digitação.
2. A cobertura da avaliação (eval) importa mais do que a confiança anedótica.
3. O foco da revisão muda da sintaxe para o comportamento do sistema.

## Requisitos de Arquitetura

Prefira arquiteturas que sejam amigáveis aos agentes:
- limites explícitos
- contratos estáveis
- interfaces tipadas
- testes determinísticos

Evite comportamentos implícitos espalhados por convenções ocultas.

## Revisão de Código em Equipes com Foco em IA

Revise para:
- regressões de comportamento
- suposições de segurança
- integridade dos dados
- tratamento de falhas
- segurança do rollout

Minimize o tempo gasto em questões de estilo já cobertas pela automação.

## Recrutamento e Sinais de Avaliação

Fortes engenheiros com foco em IA:
- decompõem trabalhos ambíguos de forma limpa
- definem critérios de aceitação mensuráveis
- produzem prompts e evals de alto sinal
- impõem controles de risco sob pressão de entrega

## Padrão de Testes

Eleve o nível dos testes para código gerado:
- cobertura de regressão obrigatória para domínios tocados
- asserções explícitas de casos extremos
- verificações de integração para limites de interface