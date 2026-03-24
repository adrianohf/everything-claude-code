---
name: backend-patterns
description: Padrões de arquitetura de backend, design de API, otimização de banco de dados e melhores práticas no lado do servidor para Node.js, Express e rotas de API Next.js.
origin: ECC
---

# Padrões de Desenvolvimento Backend

Padrões de arquitetura de backend e melhores práticas para aplicações escaláveis no lado do servidor.

## Quando Ativar

- Projetando endpoints de API REST ou GraphQL
- Implementando camadas de repositório (repository), serviço (service) ou controlador (controller)
- Otimizando consultas de banco de dados (N+1, indexação, pool de conexões)
- Adicionando cache (Redis, em memória, cabeçalhos de cache HTTP)
- Configurando tarefas em segundo plano ou processamento assíncrono
- Estruturando o tratamento de erros e validação para APIs
- Construindo middlewares (autenticação, log, limitação de taxa)

## Padrões de Design de API

### Estrutura de API RESTful

```typescript
// ✅ URLs baseadas em recursos
GET    /api/markets                 # Listar recursos
GET    /api/markets/:id             # Obter um único recurso
POST   /api/markets                 # Criar recurso
PUT    /api/markets/:id             # Substituir recurso
PATCH  /api/markets/:id             # Atualizar recurso
DELETE /api/markets/:id             # Excluir recurso

// ✅ Parâmetros de consulta para filtragem, ordenação, paginação
GET /api/markets?status=active&sort=volume&limit=20&offset=0
```

### Padrão de Repositório (Repository Pattern)

```typescript
// Abstrair a lógica de acesso a dados
interface MarketRepository {
  findAll(filters?: MarketFilters): Promise<Market[]>
  findById(id: string): Promise<Market | null>
  create(data: CreateMarketDto): Promise<Market>
  update(id: string, data: UpdateMarketDto): Promise<Market>
  delete(id: string): Promise<void>
}

class SupabaseMarketRepository implements MarketRepository {
  async findAll(filters?: MarketFilters): Promise<Market[]> {
    let query = supabase.from('markets').select('*')

    if (filters?.status) {
      query = query.eq('status', filters.status)
    }

    if (filters?.limit) {
      query = query.limit(filters.limit)
    }

    const { data, error } = await query

    if (error) throw new Error(error.message)
    return data
  }

  // Outros métodos...
}
```

### Padrão de Camada de Serviço (Service Layer Pattern)

```typescript
// Lógica de negócio separada do acesso a dados
class MarketService {
  constructor(private marketRepo: MarketRepository) {}

  async searchMarkets(query: string, limit: number = 10): Promise<Market[]> {
    // Lógica de negócio
    const embedding = await generateEmbedding(query)
    const results = await this.vectorSearch(embedding, limit)

    // Buscar dados completos
    const markets = await this.marketRepo.findByIds(results.map(r => r.id))

    // Ordenar por similaridade
    return markets.sort((a, b) => {
      const scoreA = results.find(r => r.id === a.id)?.score || 0
      const scoreB = results.find(r => r.id === b.id)?.score || 0
      return scoreA - scoreB
    })
  }

  private async vectorSearch(embedding: number[], limit: number) {
    // Implementação de busca vetorial
  }
}
```

### Padrão de Middleware (Middleware Pattern)

```typescript
// Pipeline de processamento de solicitação/resposta
export function withAuth(handler: NextApiHandler): NextApiHandler {
  return async (req, res) => {
    const token = req.headers.authorization?.replace('Bearer ', '')

    if (!token) {
      return res.status(401).json({ error: 'Não autorizado' })
    }

    try {
      const user = await verifyToken(token)
      req.user = user
      return handler(req, res)
    } catch (error) {
      return res.status(401).json({ error: 'Token inválido' })
    }
  }
}

// Uso
export default withAuth(async (req, res) => {
  // O manipulador (handler) tem acesso a req.user
})
```

## Padrões de Banco de Dados

### Otimização de Consultas

```typescript
// ✅ BOM: Selecionar apenas as colunas necessárias
const { data } = await supabase
  .from('markets')
  .select('id, name, status, volume')
  .eq('status', 'active')
  .order('volume', { ascending: false })
  .limit(10)

// ❌ RUIM: Selecionar tudo
const { data } = await supabase
  .from('markets')
  .select('*')
```

### Prevenção de Consulta N+1

```typescript
// ❌ RUIM: Problema de consulta N+1
const markets = await getMarkets()
for (const market of markets) {
  market.creator = await getUser(market.creator_id)  // N consultas
}

// ✅ BOM: Busca em lote (Batch fetch)
const markets = await getMarkets()
const creatorIds = markets.map(m => m.creator_id)
const creators = await getUsers(creatorIds)  // 1 consulta
const creatorMap = new Map(creators.map(c => [c.id, c]))

markets.forEach(market => {
  market.creator = creatorMap.get(market.creator_id)
})
```

