# Metorial Middleware & Integration Patterns

## Advanced Middleware Components

### Connection Pool Management

```typescript
class MetorialConnectionPool {
  private connections: Map<string, MetorialAuth> = new Map();
  private connectionLimits = new Map<string, number>();
  private activeRequests = new Map<string, number>();

  constructor(private config: {
    maxConnections: number;
    idleTimeout: number;
    retryAttempts: number;
  }) {}

  async getConnection(apiKey: string): Promise<MetorialAuth> {
    const connectionId = this.hash(apiKey);

    if (this.connections.has(connectionId)) {
      return this.connections.get(connectionId)!;
    }

    if (this.connections.size >= this.config.maxConnections) {
      await this.waitForAvailableConnection();
    }

    const auth = new MetorialAuth(apiKey);
    this.connections.set(connectionId, auth);
    this.activeRequests.set(connectionId, 0);

    // Set up idle timeout
    setTimeout(() => {
      if ((this.activeRequests.get(connectionId) || 0) === 0) {
        this.connections.delete(connectionId);
        this.activeRequests.delete(connectionId);
      }
    }, this.config.idleTimeout);

    return auth;
  }

  async executeWithConnection<T>(
    apiKey: string,
    operation: (auth: MetorialAuth) => Promise<T>
  ): Promise<T> {
    const connectionId = this.hash(apiKey);
    const auth = await this.getConnection(apiKey);

    // Increment active request counter
    this.activeRequests.set(connectionId, (this.activeRequests.get(connectionId) || 0) + 1);

    try {
      return await operation(auth);
    } finally {
      // Decrement active request counter
      this.activeRequests.set(connectionId, (this.activeRequests.get(connectionId) || 0) - 1);
    }
  }

  private hash(input: string): string {
    return Buffer.from(input).toString('base64').slice(0, 16);
  }

  private async waitForAvailableConnection(): Promise<void> {
    return new Promise((resolve) => {
      const check = () => {
        if (this.connections.size < this.config.maxConnections) {
          resolve();
        } else {
          setTimeout(check, 100);
        }
      };
      check();
    });
  }
}
```

### Request/Response Middleware Pipeline

```typescript
interface MiddlewareContext {
  request: {
    method: string;
    endpoint: string;
    body?: any;
    headers: Record<string, string>;
  };
  response?: {
    status: number;
    data: any;
    headers: Record<string, string>;
  };
  metadata: Record<string, any>;
}

type Middleware = (ctx: MiddlewareContext, next: () => Promise<void>) => Promise<void>;

class MetorialMiddlewarePipeline {
  private middlewares: Middleware[] = [];

  use(middleware: Middleware): this {
    this.middlewares.push(middleware);
    return this;
  }

  async execute(
    request: MiddlewareContext['request'],
    executor: (ctx: MiddlewareContext) => Promise<any>
  ): Promise<any> {
    const ctx: MiddlewareContext = {
      request,
      metadata: {}
    };

    let index = 0;

    const next = async (): Promise<void> => {
      if (index < this.middlewares.length) {
        const middleware = this.middlewares[index++];
        await middleware(ctx, next);
      } else {
        // Execute the actual request
        const result = await executor(ctx);
        ctx.response = {
          status: 200,
          data: result,
          headers: {}
        };
      }
    };

    await next();
    return ctx.response?.data;
  }
}

// Pre-built middleware components
const loggingMiddleware: Middleware = async (ctx, next) => {
  const start = Date.now();
  console.log(`[${new Date().toISOString()}] ${ctx.request.method} ${ctx.request.endpoint}`);

  await next();

  const duration = Date.now() - start;
  console.log(`[${new Date().toISOString()}] ${ctx.response?.status} - ${duration}ms`);
};

const rateLimitMiddleware = (maxRequests: number, windowMs: number): Middleware => {
  const requests = new Map<string, number[]>();

  return async (ctx, next) => {
    const key = ctx.request.headers['authorization'] || 'anonymous';
    const now = Date.now();
    const windowStart = now - windowMs;

    if (!requests.has(key)) {
      requests.set(key, []);
    }

    const userRequests = requests.get(key)!;
    const recentRequests = userRequests.filter(time => time > windowStart);

    if (recentRequests.length >= maxRequests) {
      throw new Error(`Rate limit exceeded: ${maxRequests} requests per ${windowMs}ms`);
    }

    recentRequests.push(now);
    requests.set(key, recentRequests);

    await next();
  };
};

const cachingMiddleware = (ttl: number = 300000): Middleware => {
  const cache = new Map<string, { data: any; expires: number }>();

  return async (ctx, next) => {
    if (ctx.request.method !== 'GET') {
      await next();
      return;
    }

    const cacheKey = `${ctx.request.endpoint}:${JSON.stringify(ctx.request.body)}`;
    const cached = cache.get(cacheKey);

    if (cached && Date.now() < cached.expires) {
      ctx.response = {
        status: 200,
        data: cached.data,
        headers: { 'X-Cache': 'HIT' }
      };
      return;
    }

    await next();

    if (ctx.response?.status === 200) {
      cache.set(cacheKey, {
        data: ctx.response.data,
        expires: Date.now() + ttl
      });
    }
  };
};
```

