# Streamable HTTP Servers

Use this transport only for a remote server. A local server uses stdio.

## Express server

`createMcpHandler` converts the server factory into a web-standard HTTP handler. `toNodeHandler` adapts it to the `(req, res)` functions of Node.js. `createMcpExpressApp` is `express()` with `express.json()` and DNS rebinding protection. Add the packages:

```bash
npm install @modelcontextprotocol/express @modelcontextprotocol/node express
npm install -D @types/express
```

Move `createServer()` and its imports to `src/server.ts`, and export it. In `src/index.ts`, keep the shebang line. Replace the rest of the file with:

```typescript
import { createMcpExpressApp } from '@modelcontextprotocol/express';
import { toNodeHandler } from '@modelcontextprotocol/node';
import { createMcpHandler } from '@modelcontextprotocol/server';

import { createServer } from './server.js';

const app = createMcpExpressApp();
const node = toNodeHandler(createMcpHandler(createServer));
app.all('/mcp', (req, res) => void node(req, res, req.body));

const port = Number(process.env.PORT ?? 3000);
app.listen(port, '127.0.0.1', () => console.error(`demo MCP server listening on http://127.0.0.1:${port}/mcp`));
```

`createMcpExpressApp` already parses the JSON body. Pass `req.body` as the third argument of the handler, so that it does not read the stream again.

Read a request header in a handler with `ctx.http?.req?.headers.get('x-custom')`. The `ctx` object is the last parameter of the handler.

## Other frameworks and runtimes

For a runtime with web-standard `Request` and `Response` objects, call `handler.fetch(request)` directly:

```typescript
import { createMcpHandler, hostHeaderValidationResponse, originValidationResponse } from '@modelcontextprotocol/server';

const handler = createMcpHandler(createServer);

export default {
    async fetch(request: Request): Promise<Response> {
        const rejected =
            hostHeaderValidationResponse(request, ['api.example.com']) ?? originValidationResponse(request, ['app.example.com']);
        return rejected ?? handler.fetch(request);
    }
};
```

The SDK also has adapters for Hono and Fastify. Read the [SDK documentation](https://ts.sdk.modelcontextprotocol.io/v2/) for them.

## Tests

The stdio test of Step 3 does not work for an HTTP server. The test imports `createServer` from `dist/server.js`, so it does not start the server. Add `"declaration": true` to `tsconfig.json`. With this option, the test gets the types of `createServer`. In `test/server.test.ts`, replace all lines above the first `test(` call with:

```typescript
import { Client, StreamableHTTPClientTransport } from '@modelcontextprotocol/client';
import { createMcpHandler, type McpHttpHandler } from '@modelcontextprotocol/server';
import { afterAll, beforeAll, expect, test } from 'vitest';

import { createServer } from '../dist/server.js';

let client: Client;
let handler: McpHttpHandler;

beforeAll(async () => {
    handler = createMcpHandler(createServer);
    client = new Client({ name: 'test-client', version: '1.0.0' }, { versionNegotiation: { mode: { pin: '2026-07-28' } } });
    await client.connect(
        new StreamableHTTPClientTransport(new URL('http://test.local/mcp'), {
            fetch: (url, init) => handler.fetch(new Request(url, init))
        })
    );
});

afterAll(async () => {
    await client.close();
    await handler.close();
});
```

Close the client first, then the handler. The client sends each request to `handler.fetch` in the test process. The test opens no port. Keep the other tests.

## Client configuration

For VS Code, write `.vscode/mcp.json`:

```json
{
  "servers": {
    "demo": {
      "type": "http",
      "url": "http://127.0.0.1:3000/mcp"
    }
  }
}
```

## Gotchas

- `createMcpHandler` does not check the `Host` and `Origin` headers. Without a check, a malicious web page can use DNS rebinding to call a local server. `createMcpExpressApp` checks `Host` and `Origin` for a `127.0.0.1` bind. Pass the same host to `app.listen`. Do not use the deprecated `enableDnsRebindingProtection` option. When you bind to all interfaces, the default protection ends. Name the hosts that you serve: `createMcpExpressApp({ host: '0.0.0.0', allowedHosts: ['api.example.com'], allowedOrigins: ['app.example.com'] })`, and call `app.listen(port, '0.0.0.0', ...)`. Also name the browser origins in `allowedOrigins`. Without it, the app does not check `Origin` on this bind. On a web-standard runtime, put `hostHeaderValidationResponse` and `originValidationResponse` from `@modelcontextprotocol/server` in front of the handler. On plain `node:http`, use `localhostHostValidation` and `localhostOriginValidation` from `@modelcontextprotocol/node`. `createMcpHonoApp` and `createMcpFastifyApp` enable the protection by default.
- `createMcpHandler(factory)` also serves 2025 clients by default, without sessions. To serve only the revision 2026-07-28, use `createMcpHandler(factory, { legacy: 'reject' })`. To keep a 2025 server that has sessions, call `await isLegacyRequest(request)` in front of a strict handler. It returns a promise. Send the requests for which it resolves to `false` to the strict handler. In an Express app, the body parser reads the stream first. Make the `Request` with `await toWebRequest(req, req.body)` from `@modelcontextprotocol/node`.
- The revision 2026-07-28 removes `Mcp-Session-Id`. Each Streamable HTTP POST request needs the `MCP-Protocol-Version` and `Mcp-Method` headers. A `tools/call`, `resources/read`, or `prompts/get` request also needs `Mcp-Name`. `createMcpHandler` answers HTTP 400 with `-32020` when a header is missing or does not match the body. If a browser client calls the server, add `MCP-Protocol-Version`, `Mcp-Method`, and `Mcp-Name` to `Access-Control-Allow-Headers` in your framework.
- The POST `Content-Type` must be `application/json`. Otherwise the server answers with HTTP 415.
- The server is stateless. The factory runs for each request. Keep the factory cheap. Create database pools, API clients, and caches once at module scope. Use them in the factory. There are no sessions. Per-call state lives in handles that the server makes and the model passes as ordinary tool arguments. Use `requestState` only for the state between the retries of an `inputRequired` result.
- Mark a tool parameter with `x-mcp-header` to copy it into the `Mcp-Param-{Name}` header. Use it only for `string`, `integer`, and `boolean` parameters. Do not mark a secret or personal data with `x-mcp-header`. Network intermediaries can read headers. A browser SDK client does not send `Mcp-Param-*` headers. Thus, `createMcpHandler` rejects a call that has a value for such a parameter. Do not use `x-mcp-header` on a tool that a browser client calls. For an integer, use `z.int()`, not `z.number()`. If a header does not match the body, `createMcpHandler` answers with HTTP 400 and the code `-32020`.
- `NodeStreamableHTTPServerTransport` from `@modelcontextprotocol/node` is only for 2025-era clients.
- The HTTP+SSE transport of 2024-11-05 is deprecated (SEP-2596). Use Streamable HTTP. Streamable HTTP still sends SSE response streams. The revision 2026-07-28 removes SSE resumability and `Last-Event-ID`. A client repeats a broken request with a new ID.
- `createMcpHandler` does not verify tokens. It passes `authInfo` to the handler as it gets it. Verify the token in a middleware in front of the handler, for example with `requireBearerAuth` from `@modelcontextprotocol/express`.
- Use the `PORT` environment variable for the port.
