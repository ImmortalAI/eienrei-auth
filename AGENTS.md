# AGENTS.md

Working agreement for AI agents contributing to **EienRei Auth**.

Source of truth for design decisions: `docs/eienrei-auth-architecture.md` (written in Russian). This file
restates its requirements as actionable rules. If the two disagree, the architecture document wins —
and the disagreement should be reported, not silently resolved.

Write all code, comments, commit messages and documentation in this repository in **English**.

---

## 1. What this project is

EienRei Auth is the single identity, authentication and authorization-grant service for the whole
EienRei ecosystem (Novelier included). It is a standalone service: own repository, own deployment, own
database, own lifecycle. It does not depend on any product.

Canonical address and OIDC issuer: `https://me.eienrei.com`. Treat it as a frozen part of the protocol
contract — it must not change after production launch.

Long-term target: a full identity platform — Identity Provider, Authorization Server and Identity
Broker for local accounts, passkeys, external OAuth/OIDC providers, corporate directories and
machine-to-machine clients.

### In scope

- User registration and account lifecycle
- Storage of local credentials
- Authentication through multiple methods
- Ecosystem profile: display name, avatar, email, shared settings
- SSO across EienRei products
- Issuing and refreshing `access_token`, `id_token`, `refresh_token`
- User session management
- Linked sign-in methods and external identity providers
- Consent for OAuth clients
- Security event audit
- Sign-in / registration / recovery / Security Center pages on `me.eienrei.com`
- Later: Developer Portal for OAuth/OIDC client registration
- Later: federation with external OIDC / AD FS / Windows AD sources

### Out of scope — never implement here

- Product-internal roles and permissions (`Author`, `Reader`, `Moderator`, …)
- Product licences and subscriptions
- Product business data or project content
- Any link between an account and product entities beyond a stable subject identifier

Products never know about each other. Each product registers as a separate OAuth client
(`client_id`) with its own redirect URIs, post-logout redirect URIs, allowed grant types, allowed
scopes, permitted audiences/resources and client authentication policy.

---

## 2. Stack — do not substitute without an explicit decision

| Layer | Choice |
|---|---|
| Backend | C# / .NET 10, ASP.NET Core |
| OAuth/OIDC server | **OpenIddict** |
| User management | ASP.NET Core Identity |
| Database | PostgreSQL (dedicated identity DB, isolated from product DBs) |
| UI | Separate Nuxt frontend **or** Razor Pages — decision still open (§10) |
| Observability | OpenTelemetry + Serilog |
| Reverse proxy | Existing Traefik infrastructure (TLS and ingress stay an infra concern) |

Explicitly rejected, do not propose again without new information:

| Rejected | Reason |
|---|---|
| Duende IdentityServer | Commercial licence and dependency on a commercial model |
| Keycloak / Authentik / Zitadel / Ory | Foreign runtime/stack; the project also exists to deepen .NET expertise |
| Hand-rolled OAuth2/OIDC | Unacceptable security risk for minimal product value |
| Resource Owner Password Flow | Applications must never see the user's password |
| Implicit Flow | Obsolete and less secure browser flow |

**Never hand-implement OAuth 2.0 / OpenID Connect protocol mechanics.** Build on OpenIddict. A bug here
compromises the entire ecosystem, not one application.

---

## 3. Endpoints

Protocol endpoints use standard OpenIddict routes:

```text
/.well-known/openid-configuration
/connect/authorize
/connect/token
/connect/logout
/connect/userinfo
/connect/introspect      # if needed
/connect/revoke          # if needed
/connect/par             # once PAR is enabled
```

JWKS is published through OpenIddict discovery metadata.

User-facing pages stay separate from protocol endpoints:

```text
/login  /register  /recovery
/account  /account/security  /account/sessions
/account/applications  /account/activity
/developers
```

---

## 4. Client types and flows

- **Web applications (primary scenario):** Authorization Code + PKCE + BFF. Application JavaScript must
  never have direct access to a refresh token. For applications with their own backend, access and
  refresh tokens should live server-side only, and the browser works against a protected BFF session.
  Frontends must not store OAuth tokens in `localStorage`/`sessionStorage`; they use an HttpOnly,
  Secure, SameSite session cookie.
- **Native / desktop / mobile:** Authorization Code + PKCE, no client secret.
- **CLI and input-constrained devices (later):** OAuth 2.0 Device Authorization Grant.
- **Machine-to-machine:** Client Credentials Flow.

Canonical web flow:

```text
1. User clicks "Sign in" in the product.
2. Product backend/BFF generates state, nonce, code_verifier/code_challenge.
3. Browser is redirected to /connect/authorize?...&code_challenge=...
4. EienRei Auth checks its own SSO session.
5. The user authenticates if required.
6. Auth returns the browser to redirect_uri?code=...&state=...
7. Product backend/BFF exchanges the code + code_verifier for tokens.
8. Backend validates issuer/audience/signature/nonce and creates a local web session.
9. Frontend works through the HttpOnly session cookie only.
```

### Client authentication

Do not rely solely on long-lived `client_secret` values for confidential clients.

| Level | Client authentication |
|---|---|
| Public | PKCE, no secret |
| Standard confidential | Secret or private key |
| Preferred confidential | `private_key_jwt` |
| High security | Sender-constrained credentials / mTLS when required |

Internal BFF web applications should move to `private_key_jwt` once key management infrastructure exists.

---

## 5. Identity model and the product boundary

- The internal permanent `UserId` is **never** handed to products.
- Products receive **pairwise subject identifiers**, so two products cannot correlate the same user on
  their own.
- A product receives only: `sub` (pairwise, per client/sector), `iss` (`https://me.eienrei.com`), `aud`
  (target resource/server), permitted standard claims, `scope`, optionally `amr` and `acr`, and as
  little else as that product actually needs.
- The product creates its own local user entity keyed by the EienRei subject.
- Roles and product permissions are **never** stored in EienRei Auth.

### Audience restriction

Every access token targets one resource server or an explicitly limited set of resources. A token issued
for the Novelier API must not be accepted by any other EienRei service. Resource servers must validate
at minimum: signature, issuer, audience, lifetime, allowed scopes/permissions.

### Scopes

Scopes express permitted OAuth access, not internal product roles. Standard: `openid`, `profile`,
`email`, `offline_access`. Product scopes look like `novelier.api`, `novelier.read`, `novelier.write`.
The product still makes the final authorization decision on its own side.

---

## 6. Tokens, sessions, logout

**Token lifetimes.** Never stretch access-token lifetime to cover multi-week user sessions.

```text
Access token:        ~5–15 minutes
Refresh token:       substantially longer
Browser/BFF session: configurable
Remember me:         separate policy
```

Exact values are chosen after threat modelling — do not invent them ad hoc.

**Refresh rotation.** Every successfully used refresh token is replaced. If a previously used token
reappears: revoke the whole refresh-token family, mark the session suspicious/compromised, require
reauthentication, emit a security event/notification. A refresh-token family is bound to one user
session and one client.

**Session model.** Session, grant and token are distinct entities with independent lifecycles.

```text
User
 └── Session (SessionId, CreatedAt, LastSeenAt, device metadata,
              IP / coarse location, UserAgent, auth context, risk state)
      └── Grants / TokenFamilies (per client)
```