### Padrão de Transação (Transaction Pattern)

```typescript
async function createMarketWithPosition(
  marketData: CreateMarketDto,
  positionData: CreatePositionDto
) {
  // Usar transação Supabase
  const { data, error } = await supabase.rpc('create_market_with_position', {
    market_data: marketData,
    position_data: positionData
  })

  if (error) throw new Error('Falha na transação')
  return data
}

// Função SQL no Supabase
CREATE OR REPLACE FUNCTION create_market_with_position(
  market_data jsonb,
  position_data jsonb
)
RETURNS jsonb
LANGUAGE plpgsql
AS $$
BEGIN
  -- Inicia a transação automaticamente
  INSERT INTO markets VALUES (market_data);
  INSERT INTO positions VALUES (position_data);
  RETURN jsonb_build_object('success', true);
EXCEPTION
  WHEN OTHERS THEN
    -- O rollback acontece automaticamente
    RETURN jsonb_build_object('success', false, 'error', SQLERRM);
END;
$$;
```

## Estratégias de Cache

### Camada de Cache Redis

```typescript
class CachedMarketRepository implements MarketRepository {
  constructor(
    private baseRepo: MarketRepository,
    private redis: RedisClient
  ) {}

  async findById(id: string): Promise<Market | null> {
    // Verificar cache primeiro
    const cached = await this.redis.get(`market:${id}`)

    if (cached) {
      return JSON.parse(cached)
    }

    // Cache miss - buscar no banco de dados
    const market = await this.baseRepo.findById(id)

    if (market) {
      // Armazenar em cache por 5 minutos
      await this.redis.setex(`market:${id}`, 300, JSON.stringify(market))
    }

    return market
  }

  async invalidateCache(id: string): Promise<void> {
    await this.redis.del(`market:${id}`)
  }
}
```

### Padrão Cache-Aside

```typescript
async function getMarketWithCache(id: string): Promise<Market> {
  const cacheKey = `market:${id}`

  // Tentar cache
  const cached = await redis.get(cacheKey)
  if (cached) return JSON.parse(cached)

  // Cache miss - buscar no DB
  const market = await db.markets.findUnique({ where: { id } })

  if (!market) throw new Error('Mercado não encontrado')

  // Atualizar cache
  await redis.setex(cacheKey, 300, JSON.stringify(market))

  return market
}
```

## Padrões de Tratamento de Erros

### Manipulador de Erros Centralizado

```typescript
class ApiError extends Error {
  constructor(
    public statusCode: number,
    public message: string,
    public isOperational = true
  ) {
    super(message)
    Object.setPrototypeOf(this, ApiError.prototype)
  }
}

export function errorHandler(error: unknown, req: Request): Response {
  if (error instanceof ApiError) {
    return NextResponse.json({
      success: false,
      error: error.message
    }, { status: error.statusCode })
  }

  if (error instanceof z.ZodError) {
    return NextResponse.json({
      success: false,
      error: 'A validação falhou',
      details: error.errors
    }, { status: 400 })
  }

  // Log de erros inesperados
  console.error('Erro inesperado:', error)

  return NextResponse.json({
    success: false,
    error: 'Erro interno do servidor'
  }, { status: 500 })
}

// Uso
export async function GET(request: Request) {
  try {
    const data = await fetchData()
    return NextResponse.json({ success: true, data })
  } catch (error) {
    return errorHandler(error, request)
  }
}
```

### Tentativa com Exponential Backoff (Retentativa)

```typescript
async function fetchWithRetry<T>(
  fn: () => Promise<T>,
  maxRetries = 3
): Promise<T> {
  let lastError: Error

  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error as Error

      if (i < maxRetries - 1) {
        // Exponential backoff: 1s, 2s, 4s
        const delay = Math.pow(2, i) * 1000
        await new Promise(resolve => setTimeout(resolve, delay))
      }
    }
  }

  throw lastError!
}

// Uso
const data = await fetchWithRetry(() => fetchFromAPI())
```

## Autenticação e Autorização

### Validação de Token JWT

```typescript
import jwt from 'jsonwebtoken'

interface JWTPayload {
  userId: string
  email: string
  role: 'admin' | 'user'
}

export function verifyToken(token: string): JWTPayload {
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET!) as JWTPayload
    return payload
  } catch (error) {
    throw new ApiError(401, 'Token inválido')
  }
}

export async function requireAuth(request: Request) {
  const token = request.headers.get('authorization')?.replace('Bearer ', '')

  if (!token) {
    throw new ApiError(401, 'Token de autorização ausente')
  }

  return verifyToken(token)
}

// Uso na rota da API
export async function GET(request: Request) {
  const user = await requireAuth(request)

  const data = await getDataForUser(user.userId)

  return NextResponse.json({ success: true, data })
}
```

