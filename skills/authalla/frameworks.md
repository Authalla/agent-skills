# Framework setups

Starting points for common stacks. Each one points the library at the issuer and lets it read the discovery document. Check the library's current docs for the installed version before copying; APIs shift between major versions.

## Next.js (Auth.js v5)

```ts
// auth.ts
import NextAuth from "next-auth"

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    {
      id: "authalla",
      name: "Authalla",
      type: "oidc",
      issuer: process.env.AUTH_AUTHALLA_ISSUER,
      clientId: process.env.AUTH_AUTHALLA_ID,
      clientSecret: process.env.AUTH_AUTHALLA_SECRET,
      checks: ["pkce", "state", "nonce"],
    },
  ],
})
```

- The redirect URI to register on the app is `{app URL}/api/auth/callback/authalla`.
- Sign-out: build the end-session URL (`client_id` and `post_logout_redirect_uri`) in a server action, call `signOut({ redirectTo: endSessionUrl })`, and add a `redirect` callback that allows the issuer's origin.
- Protect routes in `middleware.ts` with `auth()`; Pages Router apps use `auth()` in `getServerSideProps`.

## Express / Node.js (openid-client v6)

```ts
import * as client from "openid-client"

const config = await client.discovery(
  new URL(process.env.AUTHALLA_ISSUER!),
  process.env.AUTHALLA_CLIENT_ID!,
  process.env.AUTHALLA_CLIENT_SECRET!, // omit for public clients
)

// GET /login
app.get("/login", async (req, res) => {
  const code_verifier = client.randomPKCECodeVerifier()
  const state = client.randomState()
  const nonce = client.randomNonce()
  req.session.oauth = { code_verifier, state, nonce }
  const url = client.buildAuthorizationUrl(config, {
    redirect_uri: process.env.AUTHALLA_REDIRECT_URI!,
    scope: "openid profile email",
    code_challenge: await client.calculatePKCECodeChallenge(code_verifier),
    code_challenge_method: "S256",
    state,
    nonce,
  })
  res.redirect(url.href)
})

// GET /callback
app.get("/callback", async (req, res) => {
  const { code_verifier, state, nonce } = req.session.oauth
  const tokens = await client.authorizationCodeGrant(
    config,
    new URL(req.originalUrl, process.env.AUTHALLA_REDIRECT_URI),
    { pkceCodeVerifier: code_verifier, expectedState: state, expectedNonce: nonce },
  )
  req.session.user = tokens.claims()
  req.session.tokens = tokens // server-side session store only
  delete req.session.oauth
  res.redirect("/")
})
```

Sign-out: destroy the session, then redirect to `client.buildEndSessionUrl(config, { client_id, post_logout_redirect_uri })`.

## React SPA (oidc-client-ts / react-oidc-context)

```ts
import { AuthProvider } from "react-oidc-context"
import { InMemoryWebStorage, WebStorageStateStore } from "oidc-client-ts"

const oidcConfig = {
  authority: import.meta.env.VITE_AUTHALLA_ISSUER,
  client_id: import.meta.env.VITE_AUTHALLA_CLIENT_ID,
  redirect_uri: window.location.origin + "/callback",
  post_logout_redirect_uri: window.location.origin,
  scope: "openid profile email",
  // Tokens in memory; the default store is sessionStorage.
  userStore: new WebStorageStateStore({ store: new InMemoryWebStorage() }),
}
```

The app is a public client: no secret, PKCE on (the library's default). Its origin must be in the tenant's Allowed origins (SKILL.md step 7). Guard routes with a component that calls `signinRedirect()` when `useAuth().isAuthenticated` is false.

## Go (coreos/go-oidc + golang.org/x/oauth2)

```go
provider, err := oidc.NewProvider(ctx, os.Getenv("AUTHALLA_ISSUER"))
conf := oauth2.Config{
    ClientID:     os.Getenv("AUTHALLA_CLIENT_ID"),
    ClientSecret: os.Getenv("AUTHALLA_CLIENT_SECRET"),
    RedirectURL:  os.Getenv("AUTHALLA_REDIRECT_URI"),
    Endpoint:     provider.Endpoint(),
    Scopes:       []string{oidc.ScopeOpenID, "profile", "email"},
}
idTokenVerifier := provider.Verifier(&oidc.Config{ClientID: conf.ClientID})

// Login: store verifier, state and nonce in the session.
verifier := oauth2.GenerateVerifier()
authURL := conf.AuthCodeURL(state, oauth2.S256ChallengeOption(verifier), oidc.Nonce(nonce))

// Callback: check state, then exchange and verify.
token, err := conf.Exchange(ctx, r.URL.Query().Get("code"), oauth2.VerifierOption(verifier))
rawIDToken, _ := token.Extra("id_token").(string)
idToken, err := idTokenVerifier.Verify(ctx, rawIDToken) // then check idToken.Nonce
```

Protect routes with HTTP middleware that checks the session before calling the handler.

## Python (Authlib, Flask)

```python
from authlib.integrations.flask_client import OAuth

oauth = OAuth(app)
oauth.register(
    name="authalla",
    server_metadata_url=f"{AUTHALLA_ISSUER}/.well-known/openid-configuration",
    client_id=os.environ["AUTHALLA_CLIENT_ID"],
    client_secret=os.environ["AUTHALLA_CLIENT_SECRET"],
    client_kwargs={"scope": "openid profile email", "code_challenge_method": "S256"},
)

@app.route("/login")
def login():
    return oauth.authalla.authorize_redirect(url_for("callback", _external=True))

@app.route("/callback")
def callback():
    token = oauth.authalla.authorize_access_token()  # checks state, nonce and the ID token
    session["user"] = token["userinfo"]
    return redirect("/")
```

Django uses `authlib.integrations.django_client` the same way.

## Anything else

Use the stack's maintained OIDC client library. With none available, implement Authorization Code + PKCE by hand: a `code_verifier` of 43–128 unreserved characters, `code_challenge = BASE64URL(SHA256(code_verifier))` sent with `code_challenge_method=S256`, and the `code_verifier` in the token request. Validate the ID token with a maintained JWT library against the `jwks_uri` (RS256).
