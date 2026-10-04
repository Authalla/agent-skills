---
name: authalla
description: Add Authalla sign-in to an app. Configures the Authalla tenant (branding, custom domain, sender address, social login), creates the OAuth2 app, then analyzes the codebase and implements an OAuth 2.1 / OIDC integration. Use when adding Authalla login to an app.
metadata:
  author: authalla
  version: "2.0.0"
allowed-tools: Bash, AskUserQuestion, Read, Glob, Grep, Edit, Write
---

# Authalla

Authalla is a hosted OAuth2/OIDC provider. Each **tenant** is one issuer with its own users, sign-in page and apps, at `https://{tenant-id}.authalla.com` (or a custom domain). An **app** is an OAuth2 client of a tenant; its ID is the `client_id`.

You configure Authalla through the Authalla MCP tools, then write the integration in the user's code. To protect the user's own MCP server with Authalla, use the `authalla-mcp-server` skill instead.

Steps 3–6 are optional: ask before each, and skip what the user declines. The MCP tools' own schemas describe every parameter; this file covers order and the gotchas. For anything the tools don't cover, send the user to the dashboard at https://app.authalla.com.

## Steps

### 1. Connect the Authalla MCP server

Check for the Authalla MCP tools (`get_me`, `list_tenants`, `create_app`; with the Claude Code plugin their names end in these, such as `mcp__plugin_authalla_authalla__get_me`). If they're there, go on.

If they're missing and you aren't Claude Code, leave the `claude` command alone: send the user to https://docs.authalla.com/docs/mcp-server to add the server to their agent, then stop. On claude.ai or in Cowork, they connect it from the plugin's **Connectors** tab.

In Claude Code, run `claude mcp list` and look for an Authalla server (`https://api.authalla.com/mcp`). You can't sign in for the user; they do it from `/mcp`:

