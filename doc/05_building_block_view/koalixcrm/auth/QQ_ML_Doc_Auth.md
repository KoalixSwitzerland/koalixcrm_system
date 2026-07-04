# Auth Package — Mid-Level Documentation

## Introduction

### Purpose

The `koalixcrm/auth/` package implements all authentication mechanisms for koalixcrm.
It covers three distinct authentication paths:

- **Browser-based OIDC session login** for the Django admin interface, using the
  Authorization Code Flow with PKCE.
- **DRF Bearer JWT authentication** for REST API endpoints, supporting both
  user-facing access tokens and machine-to-machine (Client Credentials Grant) tokens.
- **OpenAPI security scheme registration** so that generated API specifications
  accurately describe the authentication requirements of each endpoint.

### Contents Overview

| Component | Role |
|---|---|
| `oidc_utils` | Stateless shared utilities: discovery, JWKS fetching, JWT validation, header parsing |
| `OIDCAccessTokenAuthentication` | DRF authenticator for user-facing Bearer JWTs |
| `CeleryWorkerM2MAuthentication` | DRF authenticator for M2M Client Credentials tokens |
| `OIDCAuthenticationBackend` | Django auth backend for browser OIDC session login |
| Browser login views | `LoginSelectionView`, `OAuthLoginView`, `OAuthCallbackView`, `MultiProviderLogoutView` |
| `openapi_extensions` | `drf-spectacular` security scheme registrations |

### Target Audience

Software development engineers who need to understand how the authentication layer
is structured, how the components interact, and where to extend or modify behaviour.
For method-level detail, see [QQ_LL_Doc_Auth.md](QQ_LL_Doc_Auth.md).

### Glossary

| Term / Acronym | Full Form | Description |
|---|---|---|
| OIDC | OpenID Connect | Identity layer on top of OAuth 2.0 providing user authentication |
| JWT | JSON Web Token | Compact, URL-safe token format used to transmit claims between parties |
| JWKS | JSON Web Key Set | Published set of public keys used to verify JWT signatures |
| M2M | Machine-to-Machine | Non-interactive authentication using the OAuth 2.0 Client Credentials Grant |
| DRF | Django REST Framework | Python library for building REST APIs with Django |
| Bearer token | — | HTTP Authorization scheme: `Authorization: Bearer <token>` |
| PKCE | Proof Key for Code Exchange | OAuth 2.0 extension that binds an authorization code to the client that requested it |
| RS256 | RSA Signature with SHA-256 | Asymmetric JWT signing algorithm used by all supported OIDC providers |
| `at_hash` | Access Token Hash | JWT claim in an ID token binding it to a specific access token |
| `azp` | Authorized Party | JWT claim indicating the client to which the token was issued |

---

## Package Diagram

The following diagram groups the package components by responsibility and shows
the key dependency relationships between them.

```mermaid
flowchart TD
    subgraph utils["Shared Utilities"]
        OU[oidc_utils]
    end

    subgraph drf["DRF Authenticators"]
        OA[OIDCAccessTokenAuthentication]
        CM[CeleryWorkerM2MAuthentication]
    end

    subgraph views["Browser Login Views"]
        LS[LoginSelectionView]
        OL[OAuthLoginView]
        OC[OAuthCallbackView]
        ML[MultiProviderLogoutView]
    end

    subgraph backend["Django Auth Backend"]
        AB[OIDCAuthenticationBackend]
    end

    subgraph openapi["OpenAPI Extensions"]
        OE[openapi_extensions]
    end

    OA --> OU
    CM --> OU
    OC --> AB
    AB --> OU
    OE --> OA
    OE --> CM
```

Figure 1 — Auth package component groups and key dependencies

For method signatures, control-flow diagrams, and class-level detail, see
[QQ_LL_Doc_Auth.md](QQ_LL_Doc_Auth.md).

---

## Interaction Diagrams

### DRF API Request Authentication (Bearer JWT)

