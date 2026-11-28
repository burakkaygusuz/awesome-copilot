---
description: 'Instructions to build Model Context Protocol (MCP) servers in TypeScript with the MCP TypeScript SDK 2.x'
applyTo: '**/*.ts, **/*.js, **/package.json, **/tsconfig.json'
---

# TypeScript MCP Server Development

These instructions apply to the [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) 2.x. SDK 2.4.x implements the MCP specification revision [2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28). A client enables it with `versionNegotiation`. By default, a client still uses the 2025 `initialize` handshake.

> **Scope:** Apply these instructions only to projects that use the MCP TypeScript SDK 2.x directly. Do not apply them to code that uses the v1 package `@modelcontextprotocol/sdk`. Update that code first with the codemod. These instructions do not cover the Hono and Fastify adapters.

## Instructions

- Use Node.js 22.12 or later on an LTS line (22, 24, or 26) with `"type": "module"`. Use a `tsconfig.json` with `module: nodenext` and `types: ["node"]`.
- Install the SDK with `npm install @modelcontextprotocol/server zod@^4.2`. The SDK does not support Zod 3.
- Do not use the v1 package `@modelcontextprotocol/sdk`. Run `npx @modelcontextprotocol/codemod@latest v1-to-v2 .` when you update a server. Then search for `@mcp-codemod-error` markers. Read the [upgrade guide](https://ts.sdk.modelcontextprotocol.io/v2/migration/upgrade-to-v2.html) for the rest.
- Import from the package root: `McpServer`, `ResourceTemplate`, `completable`, `getDisplayName`, `createMcpHandler`, `inputRequired`, and `acceptedContent` come from `@modelcontextprotocol/server`.
- Import `serveStdio` from `@modelcontextprotocol/server/stdio`. Import `toNodeHandler` from `@modelcontextprotocol/node`. Import `createMcpExpressApp` from `@modelcontextprotocol/express`.
- Use the `McpServer` class for most servers. Use the `Server` class only for low-level control. Register handlers with `server.setRequestHandler('tools/call', async (request, ctx) => { ... })`.
- Write `import * as z from 'zod/v4'`. Give `inputSchema`, `outputSchema`, and `argsSchema` as `z.object({ ... })` values. The raw shape form `{ param: z.string() }` is deprecated.
- Always provide a `title` for tools, resources, and prompts for better UI display.
- Use `registerTool()`, `registerResource()`, and `registerPrompt()`. The variadic `tool()`, `resource()`, and `prompt()` methods do not exist.
- Make a server in a factory function. Pass the factory to `serveStdio(createServer)` or to `createMcpHandler(createServer)`.
- For a local server, use `serveStdio(createServer)`.
- For a remote server, use Streamable HTTP: `toNodeHandler(createMcpHandler(createServer))`. For Express, attach it to the app that `createMcpExpressApp()` returns.
- An HTTP server is stateless. `createMcpHandler(factory)` calls the factory for each request. Keep the factory cheap. Create database pools, API clients, and caches once at module scope. Use them in the factory. There are no sessions, no `Mcp-Session-Id` header, and no `initialize` handshake on 2026-07-28.
- Do not use `sessionIdGenerator` or `NodeStreamableHTTPServerTransport` in a new server. The transport is only for 2025-era clients.
- `createMcpHandler` does not check the `Host` and `Origin` headers. `createMcpExpressApp` protects the default `127.0.0.1` bind. When you bind to all interfaces, pass `allowedHosts` and `allowedOrigins`. Do not use `enableDnsRebindingProtection`. It is deprecated.
- Each Streamable HTTP POST request needs the `MCP-Protocol-Version` and `Mcp-Method` headers. A `tools/call`, `resources/read`, or `prompts/get` request also needs `Mcp-Name`. `createMcpHandler` answers HTTP 400 with `-32020` when a header is missing or does not match the body.
- Return both `content` (for display) and `structuredContent` (for structured data) from a tool that has an `outputSchema`.
- Use `ResourceTemplate` for dynamic resources with URI parameters: `new ResourceTemplate('resource://{param}', { list: undefined })`.
- `registerResource()` always takes a metadata object. Use `{}` when you have no metadata.
- Support completions with `completable(schema, callback).optional()`. Put `.optional()` on the outside.
- Do not use `ctx.mcpReq.elicitInput()`. It throws on a 2026-07-28 request. Return `inputRequired({ inputRequests: { id: inputRequired.elicit({ message, requestedSchema }) } })`. On the retry, read the answer with `acceptedContent(ctx.mcpReq.inputResponses, 'id', schema)`. Use the same zod schema that you gave to `inputRequired.elicit`.
- For a flow in the browser, for example OAuth or a payment, use `inputRequired.elicitUrl(...)`. There is no `elicitationId`. Use `requestState` to link the retries. `requestState` comes back from the client. Treat it as untrusted input. Create one codec at module scope, outside the factory, with `createRequestStateCodec({ key })` from `@modelcontextprotocol/server`. Load a key of at least 32 bytes from the configuration. Each server process must use the same key. Pass a `bind` function that returns the user and `ctx.mcpReq.method`. Every tool call has the method `tools/call`. Thus, also put the tool name and the important arguments in the state. On the retry, reject the state if they do not match the call. Return `requestState: await codec.mint(state, ctx)` in `inputRequired(...)`. Set `requestState: { verify: codec.verify }` in the `McpServer` options. Read the state with `ctx.mcpReq.requestState<T>()`. The codec signs the state but does not encrypt it. Do not put secrets in the state.
- Do not use sampling (`server.server.createMessage()`). The revision 2026-07-28 deprecates it. Call the LLM provider API directly. If existing code must keep sampling, use `inputRequired.createMessage` on 2026-07-28 connections.
- Do not use MCP logging (`ctx.mcpReq.log`). The revision 2026-07-28 deprecates it. Log to stderr or use OpenTelemetry.
- Do not use roots. The revision 2026-07-28 deprecates them. Get paths from tool arguments, resource URIs, or the server configuration.
- Enable notification debouncing for bulk updates: `new McpServer(info, { debouncedNotificationMethods: ['notifications/tools/list_changed'] })`.
- Dynamic updates: Change the tool, prompt, or resource set only when server-wide data changes, not as a side effect of a request. On stdio, `.enable()`, `.disable()`, `.update()`, and `.remove()` on registered items send list change notifications. On HTTP, change the data that the factory reads, then call `handler.notify.toolsChanged()`, `promptsChanged()`, `resourcesChanged()`, or `resourceUpdated(uri)` on the result of `createMcpHandler`.
- Use `getDisplayName()` for UI display names.
- Test a server with the SDK `Client` from `@modelcontextprotocol/client` and `StdioClientTransport` from `@modelcontextprotocol/client/stdio`.
- Test servers manually with the MCP Inspector: `npx @modelcontextprotocol/inspector`.

## Best Practices

- Keep tool implementations focused on single responsibilities.
- Provide clear, descriptive titles and descriptions that the LLM can understand.
- Add a `.describe(...)` to each property in `inputSchema`.
- Use proper TypeScript types for all parameters and return values.
- Return `isError: true` in tool results for error conditions that the model must read. Do not throw for a tool failure.
- Use async/await for all asynchronous operations.
- Close database connections and release resources properly.
- Validate input parameters before processing.
- On stdio servers, stdout is the protocol channel: log to stderr. MCP logging (`ctx.mcpReq.log`) is deprecated (SEP-2577). Use stderr or OpenTelemetry.
- Register tools, resources, and prompts inside the factory. Do not register them on a shared instance. The tool list must not vary per connection or as a side effect of other requests. It can vary by the authorization of the request. The SDK lists tools in a deterministic order.
- Cacheable results carry `ttlMs` and `cacheScope`. The SDK uses `ttlMs: 0` and `cacheScope: 'private'` by default. Set `cacheHints` in the server options or `cacheHint` per resource. Do this only when the data is safe to cache.
- `outputSchema` and `inputSchema` can use any JSON Schema 2020-12 keyword. `structuredContent` can be any JSON value.
- Be careful when a tool gives access to the file system or the network.
- Use environment variables for configuration (for example ports and API keys).
- Document tool capabilities and limitations clearly.

## Common Patterns

The snippets use `import * as z from 'zod/v4'`. Most of them show only the imports that they need.

### Basic Server Setup (stdio)

```typescript
import { McpServer } from '@modelcontextprotocol/server';
import { serveStdio } from '@modelcontextprotocol/server/stdio';
import * as z from 'zod/v4';

function createServer(): McpServer {
    const server = new McpServer({ name: 'my-server', version: '0.1.0' });

    server.registerTool(
        'add',
        {
            title: 'Add',
            description: 'Add two numbers.',
            inputSchema: z.object({
                a: z.number().describe('First number'),
                b: z.number().describe('Second number')
            }),
            outputSchema: z.object({ sum: z.number() })
        },
        async ({ a, b }) => {
            const sum = a + b;
            return {
                content: [{ type: 'text', text: JSON.stringify({ sum }) }],
                structuredContent: { sum }
            };
        }
    );

    return server;
}

void serveStdio(createServer);
console.error('my-server is running on stdio');
```

### Tool with Structured Output

```typescript
server.registerTool(
    'word_count',
    {
        title: 'Word Count',
        description: 'Count the words in a text.',
        inputSchema: z.object({ text: z.string().describe('The text to count') }),
        outputSchema: z.object({ words: z.number().int() }),
        annotations: { readOnlyHint: true }
    },
    async ({ text }) => {
        const words = text.trim() === '' ? 0 : text.trim().split(/\s+/).length;
        return {
            content: [{ type: 'text', text: JSON.stringify({ words }) }],
            structuredContent: { words }
        };
    }
);
```

### Error Handling

```typescript
server.registerTool(
    'divide',
    {
        title: 'Divide',
        description: 'Divide a by b.',
        inputSchema: z.object({ a: z.number(), b: z.number() })
    },
    async ({ a, b }) => {
        if (b === 0) {
            return { content: [{ type: 'text', text: 'b must not be zero.' }], isError: true };
        }
        return { content: [{ type: 'text', text: String(a / b) }] };
    }
);
```

A tool handler produces only tool errors. The SDK converts an exception from a tool handler into a result with `isError: true`. Resource, prompt, and completion callbacks have no `isError` channel. They throw `ProtocolError` (from `@modelcontextprotocol/server`). The v2 error classes are `ProtocolError`, `SdkError`, and `SdkHttpError` (with `.status`).

A missing resource throws `ResourceNotFoundError`. The SDK answers with the code `-32602`. Custom error codes use a value outside `-32768` to `-32000`. Do not use the legacy codes `-32000` to `-32019` in a new server. The specification reserves `-32020` to `-32099`. `UnsupportedProtocolVersionError` (`-32022`) and `MissingRequiredClientCapabilityError` (`-32021`) are protocol errors that the SDK sends. They are not tool errors.

### Resources and Resource Templates

```typescript
import { ResourceNotFoundError, ResourceTemplate } from '@modelcontextprotocol/server';

server.registerResource(
    'config',
    'config://app',
    { title: 'App Config', description: 'Application configuration', mimeType: 'application/json' },
    async uri => ({
        contents: [{ uri: uri.href, text: '{"debug":false}' }]
    })
);

const users = new Map([['1', 'Ada']]);

server.registerResource(
    'user',
    new ResourceTemplate('users://{id}', { list: undefined }),
    { title: 'User', description: 'A user by id' },
    async (uri, { id }) => {
        const user = users.get(String(id));
        if (user === undefined) throw new ResourceNotFoundError(uri.href);
        return { contents: [{ uri: uri.href, text: `User ${user}` }] };
    }
);
```

### Prompt with Completion

```typescript
import { completable } from '@modelcontextprotocol/server';

const languages = ['typescript', 'javascript', 'python', 'rust', 'go'];

server.registerPrompt(
    'code-review',
    {
        title: 'Code Review',
        description: 'Review code in a language.',
        argsSchema: z.object({
            language: completable(z.string().describe('Programming language'), value =>
                languages.filter(language => language.startsWith(value))
            ).optional()
        })
    },
    ({ language }) => ({
        messages: [
            {
                role: 'user' as const,
                content: { type: 'text' as const, text: `Review this ${language ?? 'source'} code.` }
            }
        ]
    })
);
```

### Async Tool with Progress

```typescript
server.registerTool(
    'lookup',
    {
        title: 'Lookup',
        description: 'Look up a query in a slow backend.',
        inputSchema: z.object({ query: z.string() })
    },
    async ({ query }, ctx) => {
        const progressToken = ctx.mcpReq._meta?.progressToken;
        if (progressToken !== undefined) {
            await ctx.mcpReq.notify({
                method: 'notifications/progress',
                params: { progressToken, progress: 1, total: 2, message: 'Halfway' }
            });
        }
        const text = await slowLookup(query);
        return { content: [{ type: 'text', text }] };
    }
);
```

Pass `ctx.mcpReq.signal` to the backend call to stop the work when the client cancels the request.

### Streamable HTTP Server

```typescript
import { createMcpExpressApp } from '@modelcontextprotocol/express';
import { toNodeHandler } from '@modelcontextprotocol/node';
import { createMcpHandler } from '@modelcontextprotocol/server';

const app = createMcpExpressApp();
const node = toNodeHandler(createMcpHandler(createServer));
app.all('/mcp', (req, res) => void node(req, res, req.body));

const port = Number(process.env.PORT ?? 3000);
app.listen(port, '127.0.0.1');
```

`createMcpExpressApp` parses the JSON body. Pass `req.body` as the third argument of the handler. `createServer` is the factory from the stdio example.

### Integration Test

```typescript
import { Client } from '@modelcontextprotocol/client';
import { StdioClientTransport } from '@modelcontextprotocol/client/stdio';
import { afterAll, beforeAll, expect, test } from 'vitest';

let client: Client;

beforeAll(async () => {
    client = new Client({ name: 'test-client', version: '1.0.0' }, { versionNegotiation: { mode: { pin: '2026-07-28' } } });
    await client.connect(new StdioClientTransport({ command: process.execPath, args: ['dist/index.js'] }));
});

afterAll(async () => {
    await client.close();
});

test('add', async () => {
    const result = await client.callTool({ name: 'add', arguments: { a: 2, b: 3 } });
    expect(result.structuredContent).toEqual({ sum: 5 });
});
```
