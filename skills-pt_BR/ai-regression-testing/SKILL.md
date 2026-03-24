---
name: ai-regression-testing
description: Estratégias de testes de regressão para desenvolvimento assistido por IA. Testes de API em modo sandbox sem dependências de banco de dados, fluxos de trabalho automatizados de verificação de bugs e padrões para capturar pontos cegos de IA onde o mesmo modelo escreve e revisa o código.
origin: ECC
---

# Testes de Regressão de IA

Padrões de teste projetados especificamente para desenvolvimento assistido por IA, onde o mesmo modelo escreve o código e o revisa — criando pontos cegos sistemáticos que apenas testes automatizados podem capturar.

## Quando Ativar

- O agente de IA (Claude Code, Cursor, Codex) modificou rotas de API ou lógica de backend
- Um bug foi encontrado e corrigido — é necessário evitar a reintrodução
- O projeto possui um modo sandbox/mock que pode ser aproveitado para testes sem banco de dados
- Execução de `/bug-check` ou comandos de revisão semelhantes após alterações de código
- Existem vários caminhos de código (sandbox vs produção, feature flags, etc.)

## O Problema Principal

Quando uma IA escreve o código e depois revisa seu próprio trabalho, ela carrega as mesmas premissas para ambas as etapas. Isso cria um padrão de falha previsível:

```
IA escreve a correção → IA revisa a correção → IA diz "parece correto" → O bug ainda existe
```

**Exemplo do mundo real** (observado em produção):

```
Correção 1: Adicionado notification_settings à resposta da API
  → Esqueceu de adicioná-lo à consulta SELECT
  → A IA revisou e perdeu (mesmo ponto cego)

Correção 2: Adicionado à consulta SELECT
  → Erro de build do TypeScript (coluna não presente nos tipos gerados)
  → A IA revisou a Correção 1, mas não capturou o problema do SELECT

Correção 3: Alterado para SELECT *
  → Corrigiu o caminho de produção, esqueceu o caminho de sandbox
  → A IA revisou e perdeu NOVAMENTE (4ª ocorrência)

Correção 4: O teste capturou instantaneamente na primeira execução ✅
```

O padrão: **inconsistência entre o caminho de sandbox/produção** é a regressão nº 1 introduzida pela IA.

## Testes de API em Modo Sandbox

A maioria dos projetos com arquitetura amigável à IA possui um modo sandbox/mock. Esta é a chave para testes de API rápidos e sem banco de dados.

### Configuração (Vitest + Next.js App Router)

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";
import path from "path";

export default defineConfig({
  test: {
    environment: "node",
    globals: true,
    include: ["__tests__/**/*.test.ts"],
    setupFiles: ["__tests__/setup.ts"],
  },
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "."),
    },
  },
});
```

```typescript
// __tests__/setup.ts
// Forçar modo sandbox — nenhum banco de dados necessário
process.env.SANDBOX_MODE = "true";
process.env.NEXT_PUBLIC_SUPABASE_URL = "";
process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY = "";
```

### Auxiliar de Teste para Rotas de API Next.js

```typescript
// __tests__/helpers.ts
import { NextRequest } from "next/server";

export function createTestRequest(
  url: string,
  options?: {
    method?: string;
    body?: Record<string, unknown>;
    headers?: Record<string, string>;
    sandboxUserId?: string;
  },
): NextRequest {
  const { method = "GET", body, headers = {}, sandboxUserId } = options || {};
  const fullUrl = url.startsWith("http") ? url : `http://localhost:3000${url}`;
  const reqHeaders: Record<string, string> = { ...headers };

  if (sandboxUserId) {
    reqHeaders["x-sandbox-user-id"] = sandboxUserId;
  }

  const init: { method: string; headers: Record<string, string>; body?: string } = {
    method,
    headers: reqHeaders,
  };

  if (body) {
    init.body = JSON.stringify(body);
    reqHeaders["content-type"] = "application/json";
  }

  return new NextRequest(fullUrl, init);
}

export async function parseResponse(response: Response) {
  const json = await response.json();
  return { status: response.status, json };
}
```

### Escrevendo Testes de Regressão

O princípio fundamental: **escreva testes para bugs que foram encontrados, não para código que funciona**.

```typescript
// __tests__/api/user/profile.test.ts
import { describe, it, expect } from "vitest";
import { createTestRequest, parseResponse } from "../../helpers";
import { GET, PATCH } from "@/app/api/user/profile/route";

