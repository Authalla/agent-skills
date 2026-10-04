# Implement the Authalla integration

Understand the user's application before writing integration code. A wrong integration is worse than none.

Use a standard, well-maintained OAuth2/OIDC library for the stack, pointed at the tenant's discovery document.

## Security policy

Every integration this skill writes follows OAuth 2.1. When in doubt, pick the more secure option.

- **Authorization Code + PKCE (S256)**, for every client type. Authalla requires PKCE for public apps and accepts it from confidential ones; send it from both. Implicit, password and `client_credentials` grants are out for user sign-in.
- **`state` on every authorization request**, checked on the callback. Authalla rejects requests without it. Send a `nonce` too and check it in the ID token.
- **Exact redirect URIs**, registered on the app as used. Non-localhost redirect URIs use HTTPS.
- **Tokens travel in the `Authorization` header or a POST body**, never in a URL.
- **Client secrets stay server-side.** Confidential apps authenticate at the token endpoint with `client_secret_post` or `client_secret_basic`, one or the other in a request (Authalla refuses both at once). The library's default is fine.
- **Refresh tokens rotate.** Each refresh returns a new refresh token and invalidates the old one; replaying an old one revokes the whole family. Store the new token after every refresh, and serialize refreshes so two requests don't race.

### Token storage

- **Server-side apps**: tokens live in the server-side session only. The browser gets an `HttpOnly`, `Secure`, `SameSite=Lax` session cookie and never sees `access_token` or `refresh_token`.
- **Browser-only SPAs**: tokens live in memory. A reload means a new authorization redirect; to keep users signed in across reloads, move token handling to a backend (backend-for-frontend) instead of persisting tokens in the browser.
- **Mobile/native**: the platform's secure storage (Keychain, Keystore, the OS credential manager).

Existing code that breaks this policy (tokens in `localStorage`/`sessionStorage` or URLs, implicit flow, missing PKCE or `state`, secrets in browser code, wildcard redirect URIs): flag it to the user and replace it with the secure equivalent.

## 1. Discover the codebase

Build the full picture:

- **Stack**: framework config files and package manifests (`package.json`, `go.mod`, `pyproject.toml`, ...), monorepo layout, deployment (`Dockerfile`).
- **Existing auth**: auth libraries (Auth.js, passport, oidc-client-ts, Auth0, Clerk, Supabase, Firebase, Lucia, iron-session, ...), auth config files, OAuth/OIDC code (`code_verifier`, `/.well-known/openid-configuration`), JWT handling, session handling, auth middleware and guards.
- **Architecture**: routing pattern, protected routes, app entry point and layout, API routes, how env vars are loaded and named (`.env.example`).
- **Integration points**: where the app checks sign-in, where it sends signed-out users, how it stores session and user data, which user fields it reads (name, email, avatar, roles).

**Done when** you can name the stack, the existing auth (or none), the session mechanism, every protected route, and each place that reads the user.

## 2. Propose the approach

Choose by what exists:

- **No auth**: add it with the stack's standard library: config, callback route, sign-in and sign-out handlers, route protection, env template.
- **A library that takes custom OIDC providers** (Auth.js, passport, authlib): add Authalla as a provider and keep the existing structure. If it replaces another provider, remove that one cleanly.
- **A vendor-locked SDK** (Auth0, Clerk, Supabase Auth, Firebase Auth): this is a migration, not a new provider. List the files that change, and migrate in order: auth config, then middleware, then UI.
- **Hand-rolled auth**: integrate with it if it is sound; if it breaks the security policy, recommend replacing it and say why.

Present it:

```
## Codebase analysis
**Stack:** Next.js 15 (App Router), TypeScript
**Existing auth:** none
**Sessions:** none
**Protected routes:** /dashboard/*, /settings
**Approach:** Auth.js v5 with Authalla as an OIDC provider; middleware protects /dashboard and /settings

Shall I proceed?
```

**Done when** the user has confirmed the approach. Wait for it.

## 3. Read the discovery document

```bash
curl -s https://{tenant-id}.authalla.com/.well-known/openid-configuration
```

Use the custom domain instead if it is `active`. Configure the library with the issuer, not hand-copied endpoints; the library reads the rest. Expect `code_challenge_methods_supported` to include `S256` and `response_types_supported` to include `code`.

**Done when** the document loads and its `issuer` is the URL you'll configure. If it fails or differs, stop and tell the user.

## 4. Write the code

Follow the project's conventions (naming, style, imports, folder structure). For the stack's library and configuration, read [frameworks.md](frameworks.md).

Every integration has:

- **Auth configuration**: issuer, client ID, client secret (confidential apps only), redirect URI, scopes `openid profile email` (plus `offline_access` for refresh tokens, which the app must allow).
- **Callback route**: checks `state`, exchanges the code with the `code_verifier`, validates the ID token (signature from `jwks_uri`, `iss`, `aud` contains the client ID, `exp`, `nonce`), creates the session, then redirects to a `returnTo` stored before sign-in, not one taken from the query.
- **Sign-out**: destroys the local session first, then redirects to the `end_session_endpoint` with `client_id` and `post_logout_redirect_uri`. Without `client_id`, Authalla shows its sign-in page instead of redirecting back. `post_logout_redirect_uri` must match one of the app's `allowed_logout_uris` exactly; omitted, Authalla uses the first one.
- **Route protection**: in the app's existing pattern (middleware, guards, server checks). Public routes as an allowlist; everything else requires sign-in.
- **Environment variables**: added to the env example file with placeholders, in the project's naming convention:

  ```bash
  AUTHALLA_ISSUER=https://{tenant-id}.authalla.com
  AUTHALLA_CLIENT_ID=
  AUTHALLA_CLIENT_SECRET=   # confidential apps only
  ```

- **User mapping**: map claims to the app's user model. `sub` is the stable user ID. `email` and `email_verified` come with the `email` scope; `name` (when set) and `picture` (from a social login only) with `profile`. Tell the user about fields the app expects that Authalla doesn't provide.

**Done when** every item above exists in the code.

## 5. Audit

Check the code you wrote:

1. PKCE: `code_challenge` (S256) sent, `code_verifier` used in the exchange.
2. `state` and `nonce` generated, sent and checked.
3. `response_type` is `code`.
4. No token in a URL.
5. Token storage follows the policy for the app type.
6. Redirect and logout URIs in the code match the app's registered ones exactly (`get_app`); non-localhost ones use HTTPS.
7. The client secret appears only in server-side code and gitignored env files.
8. The ID token's signature, `iss`, `aud` and `exp` are checked.
9. Sign-out destroys the local session and calls the end-session endpoint with `client_id`.
10. Refresh (if used) stores the rotated refresh token.

**Done when** all ten pass. Fix any that fail before reporting back.