### Advanced Error Recovery

```typescript
class MetorialCircuitBreaker {
  private failures = 0;
  private lastFailureTime = 0;
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';

  constructor(
    private threshold: number = 5,
    private timeout: number = 60000,
    private resetTimeout: number = 30000
  ) {}

  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.resetTimeout) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const result = await Promise.race([
        operation(),
        new Promise<never>((_, reject) =>
          setTimeout(() => reject(new Error('Operation timeout')), this.timeout)
        )
      ]);

      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.failures = 0;
    this.state = 'CLOSED';
  }

  private onFailure(): void {
    this.failures++;
    this.lastFailureTime = Date.now();

    if (this.failures >= this.threshold) {
      this.state = 'OPEN';
    }
  }

  getState(): string {
    return this.state;
  }
}

class MetorialRetryStrategy {
  constructor(
    private maxAttempts: number = 3,
    private baseDelay: number = 1000,
    private maxDelay: number = 30000,
    private backoffFactor: number = 2
  ) {}

  async execute<T>(operation: () => Promise<T>): Promise<T> {
    let lastError: Error;

    for (let attempt = 1; attempt <= this.maxAttempts; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error as Error;

        if (attempt === this.maxAttempts) {
          break;
        }

        if (this.shouldNotRetry(error)) {
          throw error;
        }

        const delay = Math.min(
          this.baseDelay * Math.pow(this.backoffFactor, attempt - 1),
          this.maxDelay
        );

        await this.delay(delay);
      }
    }

    throw lastError!;
  }

  private shouldNotRetry(error: any): boolean {
    if (error instanceof MetorialError) {
      // Don't retry client errors (4xx)
      return error.status >= 400 && error.status < 500;
    }
    return false;
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

## Framework Integrations

### Express.js Advanced Integration

```typescript
import express from 'express';
import { MetorialConnectionPool, MetorialMiddlewarePipeline } from './middleware';

class MetorialExpressPlugin {
  private pool: MetorialConnectionPool;
  private pipeline: MetorialMiddlewarePipeline;

  constructor() {
    this.pool = new MetorialConnectionPool({
      maxConnections: 10,
      idleTimeout: 300000,
      retryAttempts: 3
    });

    this.pipeline = new MetorialMiddlewarePipeline()
      .use(loggingMiddleware)
      .use(rateLimitMiddleware(100, 60000))
      .use(cachingMiddleware(300000));
  }

  middleware() {
    return async (req: express.Request, res: express.Response, next: express.NextFunction) => {
      const apiKey = req.headers['x-metorial-key'] as string;

      if (!apiKey) {
        return res.status(401).json({ error: 'Missing Metorial API key' });
      }

      try {
        req.metorial = {
          execute: async (operation: string, params: any) => {
            return this.pipeline.execute(
              {
                method: 'POST',
                endpoint: `/operations/${operation}`,
                body: params,
                headers: { authorization: `Bearer ${apiKey}` }
              },
              async (ctx) => {
                return this.pool.executeWithConnection(apiKey, async (auth) => {
                  switch (operation) {
                    case 'store':
                      return auth.makeRequest('/memories', {
                        method: 'POST',
                        body: JSON.stringify(params)
                      });
                    case 'search':
                      return auth.makeRequest(`/memories/search?${new URLSearchParams(params)}`);
                    default:
                      throw new Error(`Unknown operation: ${operation}`);
                  }
                });
              }
            );
          }
        };

        next();
      } catch (error) {
        res.status(500).json({
          error: 'Failed to initialize Metorial',
          details: error instanceof Error ? error.message : 'Unknown error'
        });
      }
    };
  }
}

// Usage
const app = express();
const metorialPlugin = new MetorialExpressPlugin();

app.use('/api', metorialPlugin.middleware());

