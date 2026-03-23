---
name: authalla
description: Set up Authalla authentication for your app. Connects your project to Authalla OAuth2/OIDC — configures branding, custom domain, email, social login, and creates your OAuth2 client, all through a guided interactive flow.
metadata:
  author: authalla
  version: "1.0.0"
allowed-tools: Bash, WebFetch, AskUserQuestion, Read, Glob, Grep
---

# Authalla

Set up Authalla authentication for your application through an interactive, guided flow.

## When to Use

Use this skill when the user wants to:
- Set up Authalla for their application
- Configure their Authalla tenant
- Integrate Authalla authentication into their project
- Set up custom domain, email branding, or login methods

## Prerequisites

The user must have:
1. An Authalla account (sign up at https://authalla.com)
2. A Backend Service client with `client_id` and `client_secret` (created automatically on signup, visible in the Authalla Admin UI under API settings)
3. The Authalla CLI installed:
   ```bash
   brew tap authalla/tap
   brew install authalla
   ```

## Flow

Execute these steps in order. Be conversational and guide the user through each step.

### Step 1: Configure CLI

First, check if the CLI is installed and configured:

```bash
authalla config show
```

**If this fails** (CLI not installed), tell the user to install it:
```bash
brew tap authalla/tap
brew install authalla
```

**If config is not set**, guide the user to configure credentials. Use AskUserQuestion to ask for their credentials, then run:

```bash
authalla config set \
  --api-url https://TENANT_ID.authalla.com \
  --client-id CLIENT_ID \
  --client-secret CLIENT_SECRET
```

**Important about the API URL:** The CLI must be configured with the **tenant-specific** URL (`https://TENANT_ID.authalla.com`), where the tenant ID is the subdomain. The user can find their tenant ID by opening the tenant in the Authalla Admin UI — the ID is shown in the URL and at the top of the page.

**Important:** Never ask the user to paste credentials directly into the chat. Ask them to run the config command themselves if they prefer, or use AskUserQuestion to collect the values.

Once configured, verify it works by listing tenants:

```bash
authalla tenant list
```

If you get a `403 Forbidden` or a response containing "does not match tenant domains", the API URL is wrong. Check the error message — it usually includes the correct tenant domain (e.g., `public: xyz.authalla.com`). Reconfigure with the correct URL:

```bash
authalla config set \
  --api-url https://CORRECT_TENANT_ID.authalla.com \
  --client-id CLIENT_ID \
  --client-secret CLIENT_SECRET
```

### Step 2: Discover Tenant

List the user's tenants:

```bash
authalla tenant list
```

Store the first tenant's `id` and `name` for subsequent commands.

### Step 3: Brand Setup

Ask the user about their brand. Use AskUserQuestion with these options:
- "I have a website URL you can check" → Fetch the website using WebFetch, extract brand colors from CSS/meta tags, and present them for confirmation
- "I'll provide brand colors manually" → Ask for primary color (hex)
- "Use default theme" → Skip this step

If analyzing a website, look for:
- CSS custom properties (--primary-color, --brand-color, etc.)
- Meta theme-color tag
- Common CSS patterns for brand colors
- Logo images

To see all available theme fields:

```bash
authalla theme schema update
```

After determining colors, update the theme:

```bash
authalla theme update --json '{"primary_color": "#HEX_COLOR", "background_color": "#ffffff", "text_color": "#111827"}'
```

### Step 4: Custom Domain (Optional)

Ask the user if they want to set up a custom domain for their login pages (e.g., `auth.example.com`).

To see the required fields:

```bash
authalla custom-domain schema create
```

If yes, create the custom domain:

```bash
authalla custom-domain create --json '{"tenant_id": "TENANT_ID", "domain": "auth.example.com"}'
```

Then fetch the DNS records:

```bash
authalla custom-domain get --id DOMAIN_ID
```

Present the DNS records clearly and provide **registrar-specific instructions**. Ask the user which DNS provider/registrar they use (Cloudflare, Namecheap, GoDaddy, Gandi, Route 53, Google Domains, Vercel, etc.). Then use WebFetch to look up the registrar's documentation for adding DNS records, and present step-by-step instructions tailored to their provider.

For example:

```
To activate your custom domain, add these DNS records:

| Type  | Name                | Value                          |
|-------|---------------------|--------------------------------|
| CNAME | auth.example.com    | tenant_xxx.authalla.com        |
| TXT   | _cf-custom-hostname | verification-token-here        |
```

Then provide registrar-specific guidance. Common patterns:

- **Cloudflare**: Dashboard → DNS → Add Record. For CNAME records, set Proxy Status to "DNS only" (gray cloud). For the Name field, use just the subdomain part (e.g., `auth` not `auth.example.com`).
- **Namecheap**: Dashboard → Domain List → Manage → Advanced DNS → Add New Record. For Host, use just the subdomain (e.g., `auth`). For CNAME Value, include trailing dot.
- **GoDaddy**: My Products → DNS → Add Record. Use subdomain only in Name field.
- **Gandi**: Domain → DNS Records → Add Record. Use full subdomain in Name field.
- **AWS Route 53**: Hosted Zone → Create Record. For CNAME, use the full domain name.
- **Vercel**: Project Settings → Domains. Vercel may require specific CNAME setup.
- **Google Domains / Google Cloud DNS**: DNS → Manage custom records → Create new record.

If you're unsure about a specific registrar, use WebFetch to look up their DNS documentation and provide accurate instructions.

After presenting the records, ask the user to confirm when they've added them:

```
Have you added the DNS records? I can verify the setup once you're ready.
```

When the user confirms, trigger verification:

```bash
authalla custom-domain verify --id DOMAIN_ID
```

Check the `status` field in the response:
- `active` → Domain is verified and working. Celebrate!
- `pending` → DNS changes haven't propagated yet. Tell the user DNS propagation can take up to 24-48 hours, but often completes within minutes. Offer to check again.
- `error` → Something is wrong. Check the `verification_records` for status details. Help the user troubleshoot (wrong record value, proxied CNAME on Cloudflare, etc.).

If pending, poll by checking status again:

```bash
authalla custom-domain get --id DOMAIN_ID
```

Continue checking until the status becomes `active` or the user wants to move on. Suggest the user can always check back later from the Authalla Admin UI.

### Step 5: Custom Email (Optional)

Ask the user if they want branded authentication emails (e.g., emails from `noreply@mail.example.com`).

To see the required fields:

```bash
authalla custom-email schema create
```

If yes, create the custom email domain:

```bash
authalla custom-email create --json '{"tenant_id": "TENANT_ID", "email_domain": "mail.example.com", "sender_email": "noreply@mail.example.com", "sender_name": "Company Name"}'
```

Present the DNS records for email verification. Same as with custom domains, ask which registrar the user uses and provide **registrar-specific instructions** for adding TXT and CNAME records.

```
To activate custom email delivery, add these DNS records:

| Type  | Name                        | Value                    |
|-------|-----------------------------|--------------------------|
| TXT   | _brevo_code.mail.example.com| brevo-verification-code  |
| CNAME | _dkim.mail.example.com      | dkim.brevo.com           |
| TXT   | _dmarc.mail.example.com     | v=DMARC1; p=none         |
```

Explain that:
- The `_brevo_code` TXT record proves domain ownership
- The `_dkim` CNAME record enables email authentication (DKIM signing)
- The `_dmarc` TXT record sets the DMARC policy for email deliverability

After the user confirms they've added the records, trigger verification:

```bash
authalla custom-email verify --id EMAIL_ID
```

Check the `verification_status` field:
- `verified` → All DNS records are verified and email delivery is active
- `pending` → DNS records haven't been verified yet. DNS propagation can take time.
- `error` → Check the `dns_records` array for per-record status to identify which record is failing

If pending, poll by checking status again:

```bash
authalla custom-email get --id EMAIL_ID
```

### Step 6: Social Login (Optional)

Ask the user if they want to enable social login (e.g., "Sign in with Google"). Use AskUserQuestion:
- Which social login providers do you want to enable? (Google / GitHub / Apple / Microsoft / None)

For each selected provider, explain what credentials are needed and where to get them:

- **Google**: Create OAuth2 credentials at https://console.cloud.google.com/apis/credentials. You need the Client ID and Client Secret. Set the authorized redirect URI to `https://TENANT_ID.authalla.com/oauth2/social/google/callback` (or `https://auth.example.com/oauth2/social/google/callback` if custom domain is set up).
- **GitHub**: Create an OAuth App at https://github.com/settings/developers. Set the callback URL to `https://TENANT_ID.authalla.com/oauth2/social/github/callback`.
- **Apple**: Create a Services ID at https://developer.apple.com. Callback URL: `https://TENANT_ID.authalla.com/oauth2/social/apple/callback`.
- **Microsoft**: Register an app at https://portal.azure.com. Redirect URI: `https://TENANT_ID.authalla.com/oauth2/social/microsoft/callback`.

To see the full schema:

```bash
authalla social-login schema create
```

Ask the user for the Client ID and Client Secret for each selected provider. Then create and attach:

```bash
authalla social-login create --json '{"name": "Google", "provider_type": "google", "client_id": "GOOGLE_CLIENT_ID", "client_secret": "GOOGLE_CLIENT_SECRET", "tenant_ids": ["TENANT_ID"]}'
```

After creating social login providers, enable the `social_logins` auth method on the tenant. Include `magic_link` as well so users always have a fallback:

```bash
authalla tenant update --id TENANT_ID --json '{"name": "TENANT_NAME", "allow_registration": true, "auth_methods": ["magic_link", "social_logins"]}'
```

If the user skips social login, still ensure `magic_link` is enabled:

```bash
authalla tenant update --id TENANT_ID --json '{"name": "TENANT_NAME", "allow_registration": true, "auth_methods": ["magic_link"]}'
```

### Step 7: Create Application OAuth2 Client

Ask the user about their application to create the right type of OAuth2 client.

Use AskUserQuestion:
- What is your application type? (SPA / Web App / Mobile/Native / Backend Service)
- What is your application's URL? (for redirect URIs)

To see all available fields:

```bash
authalla client schema create
```

Then create the client:

```bash
authalla client create --json '{"name": "My App", "tenant_id": "TENANT_ID", "application_type": "spa", "redirect_uris": ["http://localhost:3000/callback", "https://app.example.com/callback"], "allowed_logout_uris": ["http://localhost:3000", "https://app.example.com"]}'
```

**Important:** The `client_secret` is only returned once on creation for confidential clients. Store it immediately and present it clearly to the user.

### Step 8: Analyze Codebase and Suggest Integration

After creating the client, analyze the user's codebase to suggest how to integrate Authalla:

1. Check for existing auth libraries (next-auth, passport, oidc-client, etc.)
2. Look at the tech stack (React, Next.js, Express, Go, etc.)
3. Suggest the appropriate integration approach

#### OIDC Discovery Check

Before writing integration code, **always fetch the tenant's OIDC discovery document** to check supported features:

```bash
curl -s https://TENANT_ID.authalla.com/.well-known/openid-configuration
```

Check these fields and configure the integration accordingly:
- `token_endpoint_auth_methods_supported` — determines how the client authenticates at the token endpoint (e.g. `client_secret_post`, `client_secret_basic`)
- `code_challenge_methods_supported` — if `S256` is listed, PKCE is supported and should be enabled
- `response_types_supported` — verify `code` is supported (authorization code flow)

#### Security Requirements

Authalla integrations **must** use both `state` and `pkce` checks:
- **state**: CSRF protection — a random value sent in the authorization request and validated on callback
- **pkce** (S256): Proof Key for Code Exchange — prevents authorization code interception attacks

For confidential clients (application_type `web` or `backend`), also configure the token endpoint authentication method to `client_secret_post` (sends `client_id` and `client_secret` in the request body rather than the Authorization header).

#### Framework-Specific Guidance

- **Next.js (Auth.js / next-auth v5)**: Use a custom OIDC provider. Install `next-auth@beta`. Configure with:
  ```typescript
  {
    id: "authalla",
    name: "Authalla",
    type: "oidc",
    issuer: process.env.AUTH_AUTHALLA_ISSUER,
    clientId: process.env.AUTH_AUTHALLA_ID,
    clientSecret: process.env.AUTH_AUTHALLA_SECRET,
    checks: ["state", "pkce"],
    client: {
      token_endpoint_auth_method: "client_secret_post",
    },
  }
  ```
  **Important:** The `client` property sets the `token_endpoint_auth_method` at the oauth4webapi level. Do NOT use `token.clientAuthentication` — it may not be applied correctly.

- **React SPA**: Use `oidc-client-ts` or `react-oidc-context`. Enable PKCE (default in oidc-client-ts). Set `response_type: "code"`.
- **Express/Node**: Use `passport-openidconnect`. Configure with PKCE enabled and `tokenEndpointAuthMethod: "client_secret_post"`.
- **Go**: Use `coreos/go-oidc` with `golang.org/x/oauth2`. Enable PKCE with S256 challenge method.

#### OIDC Configuration Values

Provide these to the user:
```
OIDC Configuration:
- Issuer URL: https://TENANT_ID.authalla.com (or https://auth.example.com if custom domain)
- Discovery URL: https://TENANT_ID.authalla.com/.well-known/openid-configuration
- Client ID: client_xxx
- Client Secret: secret_xxx (for confidential clients)
- Redirect URI: http://localhost:3000/callback
- Scopes: openid profile email
- Checks: state, pkce (S256)
- Token auth method: client_secret_post
```

### Step 9: Summary

Present a clear summary of everything that was set up:

```
## Authalla Setup Complete!

### Tenant
- Name: [tenant name]
- ID: [tenant_id]

### Theme
- Primary color: [color]

### Custom Domain (if configured)
- Domain: auth.example.com
- Status: Pending DNS verification

### Custom Email (if configured)
- Domain: mail.example.com
- Status: Pending DNS verification

### Social Login (if configured)
- Providers: Google, GitHub (etc.)
- Auth methods: magic_link, social_logins

### OAuth2 Client
- Name: [app name]
- Client ID: [client_id]
- Client Secret: [client_secret] (store this securely!)
- Type: [spa/web/native/backend]

### DNS Records to Add
[List all DNS records that need to be configured]

### Next Steps
1. Add the DNS records listed above to your DNS provider
2. Integrate the OIDC client into your application using the configuration above
3. Test the login flow at https://TENANT_ID.authalla.com
```

## Important Notes

- The CLI automatically manages access tokens (fetching, caching, and refreshing)
- Use `authalla <resource> schema <operation>` to see the full JSON schema for any create/update operation
- Use `authalla <resource> <operation> --help` for inline documentation with examples
- Never log or display the client_secret in full except when first presenting it to the user
- Be encouraging and celebrate progress after each successful step
