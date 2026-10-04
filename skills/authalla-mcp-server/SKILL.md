---
name: authalla-mcp-server
description: Protect your own MCP server with Authalla, so users sign in and approve Claude, ChatGPT and other AI agents before they can call its tools. Use when adding sign-in or OAuth to an MCP server, or registering an MCP server as an Authalla resource server.
metadata:
  author: authalla
  version: "1.0.0"
allowed-tools: Bash, AskUserQuestion, Read, Glob, Grep, Edit, Write
---

# Protect an MCP server with Authalla

The user hosts the MCP server. Authalla is its **authorization server**: it signs users in, shows them a consent screen, and issues access tokens for the server. The server only **verifies** tokens, locally against the tenant's public keys, and runs no OAuth flows of its own.

The full guide, which is the source of truth when this file and the guide disagree: https://docs.authalla.com/docs/build-an-mcp-server

## Steps

### 1. Connect the Authalla MCP server

Check for the Authalla MCP tools (`get_me`, `list_tenants`, `create_resource_server`). If they're missing, ask the user to run this and restart their agent:

```bash
claude mcp add --transport http authalla https://api.authalla.com/mcp
claude mcp login authalla
```

**Done when** `get_me` returns the user's account.

### 2. Read the MCP server

Find, in the user's code:

- The language and MCP SDK (`@modelcontextprotocol/sdk` for TypeScript, `mcp` for Python), and where the HTTP transport handles requests.
- The **identifier**: the server's public URL including the MCP path, such as `https://mcp.acme.com/mcp`. Ask the user if it isn't deployed yet. `http` only works on `localhost`/`127.0.0.1`.
- Every tool, marked as **read** (only looks) or **write** (changes data or acts on the user's behalf).

**Done when** you have the identifier and every tool is marked read or write.

### 3. Agree the scopes with the user

Propose a **scope prefix** (short, lowercase, such as `crm`; it can't be changed later) and one scope per kind of access, named `read`, `write` and so on. Each gets a consent-screen description written for the end user ("Read your contacts and deals"). Then ask who gets each scope:

- **Grant to all users**: every current and future user of the tenant holds it. The usual choice for `read`.
- **Per user**: only users an administrator grants it to. Tokens never carry a scope the user doesn't hold. It's dropped without an error.

**Done when** the user has confirmed the prefix, the scopes, their descriptions and who gets each.

### 4. Configure Authalla

1. Pick the tenant whose users will sign in (`list_tenants`; ask if there are several). Note its **issuer** URL: every `ISSUER` below is this value.
2. `create_resource_server` with the identifier, a name users will recognise (it appears as "Claude wants access to <name>") and the prefix. Then `add_resource_server_scope` for each scope, with `grant_to_all_users` as agreed.
3. AI agents connect with **Client ID Metadata Documents**, which are off on new tenants. If the result says so, tell the user that turning it on lets any agent start a sign-in on this tenant, and that users still sign in and approve it on the consent screen. With their OK, set `allow_client_id_metadata_documents` with `update_tenant`. If they decline, tell them Claude and ChatGPT can't connect until it's on, and stop after step 5.

**Done when** `get_resource_server` lists every agreed scope, and the tenant allows Client ID Metadata Documents (or the user declined, as above).

### 5. Add token verification to the server

Follow the file for the server's language. It adds three things: the protected resource metadata route, a token verifier and the bearer check on the MCP endpoint.

- TypeScript (`@modelcontextprotocol/sdk`): [typescript.md](typescript.md)
- Python (`mcp`): [python.md](python.md)
- Anything else: implement the [token checks](#token-checks) with the platform's JWT library, and serve the metadata as described in the guide.

The metadata route serves the document `get_resource_server` returned, with the same values. Put the read scope in the **required scopes** for every request, and check each write scope inside the tools that need it.

**Done when** every write tool checks its scope, and the metadata route returns the same JSON as `get_resource_server`.

### 6. Prove it

Start the server, then:

1. `curl -i -X POST <identifier>` without a token returns `401` with `WWW-Authenticate: Bearer ... resource_metadata="<metadata URL>"`.
2. `curl <metadata URL>` returns the metadata document.
3. Connect an agent: `claude mcp add --transport http <name> <identifier>`, then `claude mcp login <name>`. The user signs in to the tenant in the browser and approves the consent screen.
4. Ask the agent to use a read tool, then a write tool.

**Done when** all four pass. If sign-in fails, see [Troubleshooting](#troubleshooting).

### 7. Hand over

Tell the user:
- Access is revoked per user under **Users → (user) → Connected apps**. The token already issued stays valid until it expires (15 minutes by default), because the server checks tokens locally.
- If the tenant gets a custom domain later, the issuer changes. Update the metadata and `ISSUER` together.

## Reference

### Token checks

Access tokens are RS256 JWTs (RFC 9068). Check all of these:

| Check | Expect |
| --- | --- |
| Signature | RS256, key from `{ISSUER}/.well-known/jwks.json` |
| `typ` header | `at+jwt`, so an ID token is refused |
| `iss` | `ISSUER`, exactly |
| `aud` | The identifier, exactly |
| `exp` | In the future |
| `scope` | Space-separated; contains what the request or tool needs |

`sub` is the user's ID and `client_id` is the agent, such as Claude Code's metadata URL.

### Troubleshooting

- **`invalid_target`** at sign-in: the `resource` the agent sent isn't a registered identifier. The identifier must match the URL the agent was given character for character, including the path and any trailing slash.
- **The agent never opens a sign-in**: the server doesn't answer `401` with `resource_metadata`, or the metadata URL isn't served at the path RFC 9728 expects (`/.well-known/oauth-protected-resource` followed by the identifier's path).
- **The consent screen lists no scopes**, or calls return `403 insufficient_scope`: the user doesn't hold the scope. Grant it on the user or turn on Grant to all users. A scope added to a user reaches the agent at its next sign-in, not at a token refresh: run `claude mcp login <name>` again.
- **The agent can't connect at all**: it only supports Dynamic Client Registration, which Authalla doesn't offer. Claude, Claude Code and ChatGPT use Client ID Metadata Documents. Check that they're allowed on the tenant (step 4).
- **`500` instead of `401` on a bad token** (TypeScript): the verifier must throw `InvalidTokenError`; any other error becomes a 500.