This must independently support: log out of one product (revoke that client's grant/token family), log
out one device (revoke one session), log out all other devices, log out everywhere, and revoke an
application's access (drop its consent/grant).

**Logout.** Use standard OIDC mechanisms, never a bespoke protocol: RP-Initiated Logout, Back-Channel
Logout, and Front-Channel Logout where needed. Prefer back-channel where supported — it does not depend
on open browser tabs.

---

## 7. Authentication

Do not model authentication as `Password + optional TOTP`. Introduce an **AuthenticationMethod**
abstraction; a user may hold several methods at once:

```text
Password, Passkey/WebAuthn, TOTP, RecoveryCode,
ExternalIdentity, EmailMagicLink (optional), WindowsIdentity (enterprise/federation)
```

**Passkeys are a first-class method.** Strategy: *passkey-first, password-compatible*. Support adding a
passkey to an existing account, multiple passkeys per user, passwordless login, potentially
password-free account creation, platform authenticators (Touch ID, Face ID, Windows Hello, Android) and
roaming FIDO2 security keys.

**Assurance and step-up.** Replace a single `2FA enabled` flag with an authentication context —
e.g. AAL1 (password / trusted external login), AAL2 (passkey, or password + TOTP), AAL3
(hardware-backed) — surfaced in tokens as `amr` and `acr` (`urn:eienrei:aal2`). Sensitive operations may
require step-up authentication: changing email, changing/recovering a password, adding or removing a
passkey, viewing or regenerating recovery codes, deleting an account, creating high-privilege
credentials, changing security settings. Products may also request a higher assurance level through the
OIDC authentication request.

**External identities.** The domain term is **ExternalIdentity**, never `SocialLogin`. Initial
providers: Google, GitHub, Discord, Microsoft, Apple, GitLab. A matching email address is **not**
sufficient grounds for automatic account merging — require authentication into the existing EienRei
account first, then link explicitly. Users may unlink a provider as long as at least one valid sign-in
or recovery method remains.

**Federation.** EienRei Auth also acts as an Identity Broker: built-in providers, then generic OIDC
provider registration (issuer + client id + client authentication, metadata loaded from
`/.well-known/openid-configuration`), covering third-party IdPs, corporate identity platforms,
self-hosted OIDC servers and AD FS / Entra-compatible providers. Products always trust only EienRei Auth
and never integrate with external providers directly.

**Enterprise.** Treat LDAP/AD as part of federation, not a special-case login. Preference order:
generic OIDC / AD FS → Windows Integrated Authentication (Kerberos) for trusted internal environments →
LDAP/AD bind only if no better federation protocol exists. A domain password must never reach EienRei
products, and direct LDAP bind through the public login form is not a preferred design.

**Account recovery** is its own subsystem, not just "forgot password". Channels: passkey, recovery
codes, verified email, external identity, and (only if ever needed) a support/admin procedure. After a
high-risk recovery, a temporary security hold may apply to the most sensitive operations (changing
recovery email, deleting the account, regenerating recovery codes, removing all authentication methods,
creating privileged credentials).

---

## 8. Security surfaces

**Security Center** (`/account/security`) is the central user-facing security page: available
authentication methods, passkeys, TOTP/MFA state, recovery codes, external identities, active sessions,
recent security activity, connected applications, and emergency recovery/logout actions.

**Connected applications** (`/account/applications`) lists OAuth/OIDC clients the user granted access
to, with permissions, grant date and last use. Revoking consent must also revoke the related grants and
refresh-token families. First-party internal applications may have their own consent policy.

**SecurityEvent is a domain concept, not a log line.** Examples:

```text
LOGIN_SUCCEEDED, LOGIN_FAILED, PASSWORD_CHANGED, PASSWORD_RECOVERED,
PASSKEY_ADDED, PASSKEY_REMOVED, TOTP_ENABLED, TOTP_DISABLED,
RECOVERY_CODES_REGENERATED, EXTERNAL_IDENTITY_LINKED, EXTERNAL_IDENTITY_UNLINKED,
NEW_SESSION, SESSION_REVOKED, APPLICATION_AUTHORIZED, APPLICATION_REVOKED,
REFRESH_TOKEN_REUSE_DETECTED, HIGH_RISK_LOGIN
```

Critical events may trigger email notifications (other channels later). IP and location metadata are
security telemetry and are retained for a limited time per the chosen privacy policy.

**Risk-based authentication** (late-stage): signals include known/unknown device, sign-in history,
coarse geography, ASN/network reputation, travel velocity between login events, failed-attempt count,
unusual authentication method, refresh-token reuse indicators, and the sensitivity of the requested
operation, producing Low/Medium/High. The risk engine must never take irreversible decisions on its own
— it raises identity-confirmation requirements and emits security events.

**Developer Portal** (`/developers`, later) manages OAuth clients: create an application, obtain
`client_id`, choose client type, configure redirect and post-logout URIs, allowed grants, scopes and
resources, client authentication, key management, basic audit info, credential revoke/rotate. Early
versions may be admin-only.

**Account lifecycle** (`/account`): profile editing, primary email change, password/security management,
connected applications, connected external identities, active sessions, account data export, account
deletion. Deleting an EienRei identity does not by itself physically delete product data — there must be
an explicit lifecycle/notification contract between Auth and products, and product business data must
never flow back into Auth to implement deletion.

---

## 9. Observability, logging, keys

Security audit and operational telemetry are separate streams.

- **Operational telemetry** via OpenTelemetry: request traces, latency, error rates, database metrics,
  token endpoint metrics, external provider latency, health checks.
- **Security audit**: significant security events stored separately, protected from accidental
  modification or deletion within available infrastructure.

**Logs must never contain:** passwords, raw access tokens, raw refresh tokens, authorization codes, TOTP
secrets, recovery codes, private keys or client secrets. Treat this as a hard review gate on every
change that touches logging.

**Cryptographic keys** are a critical infrastructure secret: never commit production private keys;
support rotation; publish old and new public keys in JWKS simultaneously during migration; do not
invalidate all active tokens instantly without cause; keep a backup/recovery procedure; consider an
external secret manager / HSM / KMS as infrastructure grows.

---

## 10. Roadmap — build in this order

Security-sensitive features are never added for feature count. Each is preceded by tests and threat
modelling.

- **v1 — Core Identity Provider:** .NET 10 + ASP.NET Core, OpenIddict server, ASP.NET Core Identity,
  PostgreSQL, canonical issuer, local account + password, Authorization Code + PKCE, BFF-oriented web
  flow, audiences/resources, short-lived access tokens, refresh rotation, session model, product/session/
  global logout, baseline SecurityEvent audit, basic account management.
- **v1.1 — Modern Authentication:** passkeys/WebAuthn, passwordless login, TOTP, recovery codes,
  AuthenticationMethod model, initial step-up.
- **v1.2 — External Identities:** Google, GitHub, Discord, Microsoft, optional Apple/GitLab, secure
  linking/unlinking, external-provider security events.
- **v1.3 — Security Center:** active sessions UI, revoke session, log out all other sessions, recent
  activity, security notifications, connected applications, revoke application access.
- **v1.4 — Identity Federation:** generic OIDC provider, AD FS, enterprise provider configuration, first
  broker administration UI.
- **v1.5 — Developer Platform:** Developer Portal, client registration UI, redirect/scope/resource
  management, key/credential rotation, consent management.
- **v2 — Enterprise and Machine Identity:** Windows Integrated Authentication / Kerberos where
  appropriate, optional LDAP/AD fallback, Device Authorization Flow, machine identities, Client
  Credentials, richer audit/administration.
- **v2+:** PAR, advanced step-up policies, risk-based authentication, sender-constrained tokens,
  mTLS/DPoP where justified, deeper security analytics.

PAR is not a v1 requirement, but the architecture must not obstruct enabling it later. The same holds
for sender-constrained tokens, mTLS and DPoP — introduce them only once a real threat model justifies
their operational cost.

### Machine identities

Principals split into at least `Human` and `Service`. A machine identity must not impersonate a user:

```json
{ "sub": "service:novelier-storyteller", "aud": "novelier-api", "scope": "story.manage" }
```

Service principals (Novelier Storyteller, background workers, scheduled jobs, CI/CD, monitoring and
automation clients, future internal services) receive no user profile claims and take no part in the
user session model. They have their own credentials, scopes, audiences, rotation policies and audit
events.

### Open questions — do not settle these unilaterally

Ask before designing around any of them: exact token/session lifetimes; UI technology (Razor Pages vs
Nuxt); pairwise subject sector policy (per client or per group of related first-party clients); consent
policy for first-party applications; account-deletion orchestration; email provider and transactional
notifications; risk-data retention and privacy; recovery security hold scope and duration; whether
direct LDAP bind is needed at all; when Docker secrets/environment variables become insufficient and a
Vault/KMS-like layer is required.

---

## 11. Core principles

1. Identity is centralized; product authorization is decentralized.
2. Products do not receive user passwords or refresh tokens unless genuinely necessary.
3. The browser is not a safe store for OAuth credentials; web applications are BFF-oriented.
4. Access tokens are short-lived; long sessions come from a protected refresh/session lifecycle.
5. Every token carries a correct issuer and audience.
6. Products receive a minimal claim set.
7. Pairwise subject identifiers reduce cross-product user correlation.
8. Passkeys are a first-class authentication method.
9. External login is identity federation, not a pile of special cases.
10. LDAP/AD belongs to enterprise federation; standard OIDC/Kerberos mechanisms are preferred.
11. Session, grant and token are distinct entities with independent lifecycles.
12. Any sensitive action may require step-up authentication.
13. Security events are part of the domain model, not just application logs.
14. Machine identities are separate from human identities.
15. Advanced security mechanisms are adopted per threat model, never for their own sake.

---

## 12. Repository state and commands

Current state: fresh ASP.NET Core scaffold. `EienRei.Auth.slnx` → `EienRei.Auth/EienRei.Auth.csproj`
(net10.0, nullable and implicit usings enabled), `Program.cs` still a minimal "Hello World" host.
Packages already referenced: OpenIddict.AspNetCore / OpenIddict.EntityFrameworkCore, ASP.NET Core
Identity + EF Core, Npgsql, OpenTelemetry (ASP.NET Core / HTTP / OTLP), Serilog, NpgSql health checks.
Architecture documents live in `docs/`.

```bash
dotnet restore
dotnet build
dotnet run --project EienRei.Auth
dotnet test          # once test projects exist
```

Keep `docs/eienrei-auth-architecture.md` and this file in sync when an architectural decision changes.
