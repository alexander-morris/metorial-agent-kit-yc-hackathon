# Metorial MCP Integration Guide

## Overview

This guide provides comprehensive instructions for integrating Metorial with the Model Context Protocol (MCP) to enable AI-powered memory management and retrieval in your development stack.

## What is Metorial?

Metorial (https://metorial.com) is a memory management platform that provides:
- RESTful API for memory storage and retrieval
- Flexible authentication with publishable and secret API keys
- Resource management for instances, files, sessions, and more
- Built-in rate limiting and error handling
- JavaScript SDK for multiple environments (browser, Node.js, Deno)

## What is MCP?

Model Context Protocol (MCP) is an open standard developed by Anthropic that enables seamless integration between LLM applications and external data sources. MCP consists of:
- **MCP Clients**: AI applications that access external systems
- **MCP Servers**: Provide prompts, resources, and tools for language models

## Integration Architecture

```
Your Application
       ↓
  MCP Client
       ↓
  MCP Server (Custom Metorial Integration)
       ↓
  Metorial API (https://api.metorial.com)
```

## Prerequisites

1. **Metorial Account & API Keys**
   - Sign up at https://metorial.com
   - Generate API keys:
     - Publishable key (`metorial_pk_*`) for client-side operations
     - Secret key (`metorial_sk_*`) for server-side operations

2. **Development Environment**
   - Node.js 18+ or compatible runtime
   - TypeScript support (recommended)
   - Docker (optional, for containerized deployment)

## Step 1: Create Metorial MCP Server

### 1.1 Project Setup

```bash
mkdir metorial-mcp-server
cd metorial-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk-javascript
npm install -D typescript @types/node
```

### 1.2 Environment Configuration

Create `.env` file:
```env
METORIAL_SECRET_KEY=metorial_sk_your_secret_key_here
METORIAL_PUBLISHABLE_KEY=metorial_pk_your_publishable_key_here
METORIAL_BASE_URL=https://api.metorial.com
```

### 1.3 TypeScript Configuration

Create `tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "node",
    "outDir": "./build",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "build"]
}
```

## Step 2: Implement MCP Server

### 2.1 Core Server Implementation

Create `src/index.ts`:
```typescript
import { Server } from '@modelcontextprotocol/sdk-javascript/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk-javascript/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  Tool,
} from '@modelcontextprotocol/sdk-javascript/types.js';

class MetorialMCPServer {
  private server: Server;
  private apiKey: string;
  private baseUrl: string;

  constructor() {
    this.apiKey = process.env.METORIAL_SECRET_KEY!;
    this.baseUrl = process.env.METORIAL_BASE_URL || 'https://api.metorial.com';

    this.server = new Server(
      {
        name: 'metorial-mcp-server',
        version: '1.0.0',
      },
      {
        capabilities: {
          tools: {},
        },
      }
    );

    this.setupHandlers();
  }

  private setupHandlers() {
    this.server.setRequestHandler(ListToolsRequestSchema, async () => {
      return {
        tools: [
          {
            name: 'store_memory',
            description: 'Store information in Metorial memory',
            inputSchema: {
              type: 'object',
              properties: {
                content: {
                  type: 'string',
                  description: 'Content to store in memory',
                },
                tags: {
                  type: 'array',
                  items: { type: 'string' },
                  description: 'Tags to categorize the memory',
                },
                metadata: {
                  type: 'object',
                  description: 'Additional metadata for the memory',
                },
              },
              required: ['content'],
            },
          },
          {
            name: 'retrieve_memory',
            description: 'Retrieve memories from Metorial',
            inputSchema: {
              type: 'object',
              properties: {
                query: {
                  type: 'string',
                  description: 'Search query for retrieving memories',
                },
                tags: {
                  type: 'array',
                  items: { type: 'string' },
                  description: 'Filter by specific tags',
                },
                limit: {
                  type: 'number',
                  description: 'Maximum number of results to return',
                  default: 10,
                },
              },
              required: ['query'],
            },
          },
          {
            name: 'list_sessions',
            description: 'List available Metorial sessions',
            inputSchema: {
              type: 'object',
              properties: {
                limit: {
                  type: 'number',
                  description: 'Maximum number of sessions to return',
                  default: 20,
                },
              },
            },
          },
        ],
      };
    });

    this.server.setRequestHandler(CallToolRequestSchema, async (request) => {
      const { name, arguments: args } = request.params;

      try {
        switch (name) {
          case 'store_memory':
            return await this.storeMemory(args);
          case 'retrieve_memory':
            return await this.retrieveMemory(args);
          case 'list_sessions':
            return await this.listSessions(args);
          default:
            throw new Error(`Unknown tool: ${name}`);
        }
      } catch (error) {
        return {
          content: [
            {
              type: 'text',
              text: `Error: ${error instanceof Error ? error.message : 'Unknown error'}`,
            },
          ],
        };
      }
    });
  }

  private async storeMemory(args: any) {
    const response = await fetch(`${this.baseUrl}/memories`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        content: args.content,
        tags: args.tags || [],
        metadata: args.metadata || {},
      }),
    });

    if (!response.ok) {
      throw new Error(`Failed to store memory: ${response.statusText}`);
    }

    const result = await response.json();
    return {
      content: [
        {
          type: 'text',
          text: `Memory stored successfully with ID: ${result.id}`,
        },
      ],
    };
  }

  private async retrieveMemory(args: any) {
    const params = new URLSearchParams({
      q: args.query,
      limit: args.limit?.toString() || '10',
    });

    if (args.tags?.length > 0) {
      params.append('tags', args.tags.join(','));
    }

    const response = await fetch(`${this.baseUrl}/memories/search?${params}`, {
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
      },
    });

    if (!response.ok) {
      throw new Error(`Failed to retrieve memories: ${response.statusText}`);
    }

    const results = await response.json();
    return {
      content: [
        {
          type: 'text',
          text: `Found ${results.length} memories:\n\n${results.map((mem: any) =>
            `ID: ${mem.id}\nContent: ${mem.content}\nTags: ${mem.tags?.join(', ') || 'None'}\n---`
          ).join('\n')}`,
        },
      ],
    };
  }

  private async listSessions(args: any) {
    const response = await fetch(`${this.baseUrl}/sessions?limit=${args.limit || 20}`, {
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
      },
    });

    if (!response.ok) {
      throw new Error(`Failed to list sessions: ${response.statusText}`);
    }

    const sessions = await response.json();
    return {
      content: [
        {
          type: 'text',
          text: `Active sessions:\n${sessions.map((session: any) =>
            `- ${session.id}: ${session.name || 'Unnamed'} (${session.created_at})`
          ).join('\n')}`,
        },
      ],
    };
  }

  async run() {
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
    console.error('Metorial MCP server running on stdio');
  }
}

const server = new MetorialMCPServer();
server.run().catch(console.error);
```

### 2.2 Build Scripts

Add to `package.json`:
```json
{
  "scripts": {
    "build": "tsc",
    "start": "node build/index.js",
    "dev": "tsc --watch"
  }
}
```

## Step 3: Client Integration

### 3.1 MCP Client Configuration

For Claude Desktop or other MCP clients, add to configuration:

```json
{
  "mcpServers": {
    "metorial": {
      "command": "node",
      "args": ["/path/to/your/metorial-mcp-server/build/index.js"],
      "env": {
        "METORIAL_SECRET_KEY": "metorial_sk_your_key_here"
      }
    }
  }
}
```

### 3.2 Programmatic Client Usage

```typescript
import { Client } from '@modelcontextprotocol/sdk-javascript/client/index.js';
import { StdioClientTransport } from '@modelcontextprotocol/sdk-javascript/client/stdio.js';

async function connectToMetorial() {
  const transport = new StdioClientTransport({
    command: 'node',
    args: ['build/index.js'],
  });

  const client = new Client(
    {
      name: 'metorial-client',
      version: '1.0.0',
    },
    {
      capabilities: {},
    }
  );

  await client.connect(transport);

  // Store a memory
  const storeResult = await client.request(
    {
      method: 'tools/call',
      params: {
        name: 'store_memory',
        arguments: {
          content: 'Important project meeting notes from today',
          tags: ['meeting', 'project', 'notes'],
          metadata: { date: new Date().toISOString() }
        },
      },
    }
  );

  // Retrieve memories
  const retrieveResult = await client.request(
    {
      method: 'tools/call',
      params: {
        name: 'retrieve_memory',
        arguments: {
          query: 'meeting notes',
          tags: ['meeting'],
          limit: 5
        },
      },
    }
  );

  console.log('Retrieved memories:', retrieveResult);
}
```

## Step 4: Deployment Options

### 4.1 Local Development

```bash
npm run build
npm start
```

### 4.2 Docker Deployment

Create `Dockerfile`:
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY build/ ./build/
COPY .env ./

EXPOSE 3000
CMD ["npm", "start"]
```

Build and run:
```bash
docker build -t metorial-mcp-server .
docker run --env-file .env metorial-mcp-server
```

## Step 5: Advanced Configuration

### 5.1 Rate Limiting Compliance

```typescript
class RateLimiter {
  private requests: number[] = [];
  private readonly maxRequests: number;
  private readonly windowMs: number;

  constructor(maxRequests = 1000, windowMs = 10000) {
    this.maxRequests = maxRequests;
    this.windowMs = windowMs;
  }

  async checkLimit(): Promise<boolean> {
    const now = Date.now();
    this.requests = this.requests.filter(time => now - time < this.windowMs);

    if (this.requests.length >= this.maxRequests) {
      return false;
    }

    this.requests.push(now);
    return true;
  }
}
```

### 5.2 Error Handling & Retry Logic

```typescript
async function withRetry<T>(
  operation: () => Promise<T>,
  maxRetries = 3,
  delay = 1000
): Promise<T> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await operation();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, delay * Math.pow(2, i)));
    }
  }
  throw new Error('Max retries exceeded');
}
```

## Security Best Practices

1. **API Key Management**
   - Never commit API keys to version control
   - Use environment variables or secure key management systems
   - Rotate keys regularly

2. **Input Validation**
   - Sanitize all user inputs
   - Validate against schema before API calls
   - Implement content filtering if needed

3. **Rate Limiting**
   - Respect Metorial's rate limits (1000/10s general, 5000/10min production)
   - Implement client-side rate limiting
   - Use exponential backoff for retries

## Troubleshooting

### Common Issues

1. **Authentication Errors**
   - Verify API key format and validity
   - Check environment variable loading
   - Ensure correct key type (secret vs publishable)

2. **Rate Limit Exceeded**
   - Implement request queuing
   - Add delays between requests
   - Monitor usage patterns

3. **Connection Issues**
   - Verify base URL configuration
   - Check network connectivity
   - Validate SSL/TLS settings

### Debug Mode

Enable debug logging:
```typescript
if (process.env.DEBUG === 'true') {
  console.log('Request:', { method, url, headers, body });
  console.log('Response:', { status, headers, body });
}
```

## Resources

- [Metorial API Documentation](https://metorial.com/api)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/specification)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [Metorial Marketplace](https://marketplace.metorial.com)

## Next Steps

1. Customize the MCP server for your specific use cases
2. Implement additional Metorial endpoints as needed
3. Add monitoring and analytics
4. Scale deployment based on usage patterns
5. Integrate with your existing AI/LLM applications

This integration enables powerful memory-augmented AI applications that can persistently store and retrieve context across sessions, making your AI assistants more capable and context-aware.