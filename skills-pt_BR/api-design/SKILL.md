---
name: api-design
description: Padrões de design de API REST, incluindo nomenclatura de recursos, códigos de status, paginação, filtragem, respostas de erro, versionamento e limitação de taxa (rate limiting) para APIs de produção.
origin: ECC
---

# Padrões de Design de API

Convenções e melhores práticas para projetar APIs REST consistentes e amigáveis ao desenvolvedor.

## Quando Ativar

- Projetando novos endpoints de API
- Revisando contratos de API existentes
- Adicionando paginação, filtragem ou ordenação
- Implementando tratamento de erros para APIs
- Planejando estratégia de versionamento de API
- Construindo APIs públicas ou voltadas para parceiros

## Design de Recursos

### Estrutura de URL

```
# Recursos são substantivos, no plural, letras minúsculas, kebab-case
GET    /api/v1/users
GET    /api/v1/users/:id
POST   /api/v1/users
PUT    /api/v1/users/:id
PATCH  /api/v1/users/:id
DELETE /api/v1/users/:id

# Sub-recursos para relacionamentos
GET    /api/v1/users/:id/orders
POST   /api/v1/users/:id/orders

# Ações que não mapeiam para CRUD (use verbos com moderação)
POST   /api/v1/orders/:id/cancel
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
```

### Regras de Nomenclatura

```
# BOM
/api/v1/team-members          # kebab-case para recursos com várias palavras
/api/v1/orders?status=active  # parâmetros de consulta para filtragem
/api/v1/users/123/orders      # recursos aninhados para propriedade

# RUIM
/api/v1/getUsers              # verbo na URL
/api/v1/user                  # singular (use plural)
/api/v1/team_members          # snake_case em URLs
/api/v1/users/123/getOrders   # verbo em recurso aninhado
```

## Métodos HTTP e Códigos de Status

### Semântica dos Métodos

| Método | Idempotente | Seguro | Uso Para |
|--------|-----------|------|---------|
| GET | Sim | Sim | Recuperar recursos |
| POST | Não | Não | Criar recursos, acionar ações |
| PUT | Sim | Não | Substituição total de um recurso |
| PATCH | Não* | Não | Atualização parcial de um recurso |
| DELETE | Sim | Não | Remover um recurso |

*PATCH pode se tornar idempotente com a implementação adequada

### Referência de Códigos de Status

```
# Sucesso
200 OK                    — GET, PUT, PATCH (com corpo de resposta)
201 Created               — POST (incluir cabeçalho Location)
204 No Content            — DELETE, PUT (sem corpo de resposta)

# Erros do Cliente
400 Bad Request           — Falha de validação, JSON malformado
401 Unauthorized          — Autenticação ausente ou inválida
403 Forbidden             — Autenticado, mas não autorizado
404 Not Found             — O recurso não existe
409 Conflict              — Entrada duplicada, conflito de estado
422 Unprocessable Entity  — Semanticamente inválido (JSON válido, dados ruins)
429 Too Many Requests     — Limite de taxa excedido

# Erros do Servidor
500 Internal Server Error — Falha inesperada (nunca exponha detalhes)
502 Bad Gateway           — O serviço upstream falhou
503 Service Unavailable   — Sobrecarga temporária, incluir Retry-After
```

### Erros Comuns

```
# RUIM: 200 para tudo
{ "status": 200, "success": false, "error": "Not found" }

# BOM: Use códigos de status HTTP semanticamente
HTTP/1.1 404 Not Found
{ "error": { "code": "not_found", "message": "Usuário não encontrado" } }

# RUIM: 500 para erros de validação
# BOM: 400 ou 422 com detalhes em nível de campo

# RUIM: 200 para recursos criados
# BOM: 201 com cabeçalho Location
HTTP/1.1 201 Created
Location: /api/v1/users/abc-123
```

## Formato de Resposta

### Resposta de Sucesso

```json
{
  "data": {
    "id": "abc-123",
    "email": "alice@example.com",
    "name": "Alice",
    "created_at": "2025-01-15T10:30:00Z"
  }
}
```

### Resposta de Coleção (com Paginação)

```json
{
  "data": [
    { "id": "abc-123", "name": "Alice" },
    { "id": "def-456", "name": "Bob" }
  ],
  "meta": {
    "total": 142,
    "page": 1,
    "per_page": 20,
    "total_pages": 8
  },
  "links": {
    "self": "/api/v1/users?page=1&per_page=20",
    "next": "/api/v1/users?page=2&per_page=20",
    "last": "/api/v1/users?page=8&per_page=20"
  }
}
```

### Resposta de Erro

```json
{
  "error": {
    "code": "validation_error",
    "message": "A validação da solicitação falhou",
    "details": [
      {
        "field": "email",
        "message": "Deve ser um endereço de e-mail válido",
        "code": "invalid_format"
      },
      {
        "field": "age",
        "message": "Deve estar entre 0 e 150",
        "code": "out_of_range"
      }
    ]
  }
}
```

