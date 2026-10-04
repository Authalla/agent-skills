# Authalla Agent Skills

Official agent skills for [Authalla](https://authalla.com), the OAuth2/OIDC authentication platform.

## Install

In Claude Code:

```
/plugin marketplace add authalla/agent-skills
/plugin install authalla@authalla
```

The plugin also connects Claude Code to the Authalla MCP server. Run `/mcp`, pick `plugin:authalla:authalla` and choose **Authenticate** to sign in to your Authalla account. If you already added the server yourself with `claude mcp add`, Claude Code keeps using that one.

To get updates automatically, turn on auto-update for the `authalla` marketplace under `/plugin` → Marketplaces. Otherwise run `/plugin marketplace update authalla`.

For other agents (Codex, Cursor, GitHub Copilot and more):

```bash
npx skills add authalla/agent-skills
```

Use one of the two. If you installed with `npx skills` before and move to the plugin, remove those skills first (`npx skills remove authalla authalla-mcp-server`), or Claude Code loads each skill twice.

## Skills

### authalla

Set up Authalla authentication for your app through a guided interactive flow. Configures your tenant (branding, custom domain, email, social login), creates your app, then writes and audits a secure OAuth 2.1 / OIDC integration for your stack.

**Prerequisites:**
- An Authalla account ([sign up](https://authalla.com))
- The Authalla MCP server, signed in. The Claude Code plugin includes it: run `/mcp`, pick the server and choose **Authenticate**. Without the plugin, run `claude mcp add --transport http authalla https://api.authalla.com/mcp` first. Other agents: see [MCP server](https://docs.authalla.com/docs/mcp-server).

**Usage:** once installed, your agent uses this skill when you ask it to set up Authalla authentication.

### authalla-mcp-server

Protect your own MCP server with Authalla: users sign in and approve Claude, ChatGPT and other AI agents before they can call its tools. Registers the server as a resource server with its scopes, adds token verification to your code (TypeScript or Python MCP SDK), and walks you through connecting an agent.

**Prerequisites:**
- An Authalla account ([sign up](https://authalla.com))
- The Authalla MCP server, signed in. The Claude Code plugin includes it: run `/mcp`, pick the server and choose **Authenticate**. Without the plugin, run `claude mcp add --transport http authalla https://api.authalla.com/mcp` first. Other agents: see [MCP server](https://docs.authalla.com/docs/mcp-server).

**Usage:** ask your agent to "protect my MCP server with Authalla". Guide: [Build an MCP server with Authalla](https://docs.authalla.com/docs/build-an-mcp-server).
