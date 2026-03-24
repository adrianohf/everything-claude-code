---
name: claude-api
description: Padrões da API Anthropic Claude para Python e TypeScript. Abrange Messages API, streaming, uso de ferramentas (tool use), visão, pensamento estendido (extended thinking), lotes (batches), cache de prompt e Claude Agent SDK. Use ao construir aplicações com a API Claude ou SDKs da Anthropic.
origin: ECC
---

# API Claude

Construa aplicações com a API e SDKs da Anthropic Claude.

## Quando Ativar

- Construindo aplicações que chamam a API Claude
- O código importa `anthropic` (Python) ou `@anthropic-ai/sdk` (TypeScript)
- O usuário pergunta sobre padrões da API Claude, uso de ferramentas, streaming ou visão
- Implementando fluxos de trabalho de agentes com o Claude Agent SDK
- Otimizando custos de API, uso de tokens ou latência

## Seleção de Modelo

| Modelo | ID | Melhor Para |
|-------|-----|----------|
| Opus 4.1 | `claude-opus-4-1` | Raciocínio complexo, arquitetura, pesquisa |
| Sonnet 4 | `claude-sonnet-4-0` | Codificação equilibrada, a maioria das tarefas de desenvolvimento |
| Haiku 3.5 | `claude-3-5-haiku-latest` | Respostas rápidas, alto volume, sensibilidade a custo |

Use por padrão o Sonnet 4, a menos que a tarefa exija raciocínio profundo (Opus) ou otimização de velocidade/custo (Haiku). Para produção, prefira IDs de snapshot fixos em vez de aliases.

## SDK para Python

### Instalação

```bash
pip install anthropic
```

### Mensagem Básica

```python
import anthropic

client = anthropic.Anthropic()  # lê ANTHROPIC_API_KEY do ambiente

message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Explique async/await em Python"}
    ]
)
print(message.content[0].text)
```

### Streaming

```python
with client.messages.stream(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Escreva um haiku sobre programação"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

### Prompt de Sistema (System Prompt)

```python
message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    system="Você é um desenvolvedor Python sênior. Seja conciso.",
    messages=[{"role": "user", "content": "Revise esta função"}]
)
```

## SDK para TypeScript

### Instalação

```bash
npm install @anthropic-ai/sdk
```

### Mensagem Básica

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic(); // lê ANTHROPIC_API_KEY do ambiente

const message = await client.messages.create({
  model: "claude-sonnet-4-0",
  max_tokens: 1024,
  messages: [
    { role: "user", content: "Explique async/await em TypeScript" }
  ],
});
console.log(message.content[0].text);
```

### Streaming

```typescript
const stream = client.messages.stream({
  model: "claude-sonnet-4-0",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Escreva um haiku" }],
});

for await (const event of stream) {
  if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
    process.stdout.write(event.delta.text);
  }
}
```

## Uso de Ferramentas (Tool Use)

Defina ferramentas e deixe o Claude chamá-las:

```python
tools = [
    {
        "name": "get_weather",
        "description": "Obter o clima atual para uma localização",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "Nome da cidade"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["location"]
        }
    }
]

message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "Como está o clima em São Paulo?"}]
)

# Lidar com a resposta de uso de ferramenta
for block in message.content:
    if block.type == "tool_use":
        # Executar a ferramenta com block.input
        result = get_weather(**block.input)
        # Enviar o resultado de volta
        follow_up = client.messages.create(
            model="claude-sonnet-4-0",
            max_tokens=1024,
            tools=tools,
            messages=[
                {"role": "user", "content": "Como está o clima em São Paulo?"},
                {"role": "assistant", "content": message.content},
                {"role": "user", "content": [
                    {"type": "tool_result", "tool_use_id": block.id, "content": str(result)}
                ]}
            ]
        )
```

## Visão (Vision)

Envie imagens para análise:

```python
import base64

with open("diagrama.png", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": image_data}},
            {"type": "text", "text": "Descreva este diagrama"}
        ]
    }]
)
```

