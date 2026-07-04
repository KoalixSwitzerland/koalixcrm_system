# Auth Package — Low-Level Documentation

## Introduction

### Scope

This document covers the `koalixcrm/auth/` package, which implements all authentication
mechanisms used by koalixcrm. The following source files are described:

| File | Purpose |
|------|---------|
| `oidc_utils.py` | Shared OIDC discovery, JWKS fetching, JWT validation, and header parsing utilities |
| `oidc_token_authentication.py` | DRF `BaseAuthentication` subclass for validating OIDC access tokens (Bearer JWT) in REST API requests |
| `m2m_authentication.py` | DRF `BaseAuthentication` subclass for machine-to-machine (Client Credentials Grant) JWT authentication |
| `oidc_backend.py` | Django authentication backend for browser-based OIDC session login |
| `oidc_views.py` | Django views implementing the browser OIDC login flow (selection, redirect, callback, logout) |
| `openapi_extensions.py` | `drf-spectacular` `OpenApiAuthenticationExtension` registrations for the two authenticators |
| `__init__.py` | Package entry point; imports `openapi_extensions` to trigger side-effect registration |

### Target Audience

Software development engineers who need to understand, modify, or extend the authentication
layer of koalixcrm.

### Glossary

| Term/Acronym | Full Form | Description |
|---|---|---|
| OIDC | OpenID Connect | Identity layer on top of OAuth 2.0 providing user authentication |
| JWT | JSON Web Token | Compact, URL-safe token format used to transmit claims between parties |
| JWKS | JSON Web Key Set | Published set of public keys used to verify JWT signatures |
| M2M | Machine-to-Machine | Non-interactive authentication using the OAuth 2.0 Client Credentials Grant |
| DRF | Django REST Framework | Python library for building REST APIs with Django |
| Bearer token | — | HTTP Authorization scheme: `Authorization: Bearer <token>` |
| `at_hash` | Access Token Hash | JWT claim in an ID token binding it to a specific access token |
| `azp` | Authorized Party | JWT claim indicating the client to which the token was issued |
| `sub` | Subject | JWT claim that identifies the principal (user or service) |
| `kid` | Key ID | JWKS claim identifying which public key was used to sign a JWT |
| RS256 | RSA Signature with SHA-256 | Asymmetric JWT signing algorithm used by all supported OIDC providers |

---

## Detailed Components

### `oidc_utils` — Shared OIDC Utilities

```mermaid
classDiagram
    direction LR

    namespace auth {
        class oidc_utils {
            +get_oidc_discovery(issuer_url) dict
            +get_jwks(authority_url) dict
            +validate_jwt(token, authority_url, access_token, client_id) dict
            +get_token_auth_header(request) str
            -_verify_at_hash(at_hash, access_token) None
        }
    }

    class DjangoCache:::external {
        <<external: django.core.cache>>
    }
    class HttpRequest:::external {
        <<external: django.http>>
    }
    class jwt:::external {
        <<external: PyJWT>>
    }

    oidc_utils --> DjangoCache : cache.get / cache.set
    oidc_utils --> HttpRequest : get_token_auth_header
    oidc_utils --> jwt : decode / PyJWK

    classDef external fill:#eee,stroke:#999,stroke-dasharray:4 4;
```

**Caption: Figure 1 — oidc_utils module relationships**

This module is stateless and provides four public functions and one private helper.
All network I/O results are stored in Django's cache framework for one hour
(`JWKS_CACHE_TIMEOUT = OIDC_DISCOVERY_CACHE_TIMEOUT = 3600 s`) to avoid repeated
round-trips to the identity provider on every request.

#### `get_oidc_discovery(issuer_url)`

**Signature:** `get_oidc_discovery(issuer_url: str | None) -> dict[str, Any] | None`

Fetches the OIDC well-known discovery document from
`{issuer_url}/.well-known/openid-configuration`. Returns `None` on any network or
parse failure. Results are cached under `oidc_discovery_{hash(issuer_url)}`.

#### `get_jwks(authority_url)`