- **Needs authentication**, usually `plugin:authalla:authalla` from the Authalla plugin: tell the user to run `/mcp`, pick that server, choose **Authenticate**, then sign in to Authalla in the browser and approve the access. End your turn and ask them to tell you when they've signed in. The tools then appear in this session, without a restart.
- **Any other status** (failed, disabled, or connected but without tools in this session): tell the user to run `/mcp`, pick the server and fix it there with **Reconnect** or **Enable**. If it was added after this session started, they start a new session instead.
- **Not listed**: the user needs an Authalla account (sign up at https://authalla.com). Then add the server:

  ```bash
  claude mcp add --transport http authalla https://api.authalla.com/mcp
  ```

  This session can't see a server added after it started. Tell the user to start a new Claude Code session in this project, run `/mcp`, pick `authalla`, choose **Authenticate**, then sign in and approve. Then stop.

**Done when** `get_me` returns the user and their account.

### 2. Pick the tenant

`get_me` lists the account's tenants; every account starts with one. Ask which to use, or `create_tenant` for a new one (for example one per environment). A tenant's ID is permanent and is its sign-in address (`https://{id}.authalla.com`), so ask whether the user wants to choose it: an adjective and two nouns such as `brave-otter-lantern`, passed as `id`; omitted, it is random. A new tenant allows sign-ups unless `allow_registration` is false, and signs users in with email and passkeys. Then `select_tenant`, so later tools default to it.

Check `get_tenant` against what the user wants and fix it with `update_tenant`:

- **Registration**: whether users can sign up themselves, or must be created first (`create_user`). Users moving from another provider are created with `create_user`; to carry over their old user ID, provision them through SCIM instead, whose `externalId` reaches apps as the `external_id` claim when the app allows and requests the `external_id` scope: https://docs.authalla.com/docs/scim.
- **Product name**: the name users see in sign-in emails, such as `Acme`. Unset, they see the tenant's name.
- **Authentication methods**: `magic_link` (email sign-in: one email with a one-time code and a link) and `passkeys`. `auth_methods` replaces the whole list; an empty list leaves sign-in to connections only (social or enterprise SSO). Social login is not a method; it works while the tenant has an enabled connection (step 6).

**Done when** the tenant is selected and its registration, product name and methods are what the user asked for.

### 3. Branding

Ask the user for their colors as hex values; read them from the user, not from their website. `get_theme` shows the current theme, and `update_theme` changes only the fields passed: a brand color, page background and card background, plus corner style, font and default language. The default language applies to the sign-in emails as well as the pages; an app's `ui_locales` still wins. For separate dark-mode colors, set `color_mode` to `light_dark` with the `dark_*` fields.

For a logo or tab icon: `create_logo_upload_url`, upload the file with the `curl` command it gives, then pass the returned URL to `update_theme` as `logo_url` for a logo or `icon_url` for the tab icon. A public https image URL works too. SVG is not accepted.

Reference: https://docs.authalla.com/docs/branding

**Done when** `get_theme` shows the user's colors, logo and tab icon.

### 4. Custom domain

A custom domain such as `auth.example.com` serves the sign-in pages and OAuth2 endpoints, and becomes the **issuer**: the `iss` claim and discovery URL move to it, and sessions on it are separate from the default host. One per tenant; use a subdomain, since the apex of a domain usually can't take a CNAME.

1. `create_custom_domain` with the domain and tenant. It returns a CNAME record.
2. Show the record and ask which DNS provider they use. Any proxy/CDN toggle on the record must be off (Cloudflare: "DNS only", grey cloud), since Authalla issues the certificate itself.
3. When they've added it, `verify_custom_domain`, and read `status` with `get_custom_domain`: `active` is done; `pending` means DNS hasn't propagated yet (minutes, sometimes hours), so offer to check again; `error` means the record is wrong or proxied.

Reference: https://docs.authalla.com/docs/tenant-custom-domains

**Done when** the status is `active`, or the user chose to finish it later. In that case the integration uses the default `{tenant-id}.authalla.com` issuer, and switching later means changing the issuer in the app's config. Tell them switching also costs their users: passkeys are bound to the host they were created on, so ones made on `{tenant-id}.authalla.com` stop working on the custom domain, and sessions don't carry over, so everyone signs in again. A domain they will want is best added before real users sign up.

### 5. Sender address

Sign-in emails come from Authalla's address (`noreply@authalla-mail.com`) until the tenant sends from one of the account's **sender addresses**, such as `hi@example.com`. Sender addresses belong to the account, so several tenants can share one. The display name is the tenant's product name (step 2).

1. `add_sender_address` with the address. It returns DNS records for the domain: a `_authalla` TXT that proves ownership, and the records that authenticate its mail (SPF, DKIM, DMARC).
2. Show them as returned, with names relative to the domain (`_authalla` on `example.com` is `_authalla.example.com`).
3. When they've added them, `verify_sender_address`. It reports `verified` or `pending` with each record's `Found` state; re-check the missing ones.
4. Once `verified`, `set_tenant_sender_address` with the tenant and address.

Reference: https://docs.authalla.com/docs/custom-email

**Done when** the tenant sends from the address, or the user chose to finish verification later.

### 6. Social login

Providers: `google`, `github`, `microsoft`, `facebook`, `linkedin`, `x`, `discord`, one per tenant each. For each provider the user picks:

1. `get_social_login_redirect_uri`, and give the user that URL to register in the provider's developer console as the OAuth redirect/callback URL. They get a client ID and secret.
2. `create_social_login` with `provider_type` and `client_id`. Ask whether they want to share the client secret in chat. If yes, pass it. If not, omit it: the login is created disabled, and the response has a dashboard link where they paste the secret and enable it.

Reference: https://docs.authalla.com/docs/sso-connections

Enterprise SSO (a company's own OIDC or SAML identity provider, reached by email domain) is set up in the dashboard only; same reference.

**Done when** `list_social_logins` shows each chosen provider with status `active`, or the user chose to finish it in the dashboard.

### 7. Create the app

Ask what kind of application it is, and its URLs:

| Application | `application_type` | Client |
| --- | --- | --- |
| Server-rendered web app, or SPA with its own backend handling login | `web` | confidential (has a secret) |
| Browser-only SPA | `spa` | public (PKCE, no secret) |
| Mobile or desktop app | `native` | public |
| Machine-to-machine service, no users | `backend` | `client_credentials` only |

`create_app` with the name, tenant, type, `redirect_uris` and `allowed_logout_uris`:

- **Redirect URIs** match exactly, except that an `http` loopback one (`localhost`, `127.0.0.1`, `[::1]`) matches any port: `http://localhost/callback` covers `http://localhost:3000/callback`. Host and path must still match.
- **Logout URIs** match exactly, port included: register the local one as the app sends it, such as `http://localhost:3000/`.
- **Scopes** default to `openid profile email`. For refresh tokens the app requests `offline_access` at sign-in; it needs no app setting. Any other scope must be in the app's `scopes`: ones it isn't allowed are dropped without an error.
- **`require_consent`**: set it when users should approve the app's access on a consent screen first, such as for a third-party app.

For confidential apps the result contains the **client secret, shown once**. Tell the user to save it now in their secret store. Write it only into a gitignored local env file, and only if they ask. If it's lost, `add_app_secret` issues another.

A `spa` also needs its origin (such as `https://app.example.com`) in the tenant's **Allowed origins**, which only the dashboard sets: https://app.authalla.com → Tenants → (tenant) → General → Allowed origins.

**Done when** `get_app` shows the redirect and logout URIs and scopes the app will use, and the user has saved the secret (confidential apps).

### 8. Implement the integration

Read [integration.md](integration.md) and follow it: it analyzes the codebase, writes the code under Authalla's OAuth 2.1 security policy, and audits it.

**Done when** integration.md's security audit passes.

### 9. Summarize

Tell the user:

- Tenant name and ID, issuer URL, and discovery URL (`{issuer}/.well-known/openid-configuration`).
- What was configured: theme, custom domain and status, sender address and status, social logins.
- The app's name, `client_id` and type, its redirect and logout URIs.
- Files created or changed, and the environment variables to set.
- Anything left pending (DNS records, a social login secret to paste, allowed origins), with where to finish it.
- How to test: run the app, sign in, sign out, then sign in again in a private window.