### Variantes de Envelope de Resposta

```typescript
// Opção A: Envelope com wrapper de dados (recomendado para APIs públicas)
interface ApiResponse<T> {
  data: T;
  meta?: PaginationMeta;
  links?: PaginationLinks;
}

interface ApiError {
  error: {
    code: string;
    message: string;
    details?: FieldError[];
  };
}

// Opção B: Resposta simples (mais simples, comum para APIs internas)
// Sucesso: basta retornar o recurso diretamente
// Erro: retornar objeto de erro
// Distinguir pelo código de status HTTP
```

## Paginação

### Baseada em Deslocamento (Simples)

```
GET /api/v1/users?page=2&per_page=20

# Implementação
SELECT * FROM users
ORDER BY created_at DESC
LIMIT 20 OFFSET 20;
```

**Prós:** Fácil de implementar, suporta "pular para a página N"
**Contras:** Lenta em grandes deslocamentos (OFFSET 100000), inconsistente com inserções simultâneas

### Baseada em Cursor (Escalável)

```
GET /api/v1/users?cursor=eyJpZCI6MTIzfQ&limit=20

# Implementação
SELECT * FROM users
WHERE id > :cursor_id
ORDER BY id ASC
LIMIT 21;  -- busca um extra para determinar has_next
```

```json
{
  "data": [...],
  "meta": {
    "has_next": true,
    "next_cursor": "eyJpZCI6MTQzfQ"
  }
}
```

**Prós:** Desempenho consistente independentemente da posição, estável com inserções simultâneas
**Contras:** Não é possível pular para uma página arbitrária, o cursor é opaco

### Quando usar cada uma

| Caso de Uso | Tipo de Paginação |
|----------|----------------|
| Painéis administrativos, conjuntos de dados pequenos (<10K) | Deslocamento (Offset) |
| Rolagem infinita, feeds, conjuntos de dados grandes | Cursor |
| APIs públicas | Cursor (padrão) com deslocamento (opcional) |
| Resultados de pesquisa | Deslocamento (os usuários esperam números de página) |

## Filtragem, Ordenação e Pesquisa

### Filtragem

```
# Igualdade simples
GET /api/v1/orders?status=active&customer_id=abc-123

# Operadores de comparação (use notação de colchetes)
GET /api/v1/products?price[gte]=10&price[lte]=100
GET /api/v1/orders?created_at[after]=2025-01-01

# Múltiplos valores (separados por vírgula)
GET /api/v1/products?category=electronics,clothing

# Campos aninhados (notação de ponto)
GET /api/v1/orders?customer.country=US
```

### Ordenação

```
# Campo único (prefixo - para decrescente)
GET /api/v1/products?sort=-created_at

# Múltiplos campos (separados por vírgula)
GET /api/v1/products?sort=-featured,price,-created_at
```

### Pesquisa de Texto Completo (Full-Text Search)

```
# Parâmetro de consulta de pesquisa
GET /api/v1/products?q=wireless+headphones

# Pesquisa específica de campo
GET /api/v1/users?email=alice
```

### Fieldsets Esparsos (Sparse Fieldsets)

```
# Retornar apenas campos especificados (reduz o payload)
GET /api/v1/users?fields=id,name,email
GET /api/v1/orders?fields=id,total,status&include=customer.name
```

## Autenticação e Autorização

### Autenticação Baseada em Token

```
# Token Bearer no cabeçalho Authorization
GET /api/v1/users
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

# Chave de API (para servidor para servidor)
GET /api/v1/data
X-API-Key: sk_live_abc123
```

### Padrões de Autorização

```typescript
// Nível de recurso: verificar propriedade
app.get("/api/v1/orders/:id", async (req, res) => {
  const order = await Order.findById(req.params.id);
  if (!order) return res.status(404).json({ error: { code: "not_found" } });
  if (order.userId !== req.user.id) return res.status(403).json({ error: { code: "forbidden" } });
  return res.json({ data: order });
});

// Baseado em função: verificar permissões
app.delete("/api/v1/users/:id", requireRole("admin"), async (req, res) => {
  await User.delete(req.params.id);
  return res.status(204).send();
});
```

## Limitação de Taxa (Rate Limiting)

### Cabeçalhos

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1640000000

