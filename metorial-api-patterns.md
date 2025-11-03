# Metorial API Patterns & Examples

## Quick Start Examples

### Basic Memory Operations

```typescript
// Store a simple memory
const storeResponse = await fetch('https://api.metorial.com/memories', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer metorial_sk_your_key',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    content: 'User prefers dark mode and compact layout',
    tags: ['preferences', 'ui'],
    metadata: {
      userId: 'user_123',
      timestamp: new Date().toISOString(),
      source: 'settings_page'
    }
  })
});

// Search memories
const searchResponse = await fetch('https://api.metorial.com/memories/search?q=dark%20mode&tags=preferences', {
  headers: {
    'Authorization': 'Bearer metorial_sk_your_key'
  }
});

// Get specific memory
const memory = await fetch('https://api.metorial.com/memories/mem_abc123', {
  headers: {
    'Authorization': 'Bearer metorial_sk_your_key'
  }
});
```

### Authentication Patterns

```typescript
class MetorialAuth {
  private apiKey: string;
  private baseUrl = 'https://api.metorial.com';

  constructor(apiKey: string) {
    this.apiKey = apiKey;
  }

  private getHeaders(): HeadersInit {
    return {
      'Authorization': `Bearer ${this.apiKey}`,
      'Content-Type': 'application/json',
      'User-Agent': 'MyApp/1.0.0'
    };
  }

  async makeRequest(endpoint: string, options: RequestInit = {}) {
    const url = `${this.baseUrl}${endpoint}`;
    const headers = { ...this.getHeaders(), ...options.headers };

    const response = await fetch(url, {
      ...options,
      headers
    });

    if (!response.ok) {
      throw new MetorialError(response.status, await response.text());
    }

    return response.json();
  }
}

class MetorialError extends Error {
  constructor(public status: number, message: string) {
    super(`Metorial API Error ${status}: ${message}`);
  }
}
```

### Session Management

```typescript
class MetorialSession {
  private sessionId: string | null = null;
  private auth: MetorialAuth;

  constructor(auth: MetorialAuth) {
    this.auth = auth;
  }

  async createSession(name?: string): Promise<string> {
    const response = await this.auth.makeRequest('/sessions', {
      method: 'POST',
      body: JSON.stringify({
        name: name || `Session ${new Date().toISOString()}`,
        metadata: { created_by: 'client_app' }
      })
    });

    this.sessionId = response.id;
    return this.sessionId;
  }

  async storeInSession(content: string, tags: string[] = []): Promise<string> {
    if (!this.sessionId) {
      await this.createSession();
    }

    const response = await this.auth.makeRequest('/memories', {
      method: 'POST',
      body: JSON.stringify({
        content,
        tags,
        session_id: this.sessionId,
        metadata: { timestamp: Date.now() }
      })
    });

    return response.id;
  }

  async getSessionMemories(): Promise<any[]> {
    if (!this.sessionId) return [];

    return this.auth.makeRequest(`/sessions/${this.sessionId}/memories`);
  }
}
```

### File Operations

```typescript
class MetorialFiles {
  constructor(private auth: MetorialAuth) {}

  async uploadFile(file: File, tags: string[] = []): Promise<string> {
    const formData = new FormData();
    formData.append('file', file);
    formData.append('tags', JSON.stringify(tags));
    formData.append('metadata', JSON.stringify({
      originalName: file.name,
      size: file.size,
      type: file.type
    }));

    const response = await fetch('https://api.metorial.com/files', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.auth.apiKey}`
      },
      body: formData
    });

    if (!response.ok) {
      throw new Error(`Upload failed: ${response.statusText}`);
    }

    const result = await response.json();
    return result.id;
  }

  async getFileLink(fileId: string): Promise<string> {
    const response = await this.auth.makeRequest(`/files/${fileId}/link`);
    return response.url;
  }

  async searchFiles(query: string, types?: string[]): Promise<any[]> {
    const params = new URLSearchParams({ q: query });
    if (types?.length) {
      params.append('types', types.join(','));
    }

    return this.auth.makeRequest(`/files/search?${params}`);
  }
}
```

### Batch Operations

```typescript
class MetorialBatch {
  constructor(private auth: MetorialAuth) {}