app.post('/api/memories', async (req, res) => {
  try {
    const result = await req.metorial.execute('store', req.body);
    res.json(result);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

### Next.js API Route Integration

```typescript
// pages/api/metorial/[...operation].ts
import { NextApiRequest, NextApiResponse } from 'next';

class NextJSMetorialHandler {
  private static instance: NextJSMetorialHandler;
  private pool: MetorialConnectionPool;

  private constructor() {
    this.pool = new MetorialConnectionPool({
      maxConnections: 5,
      idleTimeout: 180000,
      retryAttempts: 2
    });
  }

  static getInstance(): NextJSMetorialHandler {
    if (!this.instance) {
      this.instance = new NextJSMetorialHandler();
    }
    return this.instance;
  }

  async handleRequest(req: NextApiRequest, res: NextApiResponse) {
    const { operation } = req.query;
    const apiKey = req.headers['authorization']?.replace('Bearer ', '');

    if (!apiKey) {
      return res.status(401).json({ error: 'Missing API key' });
    }

    try {
      const result = await this.pool.executeWithConnection(apiKey, async (auth) => {
        switch (operation[0]) {
          case 'memories':
            if (req.method === 'POST') {
              return auth.makeRequest('/memories', {
                method: 'POST',
                body: JSON.stringify(req.body)
              });
            } else if (req.method === 'GET') {
              const params = new URLSearchParams(req.query as any);
              return auth.makeRequest(`/memories/search?${params}`);
            }
            break;

          case 'sessions':
            return auth.makeRequest(`/sessions/${operation[1] || ''}`, {
              method: req.method,
              body: req.method !== 'GET' ? JSON.stringify(req.body) : undefined
            });

          default:
            throw new Error(`Unsupported operation: ${operation[0]}`);
        }
      });

      res.json(result);
    } catch (error) {
      res.status(500).json({
        error: error instanceof Error ? error.message : 'Unknown error'
      });
    }
  }
}

export default function handler(req: NextApiRequest, res: NextApiResponse) {
  return NextJSMetorialHandler.getInstance().handleRequest(req, res);
}
```

### FastAPI Integration

```python
from fastapi import FastAPI, HTTPException, Depends, Header
from typing import Optional, Dict, Any
import httpx
import asyncio
from contextlib import asynccontextmanager

class MetorialFastAPIClient:
    def __init__(self, base_url: str = "https://api.metorial.com"):
        self.base_url = base_url
        self.client_pool = {}
        self.semaphore = asyncio.Semaphore(10)  # Max 10 concurrent requests

    async def get_client(self, api_key: str) -> httpx.AsyncClient:
        if api_key not in self.client_pool:
            self.client_pool[api_key] = httpx.AsyncClient(
                base_url=self.base_url,
                headers={"Authorization": f"Bearer {api_key}"},
                timeout=30.0
            )
        return self.client_pool[api_key]

    async def make_request(
        self,
        api_key: str,
        method: str,
        endpoint: str,
        **kwargs
    ) -> Dict[Any, Any]:
        async with self.semaphore:
            client = await self.get_client(api_key)
            response = await client.request(method, endpoint, **kwargs)

            if response.status_code >= 400:
                raise HTTPException(
                    status_code=response.status_code,
                    detail=f"Metorial API error: {response.text}"
                )

            return response.json()

    async def cleanup(self):
        for client in self.client_pool.values():
            await client.aclose()

# Global client instance
metorial_client = MetorialFastAPIClient()

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    yield
    # Shutdown
    await metorial_client.cleanup()

app = FastAPI(lifespan=lifespan)

async def get_api_key(authorization: Optional[str] = Header(None)) -> str:
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing or invalid API key")
    return authorization[7:]  # Remove "Bearer " prefix

@app.post("/memories")
async def store_memory(
    memory_data: Dict[str, Any],
    api_key: str = Depends(get_api_key)
):
    return await metorial_client.make_request(
        api_key, "POST", "/memories", json=memory_data
    )

@app.get("/memories/search")
async def search_memories(
    q: str,
    tags: Optional[str] = None,
    limit: int = 10,
    api_key: str = Depends(get_api_key)
):
    params = {"q": q, "limit": limit}
    if tags:
        params["tags"] = tags

    return await metorial_client.make_request(
        api_key, "GET", "/memories/search", params=params
    )
```

### GraphQL Integration

```typescript
import { ApolloServer } from '@apollo/server';
import { startStandaloneServer } from '@apollo/server/standalone';

const typeDefs = `#graphql
  type Memory {
    id: ID!
    content: String!
    tags: [String!]!
    metadata: JSON
    createdAt: String!
    updatedAt: String!
  }

  type Query {
    memories(query: String!, tags: [String!], limit: Int): [Memory!]!
    memory(id: ID!): Memory
  }

  type Mutation {
    storeMemory(content: String!, tags: [String!], metadata: JSON): Memory!
    updateMemory(id: ID!, content: String, tags: [String!], metadata: JSON): Memory!
    deleteMemory(id: ID!): Boolean!
  }

  scalar JSON
`;

interface Context {
  metorial: MetorialAuth;
}

const resolvers = {
  Query: {
    memories: async (
      _: any,
      { query, tags, limit = 10 }: { query: string; tags?: string[]; limit?: number },
      { metorial }: Context
    ) => {
      const params = new URLSearchParams({ q: query, limit: limit.toString() });
      if (tags?.length) {
        params.append('tags', tags.join(','));
      }

      return metorial.makeRequest(`/memories/search?${params}`);
    },

    memory: async (_: any, { id }: { id: string }, { metorial }: Context) => {
      return metorial.makeRequest(`/memories/${id}`);
    },
  },

  Mutation: {
    storeMemory: async (
      _: any,
      { content, tags, metadata }: { content: string; tags: string[]; metadata?: any },
      { metorial }: Context
    ) => {
      return metorial.makeRequest('/memories', {
        method: 'POST',
        body: JSON.stringify({ content, tags, metadata })
      });
    },

    updateMemory: async (
      _: any,
      { id, content, tags, metadata }: { id: string; content?: string; tags?: string[]; metadata?: any },
      { metorial }: Context
    ) => {
      const updates: any = {};
      if (content !== undefined) updates.content = content;
      if (tags !== undefined) updates.tags = tags;
      if (metadata !== undefined) updates.metadata = metadata;

      return metorial.makeRequest(`/memories/${id}`, {
        method: 'PATCH',
        body: JSON.stringify(updates)
      });
    },

    deleteMemory: async (_: any, { id }: { id: string }, { metorial }: Context) => {
      await metorial.makeRequest(`/memories/${id}`, { method: 'DELETE' });
      return true;
    },
  },
};

const server = new ApolloServer({
  typeDefs,
  resolvers,
});

const { url } = await startStandaloneServer(server, {
  listen: { port: 4000 },
  context: async ({ req }) => {
    const apiKey = req.headers.authorization?.replace('Bearer ', '');
    if (!apiKey) {
      throw new Error('Missing API key');
    }

    return {
      metorial: new MetorialAuth(apiKey)
    };
  },
});
```

## Performance Monitoring & Observability

```typescript
class MetorialMetrics {
  private metrics = {
    requestCount: 0,
    errorCount: 0,
    averageResponseTime: 0,
    cacheHitRate: 0,
    activeConnections: 0
  };

  private responseTimes: number[] = [];
  private cacheStats = { hits: 0, misses: 0 };

  incrementRequest(): void {
    this.metrics.requestCount++;
  }

  incrementError(): void {
    this.metrics.errorCount++;
  }

  recordResponseTime(time: number): void {
    this.responseTimes.push(time);
    if (this.responseTimes.length > 100) {
      this.responseTimes.shift();
    }

    this.metrics.averageResponseTime =
      this.responseTimes.reduce((a, b) => a + b, 0) / this.responseTimes.length;
  }

  recordCacheHit(): void {
    this.cacheStats.hits++;
    this.updateCacheHitRate();
  }

  recordCacheMiss(): void {
    this.cacheStats.misses++;
    this.updateCacheHitRate();
  }

  private updateCacheHitRate(): void {
    const total = this.cacheStats.hits + this.cacheStats.misses;
    this.metrics.cacheHitRate = total > 0 ? this.cacheStats.hits / total : 0;
  }

  getMetrics() {
    return { ...this.metrics };
  }

  reset(): void {
    this.metrics = {
      requestCount: 0,
      errorCount: 0,
      averageResponseTime: 0,
      cacheHitRate: 0,
      activeConnections: 0
    };
    this.responseTimes = [];
    this.cacheStats = { hits: 0, misses: 0 };
  }
}

const metricsMiddleware = (metrics: MetorialMetrics): Middleware => {
  return async (ctx, next) => {
    const start = Date.now();
    metrics.incrementRequest();

    try {
      await next();
      const responseTime = Date.now() - start;
      metrics.recordResponseTime(responseTime);
    } catch (error) {
      metrics.incrementError();
      throw error;
    }
  };
};
```

This middleware framework provides a robust foundation for building production-ready Metorial integrations with proper error handling, connection management, and observability.