### Controle de Acesso Baseado em Função (RBAC)

```typescript
type Permission = 'read' | 'write' | 'delete' | 'admin'

interface User {
  id: string
  role: 'admin' | 'moderator' | 'user'
}

const rolePermissions: Record<User['role'], Permission[]> = {
  admin: ['read', 'write', 'delete', 'admin'],
  moderator: ['read', 'write', 'delete'],
  user: ['read', 'write']
}

export function hasPermission(user: User, permission: Permission): boolean {
  return rolePermissions[user.role].includes(permission)
}

export function requirePermission(permission: Permission) {
  return (handler: (request: Request, user: User) => Promise<Response>) => {
    return async (request: Request) => {
      const user = await requireAuth(request)

      if (!hasPermission(user, permission)) {
        throw new ApiError(403, 'Permissões insuficientes')
      }

      return handler(request, user)
    }
  }
}

// Uso - HOF envolve o manipulador (handler)
export const DELETE = requirePermission('delete')(
  async (request: Request, user: User) => {
    // O manipulador recebe o usuário autenticado com permissão verificada
    return new Response('Excluído', { status: 200 })
  }
)
```

## Limitação de Taxa (Rate Limiting)

### Limitador de Taxa Simples em Memória

```typescript
class RateLimiter {
  private requests = new Map<string, number[]>()

  async checkLimit(
    identifier: string,
    maxRequests: number,
    windowMs: number
  ): Promise<boolean> {
    const now = Date.now()
    const requests = this.requests.get(identifier) || []

    // Remover solicitações antigas fora da janela
    const recentRequests = requests.filter(time => now - time < windowMs)

    if (recentRequests.length >= maxRequests) {
      return false  // Limite de taxa excedido
    }

    // Adicionar solicitação atual
    recentRequests.push(now)
    this.requests.set(identifier, recentRequests)

    return true
  }
}

const limiter = new RateLimiter()

export async function GET(request: Request) {
  const ip = request.headers.get('x-forwarded-for') || 'unknown'

  const allowed = await limiter.checkLimit(ip, 100, 60000)  // 100 req/min

  if (!allowed) {
    return NextResponse.json({
      error: 'Limite de taxa excedido'
    }, { status: 429 })
  }

  // Continuar com a solicitação
}
```

## Tarefas em Segundo Plano e Filas

### Padrão de Fila Simples

```typescript
class JobQueue<T> {
  private queue: T[] = []
  private processing = false

  async add(job: T): Promise<void> {
    this.queue.push(job)

    if (!this.processing) {
      this.process()
    }
  }

  private async process(): Promise<void> {
    this.processing = true

    while (this.queue.length > 0) {
      const job = this.queue.shift()!

      try {
        await this.execute(job)
      } catch (error) {
        console.error('Falha na tarefa:', error)
      }
    }

    this.processing = false
  }

  private async execute(job: T): Promise<void> {
    // Lógica de execução da tarefa
  }
}

// Uso para indexação de mercados
interface IndexJob {
  marketId: string
}

const indexQueue = new JobQueue<IndexJob>()

export async function POST(request: Request) {
  const { marketId } = await request.json()

  // Adicionar à fila em vez de bloquear
  await indexQueue.add({ marketId })

  return NextResponse.json({ success: true, message: 'Tarefa enfileirada' })
}
```

## Log e Monitoramento

### Log Estruturado

```typescript
interface LogContext {
  userId?: string
  requestId?: string
  method?: string
  path?: string
  [key: string]: unknown
}

class Logger {
  log(level: 'info' | 'warn' | 'error', message: string, context?: LogContext) {
    const entry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      ...context
    }

    console.log(JSON.stringify(entry))
  }

  info(message: string, context?: LogContext) {
    this.log('info', message, context)
  }

  warn(message: string, context?: LogContext) {
    this.log('warn', message, context)
  }

  error(message: string, error: Error, context?: LogContext) {
    this.log('error', message, {
      ...context,
      error: error.message,
      stack: error.stack
    })
  }
}

const logger = new Logger()

// Uso
export async function GET(request: Request) {
  const requestId = crypto.randomUUID()

  logger.info('Buscando mercados', {
    requestId,
    method: 'GET',
    path: '/api/markets'
  })

  try {
    const markets = await fetchMarkets()
    return NextResponse.json({ success: true, data: markets })
  } catch (error) {
    logger.error('Falha ao buscar mercados', error as Error, { requestId })
    return NextResponse.json({ error: 'Erro interno' }, { status: 500 })
  }
}
```

**Lembre-se**: Os padrões de backend permitem aplicações escaláveis e fáceis de manter no lado do servidor. Escolha os padrões que melhor se adaptam ao seu nível de complexidade.