## Pensamento Estendido (Extended Thinking)

Para tarefas de raciocínio complexas:

```python
message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000
    },
    messages=[{"role": "user", "content": "Resolva este problema de matemática passo a passo..."}]
)

for block in message.content:
    if block.type == "thinking":
        print(f"Pensamento: {block.thinking}")
    elif block.type == "text":
        print(f"Resposta: {block.text}")
```

## Cache de Prompt (Prompt Caching)

Armazene em cache prompts de sistema grandes ou contextos para reduzir custos:

```python
message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    system=[
        {"type": "text", "text": prompt_de_sistema_grande, "cache_control": {"type": "ephemeral"}}
    ],
    messages=[{"role": "user", "content": "Pergunta sobre o contexto em cache"}]
)
# Verificar uso do cache
print(f"Leitura de cache: {message.usage.cache_read_input_tokens}")
print(f"Criação de cache: {message.usage.cache_creation_input_tokens}")
```

## API de Lotes (Batches API)

Processe grandes volumes de forma assíncrona com redução de custo de 50%:

```python
import time

batch = client.messages.batches.create(
    requests=[
        {
            "custom_id": f"request-{i}",
            "params": {
                "model": "claude-sonnet-4-0",
                "max_tokens": 1024,
                "messages": [{"role": "user", "content": prompt}]
            }
        }
        for i, prompt in enumerate(prompts)
    ]
)

# Consultar conclusão
while True:
    status = client.messages.batches.retrieve(batch.id)
    if status.processing_status == "ended":
        break
    time.sleep(30)

# Obter resultados
for result in client.messages.batches.results(batch.id):
    print(result.result.message.content[0].text)
```

## Claude Agent SDK

Construa agentes de várias etapas:

```python
# Nota: A superfície da API do Agent SDK pode mudar — verifique a documentação oficial
import anthropic

# Definir ferramentas como funções
tools = [{
    "name": "search_codebase",
    "description": "Pesquisar na base de código por código relevante",
    "input_schema": {
        "type": "object",
        "properties": {"query": {"type": "string"}},
        "required": ["query"]
    }
}]

# Executar um loop agêntico com uso de ferramentas
client = anthropic.Anthropic()
messages = [{"role": "user", "content": "Revise o módulo de autenticação em busca de problemas de segurança"}]

while True:
    response = client.messages.create(
        model="claude-sonnet-4-0",
        max_tokens=4096,
        tools=tools,
        messages=messages,
    )
    if response.stop_reason == "end_turn":
        break
    # Lidar com chamadas de ferramentas e continuar o loop
    messages.append({"role": "assistant", "content": response.content})
    # ... executar ferramentas e anexar mensagens tool_result
```

## Otimização de Custos

| Estratégia | Economia | Quando Usar |
|------------|----------|-------------|
| Cache de prompt | Até 90% em tokens em cache | Prompts de sistema ou contextos repetidos |
| API de Lotes | 50% | Processamento em lote não sensível ao tempo |
| Haiku em vez de Sonnet | ~75% | Tarefas simples, classificação, extração |
| max_tokens menor | Variável | Quando você sabe que a saída será curta |
| Streaming | Nenhuma (mesmo custo) | Melhor UX, mesmo preço |

## Tratamento de Erros

```python
import time

from anthropic import APIError, RateLimitError, APIConnectionError

try:
    message = client.messages.create(...)
except RateLimitError:
    # Aguardar e tentar novamente
    time.sleep(60)
except APIConnectionError:
    # Problema de rede, tentar novamente com backoff
    pass
except APIError as e:
    print(f"Erro de API {e.status_code}: {e.message}")
```

## Configuração de Ambiente

```bash
# Obrigatório
export ANTHROPIC_API_KEY="sua-chave-api-aqui"

# Opcional: definir modelo padrão
export ANTHROPIC_MODEL="claude-sonnet-4-0"
```

Nunca codifique chaves de API diretamente no código. Sempre use variáveis de ambiente.