**Signature:** `get_jwks(authority_url: str | None) -> dict[str, Any] | None`

Retrieves the JSON Web Key Set. First calls `get_oidc_discovery` to find the
`jwks_uri`; if discovery fails it falls back to
`{authority_url}/.well-known/jwks.json`. Results are cached under
`oidc_jwks_{hash(authority_url)}`.

```mermaid
flowchart TD
    A([Start]) --> B{Cache hit?}
    B -->|Yes| Z([Return cached JWKS])
    B -->|No| C[Call get_oidc_discovery]
    C --> D{discovery has jwks_uri?}
    D -->|Yes| E[Use jwks_uri from discovery]
    D -->|No| F[Fall back to /.well-known/jwks.json]
    E --> G[HTTP GET JWKS URL]
    F --> G
    G --> H{Success?}
    H -->|Yes| I[cache.set + return JWKS]
    H -->|No| J([Return None])
    I --> Z
```

**Caption: Figure 2 — get_jwks control flow**

#### `validate_jwt(token, authority_url, access_token, client_id)`

**Signature:**
`validate_jwt(token: str, authority_url: str | None, access_token: str | None = None, client_id: str | None = None) -> dict[str, Any]`

Full JWT validation. Fetches the JWKS, matches the key by `kid`, constructs a
`PyJWK` signing key, and calls `jwt.decode` with RS256. When `client_id` is
provided it is passed as the expected audience; when omitted audience verification
is skipped (used for access tokens whose audience varies by provider). When
`access_token` is provided and the decoded payload contains `at_hash`, the hash is
verified via `_verify_at_hash`. Raises `AuthenticationFailed` on any validation
failure; never returns `None` — callers can trust the returned payload is authentic.

```mermaid
flowchart TD
    A([Start]) --> B[get_jwks]
    B --> C{JWKS retrieved?}
    C -->|No| E1([Raise AuthenticationFailed])
    C -->|Yes| D[get_unverified_header — extract kid]
    D --> F{kid present?}
    F -->|No| E2([Raise AuthenticationFailed])
    F -->|Yes| G[Find matching key in JWKS]
    G --> H{Key found?}
    H -->|No| E3([Raise AuthenticationFailed])
    H -->|Yes| I[Build PyJWK signing key]
    I --> J[jwt.decode RS256]
    J --> K{at_hash in payload AND access_token provided?}
    K -->|Yes| L[_verify_at_hash]
    K -->|No| M([Return payload])
    L --> M
```

**Caption: Figure 3 — validate_jwt control flow**

#### `get_token_auth_header(request)`

**Signature:** `get_token_auth_header(request: HttpRequest) -> str | None`

Extracts the token from the `Authorization: Bearer <token>` header. Returns `None`
if the header is absent, does not use the `Bearer` scheme, or has an unexpected
number of parts. This function is intentionally non-raising: callers use `None` to
signal that this authenticator does not apply to the request.

#### `_verify_at_hash(at_hash, access_token)` (private)

**Signature:** `_verify_at_hash(at_hash: str, access_token: str) -> None`

Computes SHA-256 of the access token, takes the first 16 bytes, and compares
the Base64url encoding against `at_hash`. Raises `jwt.InvalidTokenError` on
mismatch. This is the OIDC Core §3.3.2.9 binding check between an ID token and
its accompanying access token.

---

### `OIDCAccessTokenAuthentication`

```mermaid
classDiagram
    direction LR

    namespace auth {
        class OIDCAccessTokenAuthentication {
            +authenticate(request) tuple
            -_fetch_email_from_userinfo(issuer, access_token)$ str
            -_find_or_create_user(email, payload)$ AbstractBaseUser
        }
    }

    class BaseAuthentication:::external {
        <<external: rest_framework.authentication>>
    }
    class oidc_utils:::external {
        <<external: koalixcrm.auth.oidc_utils>>
    }
    class UserModel:::external {
        <<external: django.contrib.auth>>
    }

    OIDCAccessTokenAuthentication --|> BaseAuthentication
    OIDCAccessTokenAuthentication --> oidc_utils : get_jwks / get_oidc_discovery
    OIDCAccessTokenAuthentication --> UserModel : get / create_user

    classDef external fill:#eee,stroke:#999,stroke-dasharray:4 4;
```

