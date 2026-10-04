# Python: `mcp` with PyJWT

Checked against `mcp` 2.2 and `PyJWT` 2.14 (install `pyjwt[crypto]`).

Replace `ISSUER` (the tenant's issuer), `RESOURCE` (the identifier), the server name and the scopes. Keep the user's existing tools.

```python
import jwt
from jwt import PyJWKClient
from mcp.server.auth.provider import AccessToken, TokenVerifier
from mcp.server.auth.routes import create_protected_resource_routes
from mcp.server.auth.settings import AuthSettings
from mcp.server.mcpserver import MCPServer

ISSUER = "https://acme.authalla.com"
RESOURCE = "https://mcp.acme.com/mcp"
jwks = PyJWKClient(f"{ISSUER}/.well-known/jwks.json")


class AuthallaTokenVerifier(TokenVerifier):
    async def verify_token(self, token: str) -> AccessToken | None:
        try:
            if jwt.get_unverified_header(token).get("typ", "").lower() not in ("at+jwt", "application/at+jwt"):
                return None
            key = jwks.get_signing_key_from_jwt(token).key
            claims = jwt.decode(token, key, algorithms=["RS256"], audience=RESOURCE, issuer=ISSUER,
                                options={"require": ["exp", "sub", "client_id"]})
        except jwt.PyJWTError:
            return None  # the SDK answers 401 with WWW-Authenticate
        return AccessToken(token=token, client_id=claims["client_id"], scopes=claims.get("scope", "").split(),
                           expires_at=claims["exp"], resource=RESOURCE, subject=claims["sub"], claims=claims)


mcp = MCPServer(
    "Acme CRM",
    token_verifier=AuthallaTokenVerifier(),
    auth=AuthSettings(issuer_url=ISSUER, resource_server_url=RESOURCE,
                      required_scopes=["crm:read"],  # every request needs these
                      validate_token_resource=False),  # verify_token already checks aud
)

# Existing tools stay registered with @mcp.tool()

app = mcp.streamable_http_app(host="0.0.0.0")  # run: uvicorn server:app
# The built-in metadata lists only required_scopes; serve the full document instead.
app.router.routes[:0] = create_protected_resource_routes(
    resource_url=RESOURCE, authorization_servers=[ISSUER], resource_name="Acme CRM",
    scopes_supported=["openid", "crm:read", "crm:write"])
```

## Checking a write scope in a tool

```python
from mcp.server.auth.middleware.auth_context import get_access_token

@mcp.tool()
def update_deal_stage(id: str, stage: str) -> str:
    token = get_access_token()
    if token is None or "crm:write" not in token.scopes:
        raise PermissionError("This needs the crm:write permission.")
    # ... the tool's existing body
```

`token.subject` is the signed-in user's ID. Use it to scope data to that user.