This sequence shows how a REST API request carrying a Bearer JWT is validated and
resolved to a Django user. `oidc_utils` is the shared utility module; cache hits
skip the network calls entirely.

```mermaid
sequenceDiagram
    participant C as API Client
    participant DRF as DRF Request Pipeline
    participant Auth as OIDCAccessTokenAuthentication
    participant Utils as oidc_utils
    participant Cache as Django Cache
    participant DB as UserModel

    C->>DRF: HTTP request with Authorization: Bearer <token>
    DRF->>Auth: authenticate(request)
    Auth->>Utils: get_token_auth_header(request)
    Utils-->>Auth: token string
    Auth->>Utils: get_jwks(OIDC_ISSUER)
    Utils->>Cache: cache.get(oidc_jwks_...)
    Cache-->>Utils: JWKS (hit) or None (miss)
    Utils-->>Auth: JWKS dict
    Auth->>Auth: jwt.decode RS256 + audience check
    Auth->>DB: _find_or_create_user(email, payload)
    DB-->>Auth: user instance
    Auth-->>DRF: (user, payload)
```

Figure 2 — DRF API request authentication sequence (Bearer JWT)

### Browser OIDC Login Flow

This sequence shows the browser-based Authorization Code Flow with PKCE. The IdP
interaction steps (authorization redirect and token exchange) are collapsed for
clarity.

```mermaid
sequenceDiagram
    participant B as Browser
    participant LS as LoginSelectionView
    participant OL as OAuthLoginView
    participant IdP as Identity Provider
    participant OC as OAuthCallbackView
    participant AB as OIDCAuthenticationBackend
    participant S as Django Session

    B->>LS: GET /auth/login/
    LS-->>B: Redirect to OAuthLoginView
    B->>OL: GET /auth/login/<provider>/
    OL-->>B: Redirect to IdP (PKCE code_challenge)
    B->>IdP: Authorization request + PKCE
    IdP-->>B: Redirect to /auth/callback/<provider>/?code=...
    B->>OC: GET /auth/callback/<provider>/
    OC->>IdP: Code exchange (token endpoint)
    IdP-->>OC: token_data (access_token, id_token)
    OC->>OC: _extract_user_info (3 fallback strategies)
    OC->>AB: authenticate(provider, user_info)
    AB-->>OC: user instance
    OC->>S: login(request, user)
    OC-->>B: Redirect to admin or next_url
```

Figure 3 — Browser OIDC login flow sequence

### M2M Authentication and Workspace Fixup

This sequence shows how a Celery worker or other M2M client is authenticated and
how the workspace gap left by middleware is repaired.

```mermaid
sequenceDiagram
    participant W as M2M Client
    participant DRF as DRF Request Pipeline
    participant CM as CeleryWorkerM2MAuthentication
    participant Utils as oidc_utils
    participant DB as UserModel
    participant WS as WorkspaceContext

    W->>DRF: HTTP request with Authorization: Bearer <m2m_token>
    DRF->>CM: authenticate(request)
    CM->>Utils: get_token_auth_header + unverified decode (iss/azp check)
    Utils-->>CM: token claims (pre-check)
    CM->>Utils: validate_jwt (full RS256 verification)
    Utils-->>CM: verified payload
    CM->>DB: get or create service user by client_id
    DB-->>CM: user instance
    CM->>WS: check request.active_workspace
    WS-->>CM: None (middleware could not set it)
    CM->>WS: assign first accessible workspace
    CM-->>DRF: (user, payload)
```

Figure 4 — M2M authentication and workspace fixup sequence

---

## Class Diagrams

The following diagram shows the inheritance hierarchy and key usage relationships
between the main classes in the package. Method bodies are omitted; see
[QQ_LL_Doc_Auth.md](QQ_LL_Doc_Auth.md) for full signatures.

