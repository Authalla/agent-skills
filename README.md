# Authalla Agent Skills

Official [Claude Code skills](https://skills.sh) for [Authalla](https://authalla.com) — the OAuth2/OIDC authentication platform.

## Install

```bash
npx skills add authalla/agent-skills
```

## Skills

### authalla

Set up Authalla authentication for your app through a guided interactive flow. Configures your tenant (branding, custom domain, email, social login), creates your app, then writes and audits a secure OAuth 2.1 / OIDC integration for your stack.

**Prerequisites:**
- An Authalla account ([sign up](https://authalla.com))
- The Authalla MCP server: `claude mcp add --transport http authalla https://api.authalla.com/mcp`, then `claude mcp login authalla`

**Usage:** Once installed, Claude Code will automatically use this skill when you ask it to set up Authalla authentication.

### authalla-mcp-server

Protect your own MCP server with Authalla: users sign in and approve Claude, ChatGPT and other AI agents before they can call its tools. Registers the server as a resource server with its scopes, adds token verification to your code (TypeScript or Python MCP SDK), and walks you through connecting an agent.

**Prerequisites:**
- An Authalla account ([sign up](https://authalla.com))
- The Authalla MCP server: `claude mcp add --transport http authalla https://api.authalla.com/mcp`, then `claude mcp login authalla`

**Usage:** ask your agent to "protect my MCP server with Authalla". Guide: [Build an MCP server with Authalla](https://docs.authalla.com/docs/build-an-mcp-server).