**Caption: Figure 4 — OIDCAccessTokenAuthentication class**

This class is the DRF authenticator for all REST API endpoints. It validates Bearer
JWTs issued by any standards-compliant OIDC provider. The provider is identified by
`OIDC_ISSUER` in Django settings; accepted audiences are listed in
`OIDC_ACCEPTED_AUDIENCES`. Users are identified by email, not by `sub`, which allows
the same Django user to authenticate across multiple OIDC clients. Names are kept in
sync on each successful login, but never overwritten with empty values.

#### `authenticate(request)`

**Signature:** `authenticate(request: HttpRequest) -> tuple[AbstractBaseUser, dict] | None`

Returns `None` (unapplied) if the `Authorization` header is missing, the token
header is malformed, `OIDC_ISSUER` is not configured, or the `kid` in the token
header does not match any key in the JWKS. Raises `AuthenticationFailed` for
tokens that are structurally valid but fail signature, expiry, or audience checks.

```mermaid
flowchart TD
    A([Start]) --> B[get_token_auth_header]
    B --> C{Token present?}
    C -->|No| Z1([Return None])
    C -->|Yes| D[get_unverified_header]
    D --> E{Header valid?}
    E -->|No| Z1
    E -->|Yes| F{OIDC_ISSUER configured?}
    F -->|No| Z1
    F -->|Yes| G[get_jwks]
    G --> H[Find matching key by kid]
    H --> I{Key found?}
    I -->|No| Z1
    I -->|Yes| J[jwt.decode RS256]
    J --> K{Audience check passes?}
    K -->|No| Z1
    K -->|Yes| L{email in payload?}
    L -->|Yes| M[_find_or_create_user]
    L -->|No| N[_fetch_email_from_userinfo]
    N --> M
    M --> O{user.is_active?}
    O -->|No| E1([Raise AuthenticationFailed])
    O -->|Yes| Z2([Return user, payload])
```

**Caption: Figure 5 — OIDCAccessTokenAuthentication.authenticate flow**

#### `_fetch_email_from_userinfo(issuer, access_token)` (private, static)

Calls the OIDC `userinfo_endpoint` (discovered from the well-known document) with
the access token as a Bearer credential and extracts the `email` field. Returns
`None` on any network or parse failure. Used as a fallback when the access token
itself does not carry an `email` claim (e.g., opaque-style access tokens from
Cognito).

#### `_find_or_create_user(email, payload)` (private, static)

Looks up a `UserModel` by email. If found, syncs `first_name` / `last_name` from
`given_name` / `family_name` claims if they are non-empty and different. If not
found, auto-provisions a new user inside a `transaction.atomic()` block. Raises
`AuthenticationFailed` on `IntegrityError` (concurrent creation race).

---

### `CeleryWorkerM2MAuthentication`

```mermaid
classDiagram
    direction LR

    namespace auth {
        class CeleryWorkerM2MAuthentication {
            +authenticate(request) tuple
        }
    }

    class BaseAuthentication:::external {
        <<external: rest_framework.authentication>>
    }
    class oidc_utils:::external {
        <<external: koalixcrm.auth.oidc_utils>>
    }
    class UserModel:::external {
        <<external: django.contrib.auth>>
    }
    class user_workspaces:::external {
        <<external: koalixcrm.core.access>>
    }

    CeleryWorkerM2MAuthentication --|> BaseAuthentication
    CeleryWorkerM2MAuthentication --> oidc_utils : validate_jwt
    CeleryWorkerM2MAuthentication --> UserModel : get / create_user
    CeleryWorkerM2MAuthentication --> user_workspaces : workspace fixup

    classDef external fill:#eee,stroke:#999,stroke-dasharray:4 4;
```

**Caption: Figure 6 — CeleryWorkerM2MAuthentication class**

