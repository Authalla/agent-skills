# TypeScript: `@modelcontextprotocol/sdk` with Express

Checked against `@modelcontextprotocol/sdk` 1.30 and `jose` 6. Install `jose` if it's missing.

Replace `ISSUER` (the tenant's issuer), `RESOURCE` (the identifier), the metadata document (paste what `get_resource_server` returned) and the scopes. Keep the user's existing server setup and tools; only the parts marked below are new.

```ts
import express from 'express';
import { createRemoteJWKSet, jwtVerify } from 'jose';
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StreamableHTTPServerTransport } from '@modelcontextprotocol/sdk/server/streamableHttp.js';
import { requireBearerAuth } from '@modelcontextprotocol/sdk/server/auth/middleware/bearerAuth.js';
import { getOAuthProtectedResourceMetadataUrl } from '@modelcontextprotocol/sdk/server/auth/router.js';
import { InvalidTokenError } from '@modelcontextprotocol/sdk/server/auth/errors.js';
import type { OAuthTokenVerifier } from '@modelcontextprotocol/sdk/server/auth/provider.js';

// New: who issues tokens, and who they are for.
const ISSUER = "https://acme.authalla.com";
const RESOURCE = new URL("https://mcp.acme.com/mcp");
const jwks = createRemoteJWKSet(new URL(`${ISSUER}/.well-known/jwks.json`));

// New: verify Authalla's access tokens locally.
const verifier: OAuthTokenVerifier = {
  async verifyAccessToken(token) {
    try {
      const { payload } = await jwtVerify(token, jwks, {
        issuer: ISSUER, audience: RESOURCE.href, typ: 'at+jwt', algorithms: ['RS256'],
      });
      return {
        token,
        clientId: String(payload.client_id),
        scopes: String(payload.scope ?? '').split(' ').filter(Boolean),
        expiresAt: payload.exp,
        extra: { sub: payload.sub },
      };
    } catch {
      throw new InvalidTokenError('Invalid access token'); // any other error becomes a 500
    }
  },
};

const app = express();
app.use(express.json());

// New: tell agents where to sign in (RFC 9728).
app.get("/.well-known/oauth-protected-resource/mcp", (_req, res) => {
  res.json({"resource":"https://mcp.acme.com/mcp","resource_name":"Acme CRM","authorization_servers":["https://acme.authalla.com"],"scopes_supported":["openid","crm:read","crm:write"],"bearer_methods_supported":["header"]});
});

// New: every MCP request needs a valid token with the read scope.
const auth = requireBearerAuth({
  verifier,
  requiredScopes: ["crm:read"],
  resourceMetadataUrl: getOAuthProtectedResourceMetadataUrl(RESOURCE),
});

app.post(RESOURCE.pathname, auth, async (req, res) => {
  const server = new McpServer({ name: "Acme CRM", version: '1.0.0' }); // existing tools register here
  const transport = new StreamableHTTPServerTransport({ sessionIdGenerator: undefined });
  res.on('close', () => { transport.close(); server.close(); });
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

app.listen(3000);
```

The metadata path is `/.well-known/oauth-protected-resource` followed by the identifier's path. For an identifier ending in `/mcp` it's the route above; for other paths, use the URL `get_resource_server` returned.

## Checking a write scope in a tool

`requireBearerAuth` puts the verified token on the request, and the transport passes it to tools as `authInfo`:

```ts
server.registerTool('update_deal_stage', {
  description: 'Move a deal to another pipeline stage',
  inputSchema: { id: z.string(), stage: z.string() },
}, async ({ id, stage }, { authInfo }) => {
  if (!authInfo?.scopes.includes('crm:write')) {
    return { content: [{ type: 'text', text: 'This needs the crm:write permission.' }], isError: true };
  }
  // ... the tool's existing body
});
```

`authInfo.extra.sub` is the signed-in user's ID. Use it to scope data to that user.