// Defina o contrato — quais campos DEVEM estar na resposta
const REQUIRED_FIELDS = [
  "id",
  "email",
  "full_name",
  "phone",
  "role",
  "created_at",
  "avatar_url",
  "notification_settings",  // ← Adicionado após o bug descobrir que estava faltando
];

describe("GET /api/user/profile", () => {
  it("retorna todos os campos obrigatórios", async () => {
    const req = createTestRequest("/api/user/profile");
    const res = await GET(req);
    const { status, json } = await parseResponse(res);

    expect(status).toBe(200);
    for (const field of REQUIRED_FIELDS) {
      expect(json.data).toHaveProperty(field);
    }
  });

  // Teste de regressão — este exato bug foi introduzido pela IA 4 vezes
  it("notification_settings não é undefined (regressão BUG-R1)", async () => {
    const req = createTestRequest("/api/user/profile");
    const res = await GET(req);
    const { json } = await parseResponse(res);

    expect("notification_settings" in json.data).toBe(true);
    const ns = json.data.notification_settings;
    expect(ns === null || typeof ns === "object").toBe(true);
  });
});
```

### Testando a Paridade Sandbox/Produção

A regressão de IA mais comum: corrigir o caminho de produção, mas esquecer o caminho de sandbox (ou vice-versa).

```typescript
// Teste se as respostas do sandbox correspondem ao contrato esperado
describe("GET /api/user/messages (lista de conversas)", () => {
  it("inclui partner_name no modo sandbox", async () => {
    const req = createTestRequest("/api/user/messages", {
      sandboxUserId: "user-001",
    });
    const res = await GET(req);
    const { json } = await parseResponse(res);

    // Isso capturou um bug onde partner_name foi adicionado
    // ao caminho de produção, mas não ao caminho de sandbox
    if (json.data.length > 0) {
      for (const conv of json.data) {
        expect("partner_name" in conv).toBe(true);
      }
    }
  });
});
```

## Integrando Testes ao Fluxo de Trabalho de Verificação de Bugs

### Definição de Comando Personalizado

```markdown
<!-- .claude/commands/bug-check.md -->
# Verificação de Bugs

## Passo 1: Testes Automatizados (obrigatório, não pode pular)

Execute estes comandos PRIMEIRO antes de qualquer revisão de código:

    npm run test       # Suíte de testes Vitest
    npm run build      # Verificação de tipo TypeScript + build

- Se os testes falharem → relate como bug de prioridade máxima
- Se o build falhar → relate erros de tipo como prioridade máxima
- Só prossiga para o Passo 2 se ambos passarem

## Passo 2: Revisão de Código (revisão por IA)

1. Consistência entre o caminho de sandbox / produção
2. O formato da resposta da API corresponde às expectativas do frontend
3. Completude da cláusula SELECT
4. Tratamento de erros com rollback
5. Condições de corrida em atualizações otimistas

## Passo 3: Para cada bug corrigido, proponha um teste de regressão
```

### O Fluxo de Trabalho

```
Usuário: "Verificar bugs" (ou "/bug-check")
  │
  ├─ Passo 1: npm run test
  │   ├─ FALHA → Bug encontrado mecanicamente (sem necessidade de julgamento da IA)
  │   └─ SUCESSO → Continuar
  │
  ├─ Passo 2: npm run build
  │   ├─ FALHA → Erro de tipo encontrado mecanicamente
  │   └─ SUCESSO → Continuar
  │
  ├─ Passo 3: Revisão de código por IA (com pontos cegos conhecidos em mente)
  │   └─ Descobertas relatadas
  │
  └─ Passo 4: Para cada correção, escreva um teste de regressão
      └─ A próxima verificação de bugs captura se a correção quebrar
```

## Padrões Comuns de Regressão de IA

### Padrão 1: Desacordo entre os Caminhos de Sandbox/Produção

**Frequência**: Mais comum (observado em 3 de cada 4 regressões)

```typescript
// ❌ IA adiciona campo apenas ao caminho de produção
if (isSandboxMode()) {
  return { data: { id, email, name } };  // Campo novo ausente
}
// Caminho de produção
return { data: { id, email, name, notification_settings } };