This class authenticates machine-to-machine clients using the OAuth 2.0 Client
Credentials Grant. The issuer is separate from the user-facing OIDC issuer and is
configured via `CELERY_WORKER_M2M_OIDC_ISSUER`. Identity is established via the
`client_id` or `azp` claim, which is mapped to a Django service user by username.
Service users are auto-provisioned on first contact.

A non-obvious responsibility of this authenticator is patching
`request.active_workspace` when it is `None`. Because `WorkspaceContextMiddleware`
runs before DRF authentication and sees `AnonymousUser`, it cannot set a workspace
for M2M requests. This authenticator repairs the gap by assigning the first
workspace accessible to the service user, enabling workspace-scoped viewsets to
function correctly for M2M clients.

#### `authenticate(request)`

**Signature:** `authenticate(request: HttpRequest) -> tuple[AbstractBaseUser, dict] | None`

Returns `None` at any early-exit condition: missing Bearer header, malformed JWT
header, unconfigured issuer, or token issued by a different issuer. Uses
unverified decoding to cheaply check `iss` and `azp`/`client_id` before performing
the full cryptographic validation via `validate_jwt`.

```mermaid
flowchart TD
    A([Start]) --> B[get_token_auth_header]
    B --> C{Token?}
    C -->|No| Z1([Return None])
    C -->|Yes| D[get_unverified_header]
    D --> E{Valid header?}
    E -->|No| Z1
    E -->|Yes| F{M2M issuer configured?}
    F -->|No| Z1
    F -->|Yes| G[Decode without verify — check iss]
    G --> H{iss matches M2M issuer?}
    H -->|No| Z1
    H -->|Yes| I{azp or client_id matches M2M_CLIENT_ID?}
    I -->|No| Z1
    I -->|Yes| J[validate_jwt full verification]
    J --> K[Extract client_id / azp from payload]
    K --> L{Django user exists?}
    L -->|Yes| M[Load user]
    L -->|No| N[Auto-provision service user]
    M --> O{user.is_active?}
    N --> O
    O -->|No| E1([Raise AuthenticationFailed])
    O -->|Yes| P{active_workspace is None?}
    P -->|Yes| Q[Assign first accessible workspace]
    P -->|No| R([Return user, payload])
    Q --> R
```

**Caption: Figure 7 — CeleryWorkerM2MAuthentication.authenticate flow**

---

### `OIDCAuthenticationBackend`

```mermaid
classDiagram
    direction LR

    namespace auth {
        class OIDCAuthenticationBackend {
            +authenticate(request, provider, user_info, id_token) AbstractBaseUser
            +get_user(user_id) AbstractBaseUser
            -_authenticate_with_user_info(provider, user_info) AbstractBaseUser
            -_authenticate_with_id_token(id_token) AbstractBaseUser
            -_find_or_create_user(email, user_info)$ AbstractBaseUser
            -_sync_groups_from_provider(user, claims)$ None
        }
    }

    class UserModel:::external {
        <<external: django.contrib.auth>>
    }
    class Group:::external {
        <<external: django.contrib.auth.models>>
    }
    class oidc_utils:::external {
        <<external: koalixcrm.auth.oidc_utils>>
    }

    OIDCAuthenticationBackend --> UserModel : get / create_user
    OIDCAuthenticationBackend --> Group : get_or_create
    OIDCAuthenticationBackend --> oidc_utils : validate_jwt

    classDef external fill:#eee,stroke:#999,stroke-dasharray:4 4;
```

**Caption: Figure 8 — OIDCAuthenticationBackend class**

This Django authentication backend handles session-based authentication for the
admin UI. It is called by `django.contrib.auth.authenticate()` after
`OAuthCallbackView` obtains user info from the OIDC provider. The primary path
(`_authenticate_with_user_info`) uses a standardized user-info dict supplied by the
view. A legacy secondary path (`_authenticate_with_id_token`) validates a raw ID
token directly; this is retained for backwards compatibility.

