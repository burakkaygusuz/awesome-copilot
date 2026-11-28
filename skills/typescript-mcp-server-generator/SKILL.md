---
name: typescript-mcp-server-generator
description: 'Use this skill when the user wants to create a new Model Context Protocol (MCP) server in TypeScript or Node.js. Also use it to expose TypeScript code, an API, or a database to AI clients as MCP tools. The skill makes an npm project with the MCP TypeScript SDK 2.x, Vitest tests, and client configuration. Do not use it for an MCP client or for a server in another language.'
compatibility: 'Node.js 22.12 or later on an LTS line (22, 24, or 26), npm, and network access to download packages. The minimum Node.js versions for the MCP Inspector are 22.23.4, 24.22.1, and 26.12.1, one for each LTS line.'
---

# TypeScript MCP Server Generator

Create a Model Context Protocol (MCP) server in TypeScript with the official [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) 2.x. SDK 2.4.x implements the MCP specification revision [2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28). A client enables it with `versionNegotiation`. By default, a client still uses the 2025 `initialize` handshake.

## Workflow

Copy this checklist. Use it to track your progress:

- [ ] Step 1: Create the project.
- [ ] Step 2: Write the server.
- [ ] Step 3: Write the tests.
- [ ] Step 4: Build until the tests pass.
- [ ] Step 5: Configure the client.

### Step 1: Create the project

Make this layout:

```text
project-name/
├── package.json
├── tsconfig.json
├── src/index.ts
└── test/server.test.ts
```

Write `package.json`:

```json
{
  "name": "my-mcp-server",
  "version": "0.1.0",
  "type": "module",
  "bin": { "my-mcp-server": "dist/index.js" },
  "scripts": {
    "build": "tsc",
    "dev": "tsx src/index.ts",
    "start": "node dist/index.js",
    "test": "vitest run"
  }
}
```

Write `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "strict": true,
    "types": ["node"],
    "rootDir": "src",
    "outDir": "dist"
  },
  "include": ["src"]
}
```

Install the dependencies:

```bash
npm install @modelcontextprotocol/server zod@^4.2
npm install -D typescript@^7.0.2 @types/node@^22 tsx vitest@^5.1.2 @modelcontextprotocol/client
```

- `@modelcontextprotocol/server` contains `McpServer` and the stdio entry `@modelcontextprotocol/server/stdio`.
- `@modelcontextprotocol/client` is only for the tests.
- `tsx` runs the server during development. Do not use Node.js native TypeScript execution.

### Step 2: Write the server

Start from this template in `src/index.ts`:

```typescript
#!/usr/bin/env node
import { McpServer } from '@modelcontextprotocol/server';
import { serveStdio } from '@modelcontextprotocol/server/stdio';
import * as z from 'zod/v4';

function createServer(): McpServer {
    const server = new McpServer({ name: 'demo', version: '0.1.0' });

    server.registerTool(
        'divide',
        {
            title: 'Divide',
            description: 'Divide a by b.',
            inputSchema: z.object({
                a: z.number().describe('Dividend'),
                b: z.number().describe('Divisor')
            }),
            outputSchema: z.object({ result: z.number() })
        },
        async ({ a, b }) => {
            if (b === 0) {
                return { content: [{ type: 'text', text: 'b must not be zero.' }], isError: true };
            }
            const result = a / b;
            return {
                content: [{ type: 'text', text: JSON.stringify({ result }) }],
                structuredContent: { result }
            };
        }
    );

    return server;
}

void serveStdio(createServer);
console.error('demo MCP server running on stdio');
```

Then change the template to match the request of the user:

1. Replace the `divide` tool with the tools that the user asks for.
2. Give each tool a zod `inputSchema` (`z.object({ ... })`), a `title`, and a `description`. The description tells the model when to use the tool.
3. When the client must get machine-readable data, add an `outputSchema`. Return the data in `structuredContent`. Also return the same data as serialized JSON in a text block for older clients.
4. Mark a read-only tool with `annotations: { readOnlyHint: true }`.
5. Add resources (`registerResource`) and prompts (`registerPrompt`) only when the user asks for them.

Use stdio by default. Read [references/streamable-http.md](references/streamable-http.md) only when the user asks for a remote server or HTTP.

### Step 3: Write the tests

The test starts the built server as a subprocess. It connects to the server with the SDK client. Write a test for each tool and for each error result in `test/server.test.ts`:

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

test('negotiates the 2026-07-28 revision', () => {
    expect(client.getNegotiatedProtocolVersion()).toBe('2026-07-28');
});

test('divide', async () => {
    const result = await client.callTool({ name: 'divide', arguments: { a: 6, b: 3 } });
    expect(result.structuredContent).toEqual({ result: 2 });
});