```mermaid
classDiagram
    direction TB

    class oidc_utils {
        <<module>>
    }

    class OIDCAccessTokenAuthentication {
        +authenticate(request)
    }

    class CeleryWorkerM2MAuthentication {
        +authenticate(request)
    }

    class OIDCAuthenticationBackend {
        +authenticate(request, ...)
        +get_user(user_id)
    }

    class BrowserViews {
        <<group>>
        LoginSelectionView
        OAuthLoginView
        OAuthCallbackView
        MultiProviderLogoutView
    }

    class openapi_extensions {
        <<module>>
        OIDCAccessTokenAuthenticationScheme
        CeleryWorkerM2MAuthenticationScheme
    }

    class BaseAuthentication {
        <<external: DRF>>
    }

    class OpenApiAuthenticationExtension {
        <<external: drf-spectacular>>
    }

    OIDCAccessTokenAuthentication --|> BaseAuthentication
    CeleryWorkerM2MAuthentication --|> BaseAuthentication
    openapi_extensions --|> OpenApiAuthenticationExtension

    OIDCAccessTokenAuthentication --> oidc_utils : uses
    CeleryWorkerM2MAuthentication --> oidc_utils : uses
    OIDCAuthenticationBackend --> oidc_utils : uses
    BrowserViews --> OIDCAuthenticationBackend : calls authenticate
    openapi_extensions --> OIDCAccessTokenAuthentication : describes
    openapi_extensions --> CeleryWorkerM2MAuthentication : describes
```

Figure 5 — Auth package class relationships

---

## Design Patterns Used

**Strategy** — Django's `AUTHENTICATION_BACKENDS` setting holds an ordered list of
backend classes, each implementing `authenticate`. Django iterates them and uses the
first non-`None` result. `OIDCAuthenticationBackend` and Django's built-in
`ModelBackend` coexist in this list, allowing OIDC-authenticated and
password-authenticated sessions to operate simultaneously.

**Chain of Responsibility** — DRF's `DEFAULT_AUTHENTICATION_CLASSES` applies the
same pattern for API requests. `CeleryWorkerM2MAuthentication`,
`OIDCAccessTokenAuthentication`, `SessionAuthentication`, and `BasicAuthentication`
are tried in order. Each returns `None` to pass the request on, or a `(user, token)`
tuple to claim it.

**Side-Effect Registration** — `koalixcrm/auth/__init__.py` imports
`openapi_extensions` solely for the side effect of registering
`OpenApiAuthenticationExtension` subclasses with `drf-spectacular`'s class registry.
No explicit registration call is needed at the call site.

**Singleton-like Module State** — The `authlib` `OAuth` instance in `oidc_views.py`
is created at module import time and holds the registered OIDC provider
configurations for the lifetime of the process. In a horizontally scaled deployment
each process maintains its own copy, which is correct because the object stores only
static configuration.

---

## External Dependencies

| Package | Minimum Version | Role in this package |
|---|---|---|
| `PyJWT` | `>=2.0` | JWT decoding and `PyJWK` signing-key construction in `oidc_utils` |
| `authlib` | `>=1.0` | Authorization Code + PKCE flow via `django_client.OAuth` in `oidc_views` |
| `djangorestframework` | `>=3.14` | `BaseAuthentication` base class and `AuthenticationFailed` exception |
| `drf-spectacular` | `>=0.27` | `OpenApiAuthenticationExtension` base class in `openapi_extensions` |
| `django` | `>=4.2` | Cache framework, auth framework, session middleware, ORM |

---

## Testing

Information not available.

---

## Appendix

### References

- [QQ_LL_Doc_Auth.md](QQ_LL_Doc_Auth.md) — Low-level documentation with full method
  signatures, control-flow diagrams, and security asset inventory for the
  `koalixcrm/auth/` package.

### List of Illustrations

| Figure | Title |
|---|---|
| Figure 1 | Auth package component groups and key dependencies |
| Figure 2 | DRF API request authentication sequence (Bearer JWT) |
| Figure 3 | Browser OIDC login flow sequence |
| Figure 4 | M2M authentication and workspace fixup sequence |
| Figure 5 | Auth package class relationships |