  async batchStore(items: Array<{content: string, tags?: string[], metadata?: any}>): Promise<string[]> {
    const response = await this.auth.makeRequest('/memories/batch', {
      method: 'POST',
      body: JSON.stringify({
        items: items.map(item => ({
          ...item,
          timestamp: Date.now()
        }))
      })
    });

    return response.ids;
  }

  async batchRetrieve(queries: string[]): Promise<any[][]> {
    const response = await this.auth.makeRequest('/memories/batch-search', {
      method: 'POST',
      body: JSON.stringify({ queries })
    });

    return response.results;
  }

  async batchUpdate(updates: Array<{id: string, content?: string, tags?: string[]}>): Promise<void> {
    await this.auth.makeRequest('/memories/batch-update', {
      method: 'PATCH',
      body: JSON.stringify({ updates })
    });
  }
}
```

### Real-time Streaming

```typescript
class MetorialStream {
  private eventSource: EventSource | null = null;

  constructor(private auth: MetorialAuth) {}

  async streamMemories(sessionId: string, callback: (memory: any) => void): Promise<void> {
    const url = `https://api.metorial.com/sessions/${sessionId}/stream?auth=${encodeURIComponent(this.auth.apiKey)}`;

    this.eventSource = new EventSource(url);

    this.eventSource.onmessage = (event) => {
      const memory = JSON.parse(event.data);
      callback(memory);
    };

    this.eventSource.onerror = (error) => {
      console.error('Stream error:', error);
      this.reconnect(sessionId, callback);
    };
  }

  private async reconnect(sessionId: string, callback: (memory: any) => void): Promise<void> {
    setTimeout(() => {
      this.streamMemories(sessionId, callback);
    }, 5000);
  }

  disconnect(): void {
    if (this.eventSource) {
      this.eventSource.close();
      this.eventSource = null;
    }
  }
}
```

### Error Handling Patterns

```typescript
class MetorialErrorHandler {
  static async withRetry<T>(
    operation: () => Promise<T>,
    maxAttempts: number = 3,
    baseDelay: number = 1000
  ): Promise<T> {
    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
      try {
        return await operation();
      } catch (error) {
        if (attempt === maxAttempts) throw error;

        if (error instanceof MetorialError) {
          if (error.status >= 400 && error.status < 500) {
            // Client error - don't retry
            throw error;
          }
        }

        const delay = baseDelay * Math.pow(2, attempt - 1);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }

    throw new Error('Max attempts exceeded');
  }

  static handleError(error: any): never {
    if (error instanceof MetorialError) {
      switch (error.status) {
        case 401:
          throw new Error('Invalid API key or authentication failed');
        case 403:
          throw new Error('Access denied - check your permissions');
        case 429:
          throw new Error('Rate limit exceeded - please slow down requests');
        case 404:
          throw new Error('Resource not found');
        default:
          throw new Error(`API error: ${error.message}`);
      }
    }
    throw error;
  }
}
```

### Performance Optimization

```typescript
class MetorialCache {
  private cache = new Map<string, { data: any, expires: number }>();
  private readonly TTL = 5 * 60 * 1000; // 5 minutes

  set(key: string, data: any): void {
    this.cache.set(key, {
      data,
      expires: Date.now() + this.TTL
    });
  }

  get(key: string): any | null {
    const item = this.cache.get(key);
    if (!item || Date.now() > item.expires) {
      this.cache.delete(key);
      return null;
    }
    return item.data;
  }

  clear(): void {
    this.cache.clear();
  }
}

class OptimizedMetorial {
  private cache = new MetorialCache();
  private requestQueue: Array<() => Promise<any>> = [];
  private processing = false;

  constructor(private auth: MetorialAuth) {}

  async getCachedMemory(id: string): Promise<any> {
    const cached = this.cache.get(`memory:${id}`);
    if (cached) return cached;

    const memory = await this.auth.makeRequest(`/memories/${id}`);
    this.cache.set(`memory:${id}`, memory);
    return memory;
  }

  async queueRequest<T>(operation: () => Promise<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      this.requestQueue.push(async () => {
        try {
          const result = await operation();
          resolve(result);
        } catch (error) {
          reject(error);
        }
      });

      this.processQueue();
    });
  }

  private async processQueue(): Promise<void> {
    if (this.processing || this.requestQueue.length === 0) return;

    this.processing = true;

    while (this.requestQueue.length > 0) {
      const request = this.requestQueue.shift()!;
      await request();

      // Rate limiting: wait between requests
      await new Promise(resolve => setTimeout(resolve, 100));
    }

    this.processing = false;
  }
}
```

### Integration Examples

#### Express.js Middleware

```typescript
import express from 'express';