# Quando excedido
HTTP/1.1 429 Too Many Requests
Retry-After: 60
{
  "error": {
    "code": "rate_limit_exceeded",
    "message": "Limite de taxa excedido. Tente novamente em 60 segundos."
  }
}
```

### Níveis de Limite de Taxa

| Nível | Limite | Janela | Caso de Uso |
|------|-------|--------|----------|
| Anônimo | 30/min | Por IP | Endpoints públicos |
| Autenticado | 100/min | Por usuário | Acesso padrão à API |
| Premium | 1000/min | Por chave de API | Planos de API pagos |
| Interno | 10000/min | Por serviço | Serviço para serviço |

## Versionamento

### Versionamento no Caminho da URL (Recomendado)

```
/api/v1/users
/api/v2/users
```

**Prós:** Explícito, fácil de rotear, cacheável
**Contras:** A URL muda entre as versões

### Versionamento por Cabeçalho

```
GET /api/users
Accept: application/vnd.myapp.v2+json
```

**Prós:** URLs limpas
**Contras:** Mais difícil de testar, fácil de esquecer

### Estratégia de Versionamento

```
1. Comece com /api/v1/ — não versione até que precise
2. Mantenha no máximo 2 versões ativas (atual + anterior)
3. Cronograma de depreciação:
   - Anunciar depreciação (aviso de 6 meses para APIs públicas)
   - Adicionar cabeçalho Sunset: Sunset: Sat, 01 Jan 2026 00:00:00 GMT
   - Retornar 410 Gone após a data sunset
4. Alterações que não quebram a compatibilidade não precisam de uma nova versão:
   - Adicionar novos campos às respostas
   - Adicionar novos parâmetros de consulta opcionais
   - Adicionar novos endpoints
5. Alterações que quebram a compatibilidade exigem uma nova versão:
   - Remover ou renomear campos
   - Alterar tipos de campo
   - Alterar a estrutura da URL
   - Alterar o método de autenticação
```

## Padrões de Implementação

### TypeScript (Rota de API Next.js)

```typescript
import { z } from "zod";
import { NextRequest, NextResponse } from "next/server";

const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
});

export async function POST(req: NextRequest) {
  const body = await req.json();
  const parsed = createUserSchema.safeParse(body);

  if (!parsed.success) {
    return NextResponse.json({
      error: {
        code: "validation_error",
        message: "A validação da solicitação falhou",
        details: parsed.error.issues.map(i => ({
          field: i.path.join("."),
          message: i.message,
          code: i.code,
        })),
      },
    }, { status: 422 });
  }

  const user = await createUser(parsed.data);

  return NextResponse.json(
    { data: user },
    {
      status: 201,
      headers: { Location: `/api/v1/users/${user.id}` },
    },
  );
}
```

### Python (Django REST Framework)

```python
from rest_framework import serializers, viewsets, status
from rest_framework.response import Response

class CreateUserSerializer(serializers.Serializer):
    email = serializers.EmailField()
    name = serializers.CharField(max_length=100)

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ["id", "email", "name", "created_at"]

class UserViewSet(viewsets.ModelViewSet):
    serializer_class = UserSerializer
    permission_classes = [IsAuthenticated]

    def get_serializer_class(self):
        if self.action == "create":
            return CreateUserSerializer
        return UserSerializer

    def create(self, request):
        serializer = CreateUserSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = UserService.create(**serializer.validated_data)
        return Response(
            {"data": UserSerializer(user).data},
            status=status.HTTP_201_CREATED,
            headers={"Location": f"/api/v1/users/{user.id}"},
        )
```

### Go (net/http)

```go
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeError(w, http.StatusBadRequest, "invalid_json", "Corpo da solicitação inválido")
        return
    }

    if err := req.Validate(); err != nil {
        writeError(w, http.StatusUnprocessableEntity, "validation_error", err.Error())
        return
    }

    user, err := h.service.Create(r.Context(), req)
    if err != nil {
        switch {
        case errors.Is(err, domain.ErrEmailTaken):
            writeError(w, http.StatusConflict, "email_taken", "E-mail já registrado")
        default:
            writeError(w, http.StatusInternalServerError, "internal_error", "Erro interno")
        }
        return
    }

    w.Header().Set("Location", fmt.Sprintf("/api/v1/users/%s", user.ID))
    writeJSON(w, http.StatusCreated, map[string]any{"data": user})
}
```

## Checklist de Design de API

Antes de lançar um novo endpoint:

- [ ] A URL do recurso segue as convenções de nomenclatura (plural, kebab-case, sem verbos)
- [ ] Método HTTP correto utilizado (GET para leituras, POST para criações, etc.)
- [ ] Códigos de status apropriados retornados (não 200 para tudo)
- [ ] Entrada validada com esquema (Zod, Pydantic, Bean Validation)
- [ ] As respostas de erro seguem o formato padrão com códigos e mensagens
- [ ] Paginação implementada para endpoints de lista (cursor ou deslocamento)
- [ ] Autenticação obrigatória (ou explicitamente marcada como pública)
- [ ] Autorização verificada (o usuário só pode acessar seus próprios recursos)
- [ ] Limitação de taxa (rate limiting) configurada
- [ ] A resposta não vaza detalhes internos (stack traces, erros de SQL)
- [ ] Nomenclatura consistente com os endpoints existentes (camelCase vs snake_case)
- [ ] Documentado (especificação OpenAPI/Swagger atualizada)