test('divide by zero', async () => {
    const result = await client.callTool({ name: 'divide', arguments: { a: 1, b: 0 } });
    expect(result.isError).toBe(true);
    expect(JSON.stringify(result.content)).toContain('must not be zero');
});
```

- For an HTTP server, use the test setup in [references/streamable-http.md](references/streamable-http.md).
- The test runs `dist/index.js`. Run `npm run build` before `npm test`.
- `process.execPath` is the `node` command that runs the tests. Do not use the `node` command from `PATH`.
- The test uses the real stdio transport and the real JSON-RPC messages.
- The `pin` option makes the client use the revision 2026-07-28. Without it, the client uses the 2025 handshake.

### Step 4: Build until the tests pass

1. Run `npm run build && npm test`. This command compiles the code to `dist/` and runs the tests.
2. If a test or the compiler fails, read the error. Fix the server or the test.
3. Run `npm run build && npm test` again.
4. Continue only when the build and all tests pass.

### Step 5: Configure the client

For VS Code, write `.vscode/mcp.json`:

```json
{
  "servers": {
    "demo": {
      "type": "stdio",
      "command": "node",
      "args": ["/absolute/path/to/project/dist/index.js"]
    }
  }
}
```

For Claude Desktop, put the same `command` and `args` in `claude_desktop_config.json`, below the `mcpServers` key.

For an HTTP server, use the client configuration in [references/streamable-http.md](references/streamable-http.md).

Tell the user these facts:

- `node dist/index.js` starts the stdio server. The server waits for a host on stdin. The logs go to stderr.
- To test the server manually, run `npx @modelcontextprotocol/inspector node dist/index.js`. The minimum Node.js versions for the MCP Inspector are 22.23.4, 24.22.1, and 26.12.1, one for each LTS line.

## Gotchas

SDK 2.x changed many patterns of the v1 package `@modelcontextprotocol/sdk`. Do not copy examples from v1. To update an existing server, run `npx @modelcontextprotocol/codemod@latest v1-to-v2 .`. Then search for `@mcp-codemod-error` markers. Read the [upgrade guide](https://ts.sdk.modelcontextprotocol.io/v2/migration/upgrade-to-v2.html) for the full list.

- Use the focused v2 packages. Do not install `@modelcontextprotocol/sdk`. The SDK removed the SSE server transport and the WebSocket client transport. The frozen copy of the old SSE code is only in `@modelcontextprotocol/server-legacy`.
- The SDK does not support Zod 3. Use `zod@^4.2` and give schemas as `z.object({ ... })`. The raw shape form `{ name: z.string() }` is deprecated.
- Use `registerTool()`, `registerResource()`, and `registerPrompt()`. The variadic `tool()` methods do not exist.
- If a tool has an `outputSchema`, return `structuredContent` in each successful result.
- Write `completable(z.string(), callback).optional()`. Put `.optional()` on the outside.
- Return a result with `isError: true` for an error that the model must read. The v2 error classes are `ProtocolError`, `SdkError`, and `SdkHttpError` (with `.status`). `McpError` and `StreamableHTTPError` do not exist.
- In a stdio server, stdout is the protocol channel. Do not call `console.log()`. Use `console.error()`.
- There are no sessions and no `initialize` handshake on 2026-07-28. Per-call state lives in handles that the server makes and the model passes as tool arguments.
- The SDK answers `server/discover`, fills `resultType`, and routes `subscriptions/listen`. Do not write them manually.
- Do not use `ctx.mcpReq.elicitInput()`. It throws on a 2026-07-28 request. Return `inputRequired({ inputRequests: { ... } })`. Read the answer with `acceptedContent(ctx.mcpReq.inputResponses, 'id', schema)`. Use the same zod schema that you gave to `inputRequired.elicit`. The content comes from the client and is untrusted. To keep state between the retries, return `requestState` in `inputRequired(...)`. `requestState` comes back from the client. Treat it as untrusted input. Create one codec at module scope, outside the factory, with `createRequestStateCodec({ key })` from `@modelcontextprotocol/server`. Load a key of at least 32 bytes from the configuration. Each server process must use the same key. Pass a `bind` function that returns the user and `ctx.mcpReq.method`. Every tool call has the method `tools/call`. Thus, also put the tool name and the important arguments in the state. On the retry, reject the state if they do not match the call. Return `requestState: await codec.mint(state, ctx)` in `inputRequired(...)`. Set `requestState: { verify: codec.verify }` in the `McpServer` options. Read the state with `ctx.mcpReq.requestState<T>()`. The codec signs the state but does not encrypt it. Do not put secrets in the state.
- The revision 2026-07-28 deprecates sampling, MCP logging, and roots ([SEP-2577](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2577)). They continue to work for at least twelve months. HTTP+SSE is deprecated since 2025-03-26 (SEP-2596). It can be removed three months after SEP-2596 becomes final. Do not add these features to a new server:
  - **Sampling** (`server.server.createMessage()`): Call the LLM provider API directly. If existing code must keep sampling, use `inputRequired.createMessage`.
  - **MCP logging** (`ctx.mcpReq.log`): Log to stderr or use OpenTelemetry.
  - **Roots**: Get paths from tool arguments, resource URIs, or the server configuration.
  - **HTTP+SSE**: Use Streamable HTTP for a remote server.