interface MetorialRequest extends express.Request {
  metorial?: MetorialSession;
}

function metorialMiddleware(apiKey: string) {
  const auth = new MetorialAuth(apiKey);

  return async (req: MetorialRequest, res: express.Response, next: express.NextFunction) => {
    try {
      const sessionId = req.headers['x-session-id'] as string;
      req.metorial = new MetorialSession(auth);

      if (sessionId) {
        req.metorial.sessionId = sessionId;
      }

      next();
    } catch (error) {
      res.status(500).json({ error: 'Failed to initialize Metorial session' });
    }
  };
}

// Usage
app.use(metorialMiddleware(process.env.METORIAL_API_KEY!));

app.post('/api/remember', async (req: MetorialRequest, res) => {
  const { content, tags } = req.body;

  try {
    const memoryId = await req.metorial!.storeInSession(content, tags);
    res.json({ success: true, memoryId });
  } catch (error) {
    res.status(500).json({ error: 'Failed to store memory' });
  }
});
```

#### React Hook

```typescript
import { useState, useEffect, useCallback } from 'react';

interface UseMetorialOptions {
  apiKey: string;
  sessionId?: string;
}

export function useMetorial({ apiKey, sessionId }: UseMetorialOptions) {
  const [auth] = useState(() => new MetorialAuth(apiKey));
  const [session] = useState(() => new MetorialSession(auth));
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (sessionId) {
      session.sessionId = sessionId;
    }
  }, [sessionId, session]);

  const storeMemory = useCallback(async (content: string, tags: string[] = []) => {
    setLoading(true);
    setError(null);

    try {
      const memoryId = await session.storeInSession(content, tags);
      return memoryId;
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error');
      throw err;
    } finally {
      setLoading(false);
    }
  }, [session]);

  const getMemories = useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      const memories = await session.getSessionMemories();
      return memories;
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error');
      throw err;
    } finally {
      setLoading(false);
    }
  }, [session]);

  return {
    storeMemory,
    getMemories,
    loading,
    error
  };
}
```

## Common Patterns

### Memory Categorization

```typescript
enum MemoryType {
  USER_PREFERENCE = 'user_preference',
  CONVERSATION = 'conversation',
  DOCUMENT = 'document',
  INTERACTION = 'interaction',
  SYSTEM_STATE = 'system_state'
}

class CategorizedMetorial {
  constructor(private session: MetorialSession) {}

  async storeUserPreference(key: string, value: any): Promise<string> {
    return this.session.storeInSession(
      JSON.stringify({ key, value }),
      [MemoryType.USER_PREFERENCE, key]
    );
  }

  async storeConversation(message: string, role: 'user' | 'assistant'): Promise<string> {
    return this.session.storeInSession(
      message,
      [MemoryType.CONVERSATION, role, `turn_${Date.now()}`]
    );
  }

  async getUserPreferences(): Promise<Record<string, any>> {
    const memories = await this.session.getSessionMemories();
    const preferences: Record<string, any> = {};

    memories
      .filter(m => m.tags.includes(MemoryType.USER_PREFERENCE))
      .forEach(m => {
        const parsed = JSON.parse(m.content);
        preferences[parsed.key] = parsed.value;
      });

    return preferences;
  }
}
```

### Semantic Search Enhancement

```typescript
class SemanticMetorial {
  constructor(private auth: MetorialAuth) {}

  async semanticSearch(query: string, options: {
    limit?: number;
    threshold?: number;
    tags?: string[];
  } = {}): Promise<any[]> {
    const params = new URLSearchParams({
      q: query,
      semantic: 'true',
      limit: (options.limit || 10).toString()
    });

    if (options.threshold) {
      params.append('threshold', options.threshold.toString());
    }

    if (options.tags?.length) {
      params.append('tags', options.tags.join(','));
    }

    return this.auth.makeRequest(`/memories/search?${params}`);
  }

  async findSimilar(memoryId: string, limit: number = 5): Promise<any[]> {
    return this.auth.makeRequest(`/memories/${memoryId}/similar?limit=${limit}`);
  }
}
```

These patterns provide a solid foundation for building robust Metorial integrations with proper error handling, caching, and performance optimization.