Group membership from the provider is merged additively into Django groups: existing
Django group assignments are never removed. Three claim locations are checked in
priority order: `cognito:groups`, `groups`, `realm_access.roles`.

#### `authenticate(request, provider, user_info, id_token)`

**Signature:**
`authenticate(request, provider=None, user_info=None, id_token=None, **kwargs) -> AbstractBaseUser | None`

Dispatches to `_authenticate_with_user_info` when `provider` and `user_info` are
both present, otherwise to `_authenticate_with_id_token` when `id_token` is
present. Returns `None` when neither condition is met, allowing Django to try the
next backend in `AUTHENTICATION_BACKENDS`.

#### `_sync_groups_from_provider(user, claims)` (private, static module function)

Additively merges provider-supplied group names into Django groups. Groups that
already exist are reused via `get_or_create`. This function is a module-level
private function (not a method) called from `_authenticate_with_user_info`.

---

### Browser Login Views (`oidc_views.py`)

```mermaid
classDiagram
    direction LR

    namespace auth {
        class LoginSelectionView {
            +get(request) HttpResponseBase
            +post(request) HttpResponseBase
        }
        class OAuthLoginView {
            +get(request, provider) HttpResponseBase
        }
        class OAuthCallbackView {
            +get(request, provider) HttpResponseBase
            -_extract_user_info(provider, token_data) dict
            -_normalize_claims(claims) dict
        }
        class MultiProviderLogoutView {
            +dispatch(request) HttpResponseBase
        }
    }

    class View:::external {
        <<external: django.views>>
    }
    class LogoutView:::external {
        <<external: django.contrib.auth.views>>
    }
    class oauth:::external {
        <<external: authlib.integrations.django_client>>
    }
    class OIDCAuthenticationBackend:::external {
        <<external: koalixcrm.auth.oidc_backend>>
    }

    LoginSelectionView --|> View
    OAuthLoginView --|> View
    OAuthCallbackView --|> View
    MultiProviderLogoutView --|> LogoutView
    OAuthLoginView --> oauth : authorize_redirect
    OAuthCallbackView --> oauth : authorize_access_token / userinfo
    OAuthCallbackView --> OIDCAuthenticationBackend : authenticate

    classDef external fill:#eee,stroke:#999,stroke-dasharray:4 4;
```

**Caption: Figure 9 — Browser OIDC login view classes**

These views implement the browser-based Authorization Code Flow with PKCE. The
OAuth client is registered at module load time using `authlib`'s `OAuth` registry
if `ADMIN_OIDC_ISSUER` is set in settings. The `code_challenge_method = 'S256'`
configuration enforces PKCE on all authorization requests. When `ADMIN_OIDC_ISSUER`
is not configured the views fall back to Django's built-in admin `LoginView`,
allowing local development and end-to-end tests to work without a live Keycloak
instance.

#### `LoginSelectionView`

Handles the admin login entry point (`/auth/login/`). When already authenticated,
redirects to `next` or the admin index. When OIDC is configured, redirects to
`OAuthLoginView`. Otherwise renders Django's admin login template, also handling
credential `POST` requests for the fallback path.

#### `OAuthLoginView`

Handles `GET /auth/login/<provider>/`. Stores any `next` URL in the session, builds
the absolute callback URI using `SITE_URL` if available, and calls
`oauth_client.authorize_redirect` to send the browser to the identity provider.

#### `OAuthCallbackView`

Handles `GET /auth/callback/<provider>/`. Exchanges the authorization code for
tokens, extracts user info via three fallback strategies (token response
`userinfo`, userinfo endpoint call, ID token claims), authenticates the user,
calls `login(request, user)`, and redirects to the stored `next` URL or the admin
index. Access to the admin is guarded: non-staff users are logged out and receive
HTTP 403.

