---
name: agent-harness-construction
description: Projete e otimize espaços de ação de agentes de IA, definições de ferramentas e formatação de observação para taxas de conclusão mais altas.
origin: ECC
---

# Construção do Harness do Agente

Use esta skill quando estiver melhorando como um agente planeja, chama ferramentas, se recupera de erros e converge para a conclusão.

## Modelo Principal

A qualidade da saída do agente é limitada por:
1. Qualidade do espaço de ação
2. Qualidade da observação
3. Qualidade da recuperação
4. Qualidade do orçamento de contexto

## Design do Espaço de Ação

1. Use nomes de ferramentas estáveis e explícitos.
2. Mantenha as entradas com esquema primeiro e restritas.
3. Retorne formas de saída determinísticas.
4. Evite ferramentas genéricas, a menos que o isolamento seja impossível.

## Regras de Granularidade

- Use microferramentas para operações de alto risco (implantação, migração, permissões).
- Use ferramentas médias para loops comuns de edição/leitura/pesquisa.
- Use macroferramentas apenas quando a sobrecarga de ida e volta for o custo dominante.

## Design da Observação

Cada resposta da ferramenta deve incluir:
- `status`: sucesso|aviso|erro
- `summary`: resultado em uma linha
- `next_actions`: acompanhamentos acionáveis
- `artifacts`: caminhos de arquivo / IDs

## Contrato de Recuperação de Erro

Para cada caminho de erro, inclua:
- dica da causa raiz
- instrução de nova tentativa segura
- condição de parada explícita

## Orçamento de Contexto

1. Mantenha o prompt do sistema mínimo e invariante.
2. Mova orientações grandes para skills carregadas sob demanda.
3. Prefira referências a arquivos em vez de embutir documentos longos.
4. Compacte nas fronteiras de fase, não em limites de token arbitrários.

## Orientação de Padrão de Arquitetura

- ReAct: melhor para tarefas exploratórias com caminho incerto.
- Chamada de função: melhor para fluxos determinísticos estruturados.
- Híbrido (recomendado): Planejamento ReAct + execução de ferramenta tipada.

## Benchmarking

Acompanhe:
- taxa de conclusão
- novas tentativas por tarefa
- pass@1 e pass@3
- custo por tarefa bem-sucedida

## Antipadrões

- Muitas ferramentas com semântica sobreposta.
- Saída de ferramenta opaca sem dicas de recuperação.
- Saída apenas de erro sem próximos passos.
- Sobre carga de contexto com referências irrelevantes.