// ✅ Ambos os caminhos devem retornar o mesmo formato
if (isSandboxMode()) {
  return { data: { id, email, name, notification_settings: null } };
}
return { data: { id, email, name, notification_settings } };
```

**Teste para capturá-lo**:

```typescript
it("sandbox e produção retornam os mesmos campos", async () => {
  // No ambiente de teste, o modo sandbox é forçado como ATIVADO
  const res = await GET(createTestRequest("/api/user/profile"));
  const { json } = await parseResponse(res);

  for (const field of REQUIRED_FIELDS) {
    expect(json.data).toHaveProperty(field);
  }
});
```

### Padrão 2: Omissão da Cláusula SELECT

**Frequência**: Comum com Supabase/Prisma ao adicionar novas colunas

```typescript
// ❌ Nova coluna adicionada à resposta, mas não ao SELECT
const { data } = await supabase
  .from("users")
  .select("id, email, name")  // notification_settings não está aqui
  .single();

return { data: { ...data, notification_settings: data.notification_settings } };
// → notification_settings é sempre undefined

// ✅ Use SELECT * ou inclua explicitamente novas colunas
const { data } = await supabase
  .from("users")
  .select("*")
  .single();
```

### Padrão 3: Vazamento de Estado de Erro

**Frequência**: Moderada — ao adicionar tratamento de erros a componentes existentes

```typescript
// ❌ Estado de erro definido, mas dados antigos não limpos
catch (err) {
  setError("Falha ao carregar");
  // as reservas ainda mostram dados da aba anterior!
}

// ✅ Limpar estado relacionado em caso de erro
catch (err) {
  setReservations([]);  // Limpar dados obsoletos
  setError("Falha ao carregar");
}
```

### Padrão 4: Atualização Otimista Sem Rollback Adequado

```typescript
// ❌ Sem rollback em caso de falha
const handleRemove = async (id: string) => {
  setItems(prev => prev.filter(i => i.id !== id));
  await fetch(`/api/items/${id}`, { method: "DELETE" });
  // Se a API falhar, o item sumiu da UI, mas ainda está no DB
};

// ✅ Capturar estado anterior e fazer rollback em caso de falha
const handleRemove = async (id: string) => {
  const prevItems = [...items];
  setItems(prev => prev.filter(i => i.id !== id));
  try {
    const res = await fetch(`/api/items/${id}`, { method: "DELETE" });
    if (!res.ok) throw new Error("Erro na API");
  } catch {
    setItems(prevItems);  // Rollback
    alert("Falha ao excluir");
  }
};
```

## Estratégia: Testar Onde os Bugs Foram Encontrados

Não busque 100% de cobertura. Em vez disso:

```
Bug encontrado em /api/user/profile     → Escrever teste para API de perfil
Bug encontrado em /api/user/messages    → Escrever teste para API de mensagens
Bug encontrado em /api/user/favorites   → Escrever teste para API de favoritos
Nenhum bug em /api/user/notifications  → Não escrever teste (ainda)
```

**Por que isso funciona com o desenvolvimento de IA:**

1. A IA tende a cometer a **mesma categoria de erro** repetidamente
2. Os bugs se agrupam em áreas complexas (auth, lógica de vários caminhos, gerenciamento de estado)
3. Uma vez testada, essa regressão exata **não pode acontecer novamente**
4. A contagem de testes cresce organicamente com as correções de bugs — sem esforço desperdiçado

## Referência Rápida

| Padrão de Regressão de IA | Estratégia de Teste | Prioridade |
|---|---|---|
| Desacordo Sandbox/Produção | Afirmar o mesmo formato de resposta no modo sandbox | 🔴 Alta |
| Omissão da cláusula SELECT | Afirmar todos os campos obrigatórios na resposta | 🔴 Alta |
| Vazamento de estado de erro | Afirmar limpeza de estado em caso de erro | 🟡 Média |
| Falta de rollback | Afirmar estado restaurado em caso de falha da API | 🟡 Média |
| Type cast mascarando null | Afirmar que o campo não é undefined | 🟡 Média |

## O QUE FAZER / O QUE NÃO FAZER

**FAZER:**
- Escrever testes imediatamente após encontrar um bug (antes de corrigi-lo, se possível)
- Testar o formato da resposta da API, não a implementação
- Executar testes como o primeiro passo de cada verificação de bugs
- Manter os testes rápidos (< 1 segundo no total com o modo sandbox)
- Nomear os testes de acordo com o bug que eles evitam (ex: "regressão BUG-R1")

**NÃO FAZER:**
- Escrever testes para códigos que nunca tiveram um bug
- Confiar na autorrevisão da IA como substituto para testes automatizados
- Pular o teste do caminho do sandbox porque "são apenas dados fictícios"
- Escrever testes de integração quando os testes unitários são suficientes
- Buscar porcentagem de cobertura — buscar prevenção de regressão