```mermaid
flowchart TD
    A([GET /auth/callback/provider]) --> B[authorize_access_token]
    B --> C[_extract_user_info]
    C --> D{email found?}
    D -->|No| E1([HTTP 401])
    D -->|Yes| F[authenticate via OIDCAuthenticationBackend]
    F --> G{user returned?}
    G -->|No| E2([HTTP 401])
    G -->|Yes| H{user.is_active?}
    H -->|No| E3([HTTP 403])
    H -->|Yes| I[login — create session]
    I --> J{next_url contains /admin AND not staff?}
    J -->|Yes| K[logout + HTTP 403]
    J -->|No| L{next_url set?}
    L -->|Yes| M([Redirect to next_url])
    L -->|No| N{user.is_staff?}
    N -->|Yes| O([Redirect to admin:index])
    N -->|No| P([Redirect to /])
```

**Caption: Figure 10 — OAuthCallbackView.get control flow**

#### `_extract_user_info(provider, token_data)` (private)

Tries three sources in order: `token_data['userinfo']`, the userinfo endpoint, and
base64-decoded ID token claims. Returns `None` if no source yields an email.

#### `MultiProviderLogoutView`

Extends Django's `LogoutView`. After calling `logout(request)`, it discovers the
`end_session_endpoint` from the OIDC discovery document and redirects the browser
there with `post_logout_redirect_uri` and `client_id` parameters, enabling
federated single logout. Falls back to `login-selection` if the endpoint cannot be
discovered.

---

### `openapi_extensions.py`

```mermaid
classDiagram
    direction LR

    namespace auth {
        class CeleryWorkerM2MAuthenticationScheme {
            +target_class str
            +name str
            +get_security_definition(auto_schema) dict
        }
        class OIDCAccessTokenAuthenticationScheme {
            +target_class str
            +name str
            +get_security_definition(auto_schema) dict
        }
    }

    class OpenApiAuthenticationExtension:::external {
        <<external: drf_spectacular.extensions>>
    }

    CeleryWorkerM2MAuthenticationScheme --|> OpenApiAuthenticationExtension
    OIDCAccessTokenAuthenticationScheme --|> OpenApiAuthenticationExtension

    classDef external fill:#eee,stroke:#999,stroke-dasharray:4 4;
```

**Caption: Figure 11 — OpenAPI authentication extension classes**

Both classes subclass `OpenApiAuthenticationExtension` from `drf-spectacular`. Each
declares `target_class` pointing to the authenticator it describes. The
`get_security_definition` method returns an OpenAPI 3 `securityScheme` object of
type `http`, scheme `bearer`, format `JWT`. The registration takes effect as a
side effect of importing the module, which is triggered unconditionally by
`koalixcrm/auth/__init__.py`.

---

## Persistent Storage

The auth package does not own any database tables. User records and group
assignments are written to Django's built-in `auth_user` and `auth_group` tables
via Django's ORM.

---

## In-Memory State

The `oauth` module-level `OAuth` instance in `oidc_views.py` holds registered
provider configurations. This is process-level state initialised at Django startup.
In a horizontally scaled deployment every process holds its own copy, which is
correct because the object only stores static configuration.

JWKS and discovery documents are cached in Django's cache framework (not in-process
memory), making the cache entries shared across processes when a distributed cache
backend is used.

---

## Access to External Interfaces

| Interface | Type of Call | Expected Duration | Notes |
|-----------|-------------|-------------------|-------|
| OIDC discovery endpoint `/.well-known/openid-configuration` | Blocking HTTP GET (stdlib `urlopen`) | ~100–500 ms | Cached for 1 hour; only on cache miss |
| JWKS endpoint `/.well-known/jwks.json` | Blocking HTTP GET (stdlib `urlopen`) | ~100–500 ms | Cached for 1 hour; only on cache miss |
| OIDC userinfo endpoint | Blocking HTTP GET (stdlib `urlopen`) | ~100–500 ms | Only called when access token lacks `email` claim |
| OIDC token endpoint | Blocking HTTP POST via authlib | ~200–800 ms | Only during callback code exchange |
| OIDC end_session_endpoint | Browser redirect (no server-side call) | — | HTTP 302 issued to client browser |

---

## Security

### Assets

| Asset | Description | Security Measure | Assessment of Criticality |
|-------|-------------|------------------|---------------------------|
| `ADMIN_OIDC_CLIENT_SECRET` | Client secret for admin OIDC app | Read from environment variable at startup | Uncritical when delivered via secret manager; critical if logged or stored in version control |
| `CELERY_WORKER_M2M_CLIENT_SECRET` | Client secret for M2M client credentials grant | Read from environment variable at startup | Uncritical when delivered via secret manager |
| `OIDC_ISSUER` | Issuer URL used as the trust anchor for JWT validation | Read from environment variable; used as `issuer` in `jwt.decode` | Uncritical — publicly known URL; critical if tampered with |
| `CELERY_WORKER_M2M_OIDC_ISSUER` | Separate trust anchor for M2M tokens | Read from environment variable | Same criticality as `OIDC_ISSUER` |
| Session cookie | Django session after successful OIDC login | Django session framework; `HttpOnly` and `Secure` flags depend on deployment settings | Uncritical when TLS is enforced at the load balancer |
| JWT token (Bearer) | Short-lived access token in `Authorization` header | Validated cryptographically (RS256, expiry, issuer, audience) | Uncritical — time-limited, provider-signed |

---

## Design Patterns Used

**Strategy** — `AUTHENTICATION_BACKENDS` is a list of backend classes, each
implementing `authenticate`. Django iterates them in order, using the first
non-`None` result. `OIDCAuthenticationBackend` and Django's built-in
`ModelBackend` are both registered, allowing OIDC-authenticated and
password-authenticated sessions to coexist.

**Chain of Responsibility** — DRF's `DEFAULT_AUTHENTICATION_CLASSES` list applies
the same pattern for API requests: `CeleryWorkerM2MAuthentication`,
`OIDCAccessTokenAuthentication`, `SessionAuthentication`, and `BasicAuthentication`
are tried in order. Each returns `None` to pass, or a `(user, token)` tuple to
claim the request.

**Decorator / Side-Effect Registration** — `__init__.py` imports
`openapi_extensions` solely for the side effect of registering the
`OpenApiAuthenticationExtension` subclasses with `drf-spectacular`'s class registry.

**Singleton-like Module State** — The `authlib` `OAuth` instance in `oidc_views.py`
is module-level and configured once at import time.

---

## External Dependencies

| Requirement | Version/Details | Notes |
|-------------|-----------------|-------|
| `PyJWT` | `>=2.0` | JWT decoding and `PyJWK` key construction |
| `authlib` | `>=1.0` | `django_client.OAuth` for Authorization Code + PKCE flow |
| `djangorestframework` | `>=3.14` | `BaseAuthentication`, `AuthenticationFailed` |
| `drf-spectacular` | `>=0.27` | `OpenApiAuthenticationExtension` |
| `django` | `>=4.2` | Cache framework, auth framework, session middleware |

---

## Appendix

### References

- [drf-spectacular Authentication Extensions](https://drf-spectacular.readthedocs.io/en/latest/customization.html#authentication)
- [OIDC Core Specification §3.3.2.9 — at_hash](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowTokenValidation)
- [RFC 6749 — Client Credentials Grant](https://datatracker.ietf.org/doc/html/rfc6749#section-4.4)
- [authlib Django Integration](https://docs.authlib.org/en/latest/client/django.html)

### List of Illustrations

| Figure | Title |
|--------|-------|
| Figure 1 | oidc_utils module relationships |
| Figure 2 | get_jwks control flow |
| Figure 3 | validate_jwt control flow |
| Figure 4 | OIDCAccessTokenAuthentication class |
| Figure 5 | OIDCAccessTokenAuthentication.authenticate flow |
| Figure 6 | CeleryWorkerM2MAuthentication class |
| Figure 7 | CeleryWorkerM2MAuthentication.authenticate flow |
| Figure 8 | OIDCAuthenticationBackend class |
| Figure 9 | Browser OIDC login view classes |
| Figure 10 | OAuthCallbackView.get control flow |
| Figure 11 | OpenAPI authentication extension classes |
