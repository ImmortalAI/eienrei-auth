# EienRei Auth — архитектура

Единый центр идентификации, аутентификации и выдачи полномочий для всех проектов экосистемы EienRei (включая Novelier). Отдельный сервис, независимый от Novelier и любого другого продукта экосистемы: свой репозиторий, свой деплой, своя БД и свой жизненный цикл.

**Канонический адрес и issuer:** `https://me.eienrei.com`. Этот адрес считается стабильной частью протокольного контракта и не должен меняться после запуска production-среды.

**Причина существования отдельно от продуктов:** протокол логина, хранения паролей и выдачи токенов — решённая, многократно атакованная задача, а не специфика конкретного продукта. Ошибка здесь стоит дороже прочих, потому что компрометация identity означает компрометацию всей экосистемы, а не одного приложения. Поэтому OAuth 2.0/OpenID Connect не реализуются вручную, а строятся поверх проверенной протокольной библиотеки.

В долгосрочной перспективе EienRei Auth должен быть не только страницей входа, но и полноценной **identity platform**: Identity Provider, Authorization Server и Identity Broker для локальных аккаунтов, passkeys, внешних OAuth/OIDC-провайдеров, корпоративных каталогов и machine-to-machine клиентов.

---

## 1. Роль и границы

EienRei Auth — **Identity Provider (IdP) / Authorization Server (AS)** по OAuth 2.0 / OpenID Connect.

Он отвечает за:

- регистрацию пользователей и жизненный цикл аккаунта;
- хранение локальных учётных данных;
- аутентификацию пользователя различными методами;
- профиль экосистемы: отображаемое имя, аватар, email и общие настройки;
- SSO между продуктами EienRei;
- выдачу и обновление `access_token`, `id_token`, `refresh_token`;
- управление пользовательскими сессиями;
- управление привязанными способами входа и внешними identity providers;
- выдачу consent для OAuth clients;
- аудит событий безопасности;
- страницы входа, регистрации, восстановления и Security Center на `me.eienrei.com`;
- в перспективе — Developer Portal для регистрации OAuth/OIDC clients;
- в перспективе — federation с внешними OIDC/AD FS/Windows AD источниками.

Он **не отвечает** за:

- роли и права внутри конкретного продукта (`Author`, `Reader`, `Moderator` и т. п.);
- продуктовые лицензии и подписки;
- бизнес-данные продуктов;
- содержимое проектов;
- связь между аккаунтом и продуктовыми сущностями, кроме устойчивого subject identifier.

Каждый продукт экосистемы регистрируется как отдельный **OAuth client** (`client_id`) со своими:

- `redirect_uri`;
- `post_logout_redirect_uri`;
- разрешёнными grant types;
- разрешёнными scopes;
- допустимыми audiences/resources;
- политиками аутентификации клиента.

Продукты друг про друга не знают.

---

## 2. Канонические endpoints

Для протокольных endpoints используются стандартные OpenIddict-маршруты:

```text
https://me.eienrei.com/.well-known/openid-configuration
https://me.eienrei.com/connect/authorize
https://me.eienrei.com/connect/token
https://me.eienrei.com/connect/logout
https://me.eienrei.com/connect/userinfo
https://me.eienrei.com/connect/introspect      # при необходимости
https://me.eienrei.com/connect/revoke          # при необходимости
https://me.eienrei.com/connect/par             # после включения PAR
```

JWKS публикуется через discovery metadata OpenIddict.

Пользовательские страницы отделены от протокольных endpoints:

```text
/login
/register
/recovery
/account
/account/security
/account/sessions
/account/applications
/account/activity
/developers
```

---

## 3. Типы клиентов и OAuth flows

### 3.1. Web applications — основной сценарий

Для web-приложений экосистемы используется **Authorization Code Flow + PKCE + BFF**.

```text
Browser
   │
   │ HttpOnly + Secure + SameSite cookie
   ▼
Product Backend / BFF
   │
   │ access token
   ▼
Product APIs
```

JavaScript-код приложения не должен иметь прямого доступа к refresh token. Для приложений с собственным backend предпочтительно, чтобы access/refresh tokens также хранились только server-side, а browser работал с защищённой BFF-сессией.

Базовый flow:

```text
1. Пользователь нажимает «Войти» в продукте.
2. Product Backend/BFF генерирует state, nonce, code_verifier/code_challenge.
3. Browser перенаправляется на:
   /connect/authorize?...&code_challenge=...
4. EienRei Auth проверяет собственную SSO-сессию.
5. При необходимости пользователь аутентифицируется.
6. Auth возвращает browser на redirect_uri?code=...&state=...
7. Product Backend/BFF обменивает authorization code + code_verifier на tokens.
8. Backend валидирует issuer/audience/signature/nonce и создаёт локальную web-сессию.
9. Frontend работает через HttpOnly session cookie и не хранит OAuth tokens в localStorage/sessionStorage.
```

### 3.2. Native/Desktop/Mobile

Используется **Authorization Code + PKCE** без client secret.

### 3.3. CLI и устройства с ограниченным вводом

В перспективе поддерживается **OAuth 2.0 Device Authorization Grant**:

```text
eienrei login
→ открыть https://me.eienrei.com/device
→ ввести короткий код
→ подтвердить вход в браузере
```

### 3.4. Machine-to-machine

Для сервисов и automation используется **Client Credentials Flow**.

Machine identity не должна притворяться пользователем. Для неё вводится отдельный principal типа `Service`.

```json
{
  "sub": "service:novelier-storyteller",
  "aud": "novelier-api",
  "scope": "story.manage"
}
```

---

## 4. Client authentication

Для confidential clients не следует полагаться исключительно на долговечные `client_secret`.

Поддерживаемые уровни:

| Уровень | Client authentication |
|---|---|
| Public | PKCE, без секрета |
| Standard confidential | secret или private key |
| Preferred confidential | `private_key_jwt` |
| High security | sender-constrained credentials / mTLS при необходимости |

Внутренние web-приложения с BFF предпочтительно переводить на `private_key_jwt`, когда инфраструктура управления ключами будет готова.

---

## 5. Стек

| Слой | Выбор | Обоснование |
|---|---|---|
| Backend | C# / .NET 10, ASP.NET Core | основной backend-стек экосистемы |
| OAuth/OIDC server | **OpenIddict** | готовая реализация протокола, PKCE, refresh flow, discovery, JWKS, device flow, PAR и других расширений |
| User management | ASP.NET Core Identity | стандартная модель пользователей, password hashing, lockout, recovery и интеграция с passkeys |
| БД | PostgreSQL | отдельная identity-БД; изоляция от продуктовых БД |
| UI | отдельный Nuxt frontend или Razor Pages | UI не должен влиять на протокольное ядро; выбор фиксируется отдельно |
| Observability | OpenTelemetry + Serilog | единая telemetry-модель экосистемы |
| Reverse proxy | существующая инфраструктура Traefik | TLS и ingress остаются инфраструктурной ответственностью |

### Явно отклонено

| Решение | Причина |
|---|---|
| Duende IdentityServer | коммерческая лицензия и зависимость от коммерческой модели |
| Keycloak / Authentik / Zitadel / Ory | отдельный чужой runtime/stack; проект также используется для углубления .NET |
| Полностью самописный OAuth2/OIDC | слишком высокий security risk при минимальной продуктовой ценности |
| Resource Owner Password Flow | приложения не должны видеть пользовательский пароль |
| Implicit Flow | устаревший и менее безопасный browser-flow |

---

## 6. Identity model и граница с продуктами

Во внутренней БД существует постоянный `UserId`, который никогда не передаётся продуктам напрямую.

Для продуктов предпочтительно использовать **pairwise subject identifiers**.

```text
Internal UserId
   │
   ├── Novelier subject = opaque A
   ├── Product B subject = opaque B
   └── Product C subject = opaque C
```

Это предотвращает возможность двум продуктам самостоятельно сопоставить одного пользователя между собой.

Продукт получает только:

- `sub` — pairwise subject identifier для данного client/sector;
- `iss` — `https://me.eienrei.com`;
- `aud` — целевой resource/server;
- разрешённые standard claims;
- `scope`;
- при необходимости `amr` и `acr`;
- минимум другой информации, необходимой конкретному продукту.

Продукт самостоятельно создаёт локальную сущность пользователя:

```text
EienRei subject → ProductUserId / ReaderId / AuthorId / ...
```

Роли и продуктовые permissions никогда не хранятся в EienRei Auth.

---

## 7. Audience restriction

Каждый access token должен быть предназначен для конкретного resource server или явно ограниченного набора ресурсов.

```json
{
  "iss": "https://me.eienrei.com",
  "sub": "opaque-product-subject",
  "aud": "novelier-api",
  "scope": "novelier.read",
  "exp": 1788192000
}
```

Токен, выданный для Novelier API, не должен приниматься другим сервисом EienRei.

Resource server обязан валидировать как минимум:

- signature;
- issuer;
- audience;
- lifetime;
- допустимые scopes/permissions.

---

## 8. Token lifetime и refresh rotation

Не увеличивать lifetime access token ради многонедельных пользовательских сессий.

Базовая политика:

```text
Access token:          ~5–15 минут
Refresh token:         существенно дольше
Browser/BFF session:   configurable
Remember me:           отдельная политика
```

Точные значения определяются отдельно после threat modeling.

Refresh token используется с **rotation**:

```text
RT1 → RT2 → RT3 → RT4
```

Каждый успешно использованный refresh token заменяется новым.

Если ранее использованный token повторно появляется:

```text
reuse detected
→ revoke refresh-token family
→ mark session suspicious/compromised
→ require reauthentication
→ security event / notification
```

Refresh token family должна быть связана с конкретной пользовательской сессией и client.

---

## 9. Session model

Сессия и token — разные сущности.

```text
User
 └── Session
      ├── SessionId
      ├── CreatedAt
      ├── LastSeenAt
      ├── Device metadata
      ├── IP / coarse location metadata
      ├── UserAgent
      ├── Authentication context
      ├── Risk state
      └── Grants / TokenFamilies
           ├── Novelier
           ├── Product B
           └── Product C
```

Это позволяет независимо реализовать:

- `Logout from this product` — revoke grant/token family конкретного client;
- `Logout this device` — revoke одну Session;
- `Logout all other devices` — revoke все Sessions кроме текущей;
- `Logout everywhere` — revoke все Sessions пользователя;
- `Revoke application access` — удалить consent/grant конкретного client.

---

## 10. Logout и Single Logout

Использовать стандартные OIDC-механизмы, а не собственный протокол.

Предпочтительная комбинация:

- RP-Initiated Logout;
- Back-Channel Logout;
- при необходимости Front-Channel Logout.

Сценарий global logout:

```text
User → "Sign out everywhere"
        ↓
EienRei Auth revokes sessions/grants
        ↓
Back-channel logout notifications
        ├── Novelier
        ├── Product B
        └── Product C
```

Back-channel logout предпочтителен там, где это поддерживается, поскольку не зависит от открытых browser tabs.

---

## 11. Authentication methods

Модель не должна быть построена вокруг пары `Password + optional TOTP`.

Вводится абстракция **AuthenticationMethod**.

Планируемые методы:

```text
Password
Passkey / WebAuthn
TOTP
RecoveryCode
ExternalIdentity
EmailMagicLink        # optional
WindowsIdentity       # enterprise/federation
```

Пользователь может иметь несколько методов одновременно.

---

## 12. Passkeys / passwordless

Passkeys должны стать одной из основных возможностей EienRei Auth.

Поддерживаются:

- добавление passkey к существующему аккаунту;
- несколько passkeys на пользователя;
- passwordless login;
- потенциально создание аккаунта без пароля;
- platform authenticators: Touch ID, Face ID, Windows Hello, Android;
- roaming FIDO2 security keys.

Целевая UX-модель:

```text
[ Continue with passkey ]

──────── or ────────

Email
Password
```

Стратегия: **passkey-first, password-compatible**.

---

## 13. MFA, authentication assurance и Step-Up Authentication

Вместо единственного флага `2FA enabled` используется authentication context.

Пример уровней:

```text
AAL1 — password / trusted external login
AAL2 — passkey или password + TOTP
AAL3 — hardware-backed/high-assurance method при необходимости
```

Токены могут содержать:

```json
{
  "amr": ["pwd", "otp"],
  "acr": "urn:eienrei:aal2"
}
```

Чувствительные операции могут требовать **step-up authentication**:

- изменение email;
- изменение/recovery password;
- добавление/удаление passkey;
- просмотр или регенерация recovery codes;
- удаление аккаунта;
- создание высокопривилегированных credentials;
- изменение security settings.

Продукт также может запросить более высокий assurance level через OIDC authentication request.

---

## 14. External identities / Social login

Термин `SocialLogin` в доменной модели не используется. Используется более общее понятие **ExternalIdentity**.

Первоначально могут поддерживаться:

- Google;
- GitHub;
- Discord;
- Microsoft;
- Apple;
- GitLab.

Структура:

```text
User
 └── ExternalIdentities
      ├── Google
      │    └── ProviderSubject
      ├── GitHub
      │    └── ProviderSubject
      └── Microsoft
           └── ProviderSubject
```

### Account linking

Совпадение email во внешнем provider и существующем EienRei Account **не является достаточным основанием** для автоматического объединения аккаунтов.

Правильный flow:

```text
External login succeeded
        ↓
Matching local email detected
        ↓
Require authentication into existing EienRei account
        ↓
Explicitly link external identity
```

Пользователь должен иметь возможность самостоятельно unlink provider, если после этого остаётся хотя бы один валидный способ входа/восстановления.

---

## 15. Identity Federation / Identity Broker

EienRei Auth должен уметь работать не только как IdP, но и как **Identity Broker**.

```text
Google ────────┐
GitHub ────────┤
Discord ───────┤
Microsoft ─────┤
Generic OIDC ──┤
AD FS ─────────┤
Windows AD ────┤
               ▼
         EienRei Auth
               │
        unified identity
               │
      ┌────────┼────────┐
      ▼        ▼        ▼
   Novelier  App B    App C
```

Продукты всегда доверяют только EienRei Auth и не интегрируются напрямую с внешними providers.

---

## 16. Generic OpenID Connect federation

После первоначальных built-in providers добавить возможность регистрации произвольного внешнего OIDC provider:

```text
Provider type: OpenID Connect
Issuer: https://accounts.example.com
Client ID: ...
Client authentication: ...
```

Metadata загружается через:

```text
/.well-known/openid-configuration
```

Это позволяет интегрировать:

- сторонние IdP;
- корпоративные identity platforms;
- другие self-hosted OIDC servers;
- AD FS / Entra-compatible providers при соответствующей конфигурации.

---

## 17. LDAP / Active Directory / Windows Integrated Authentication

Enterprise authentication рассматривается как часть federation, а не как отдельный special-case login.

Предпочтительный порядок интеграции:

1. Generic OIDC / AD FS — основной federated вариант;
2. Windows Integrated Authentication (Kerberos) для trusted internal environments;
3. LDAP/Active Directory bind — только если нет более подходящего federation-протокола.

Целевая модель:

```text
Domain Windows session
        ↓ Kerberos
EienRei Auth enterprise authentication
        ↓ OIDC
Any EienRei-compatible application
```

Пароль доменного пользователя не должен передаваться продуктам EienRei.

Прямой LDAP bind через публичную login-форму не рассматривается как preferred design.

---

## 18. Security Center

`me.eienrei.com/account/security` становится центральной пользовательской страницей безопасности.

Она показывает:

- доступные authentication methods;
- passkeys;
- состояние TOTP/MFA;
- recovery codes;
- external identities;
- active sessions;
- recent security activity;
- связанные applications;
- действия для emergency recovery/logout.

Пример:

```text
Sign-in methods
────────────────────────────
✓ Passkey — MacBook Touch ID
✓ Password
✓ Authenticator
○ Recovery codes — 7 remaining

Active sessions
────────────────────────────
MacBook Air · Safari · Current
Windows PC · Chrome · 2h ago     [Revoke]
iPhone · Safari · Yesterday      [Revoke]

[Sign out all other sessions]
```

---

## 19. Security event log и notifications

Вводится доменное понятие **SecurityEvent**.

Примеры:

```text
LOGIN_SUCCEEDED
LOGIN_FAILED
PASSWORD_CHANGED
PASSWORD_RECOVERED
PASSKEY_ADDED
PASSKEY_REMOVED
TOTP_ENABLED
TOTP_DISABLED
RECOVERY_CODES_REGENERATED
EXTERNAL_IDENTITY_LINKED
EXTERNAL_IDENTITY_UNLINKED
NEW_SESSION
SESSION_REVOKED
APPLICATION_AUTHORIZED
APPLICATION_REVOKED
REFRESH_TOKEN_REUSE_DETECTED
HIGH_RISK_LOGIN
```

Security Center показывает историю значимых событий.

Для критичных событий возможны уведомления по email и, в будущем, другим каналам.

Пример:

```text
New sign-in detected
MacBook · Safari
Riga, Latvia
31 Aug 2026 18:41

[This was me]
[Secure account]
```

IP и location metadata являются security telemetry и должны храниться ограниченное время согласно выбранной privacy policy.

---

## 20. Risk-based authentication

Поздняя функциональность: адаптивная оценка риска входа.

Сигналы могут включать:

- известное/неизвестное устройство;
- историю успешных входов;
- грубую географию;
- ASN/network reputation;
- скорость перемещения между login events;
- количество failed attempts;
- необычный authentication method;
- признаки refresh-token reuse;
- sensitivity запрашиваемой операции.

Результат:

```text
Low
Medium
High
```

Пример:

```text
Known MacBook + usual location + valid passkey
→ normal authentication

Unknown device + unusual region + repeated failures
→ require step-up authentication
```

Risk engine не должен принимать необратимые решения самостоятельно; его задача — повышать требования к подтверждению личности и создавать security events.

---

## 21. Account Recovery

Recovery — отдельная подсистема, а не только `Forgot password`.

Допустимые recovery channels:

```text
Passkey
Recovery codes
Verified email
External identity
Support/admin procedure     # только если когда-либо понадобится
```

После high-risk recovery можно вводить временный **security hold** на наиболее чувствительные операции:

- смена recovery email;
- удаление аккаунта;
- регенерация recovery codes;
- удаление всех authentication methods;
- создание privileged credentials.

Конкретные сроки и условия определяются threat model.

---

## 22. Consent и Connected Applications

`me.eienrei.com/account/applications` показывает OAuth/OIDC clients, которым пользователь дал доступ.

Пример:

```text
Novelier
Granted: Aug 15
Permissions:
✓ Basic profile
✓ Email

Last used: Today
[Revoke access]
```

Для clients, которым нужен пользовательский consent:

```text
Some App wants to:
✓ Know who you are
✓ Read your public profile
○ Access your email

[Cancel] [Authorize]
```

Отзыв consent должен отзывать связанные grants/refresh-token families.

Внутренние first-party приложения могут иметь отдельную policy по consent.

---

## 23. Developer Portal

В перспективе `me.eienrei.com/developers` предоставляет интерфейс управления OAuth clients.

Возможности:

- создать application;
- получить `client_id`;
- выбрать client type;
- настроить redirect URIs;
- настроить post-logout URIs;
- выбрать allowed grants;
- выбрать scopes/resources;
- настроить client authentication;
- управлять ключами;
- просмотреть basic audit information;
- revoke/rotate credentials.

Пример:

```text
Application: My App
Client ID: xxxxxxxx
Type: Web Application
Redirect URIs:
  https://example.com/callback
Allowed scopes:
  openid
  profile
  email
```

На ранних версиях Developer Portal может быть доступен только администратору EienRei.

---

## 24. Scopes, resources и permissions

Scopes должны отражать не внутренние роли продукта, а разрешённый OAuth-доступ.

Стандартные:

```text
openid
profile
email
offline_access
```

Продуктовые scopes:

```text
novelier.api
novelier.read
novelier.write
```

При необходимости scopes и resource permissions разделяются: authorization server выдаёт утверждённые claims/scopes, а продукт всё равно принимает окончательное решение об авторизации на своей стороне.

---

## 25. Pushed Authorization Requests (PAR)

После стабилизации базовой версии можно включить **Pushed Authorization Requests**.

Flow:

```text
Client
  ↓ POST /connect/par
EienRei Auth
  ↓ request_uri
Client redirects browser to:
/connect/authorize?request_uri=...
```

Это уменьшает количество чувствительных authorization parameters в URL и упрощает дальнейшее усиление security profile.

PAR не является обязательным условием v1, но архитектура должна не мешать его включению.

---

## 26. Sender-constrained tokens, DPoP и mTLS

Для high-security clients допускается дальнейшее усиление:

```text
Standard
→ Authorization Code + PKCE

Secure confidential
→ PKCE + private_key_jwt

High Security
→ sender-constrained token / mTLS / DPoP where supported
```

Эти механизмы не являются v1 requirement и вводятся только после появления реальной threat model, оправдывающей их эксплуатационную сложность.

---

## 27. Machine identities

В доменной модели principal разделяется минимум на:

```text
Human
Service
```

Service principal может представлять:

- Novelier Storyteller;
- background workers;
- scheduled jobs;
- CI/CD integration;
- monitoring/automation clients;
- будущие internal services.

Machine identity не получает пользовательские profile claims и не участвует в user session model.

Для неё действуют отдельные:

- credentials;
- scopes;
- audiences;
- rotation policies;
- audit events.

---

## 28. Account lifecycle и privacy

На `me.eienrei.com/account` должны существовать:

- редактирование профиля;
- смена primary email;
- password/security management;
- connected applications;
- connected external identities;
- active sessions;
- экспорт данных аккаунта;
- удаление аккаунта.

Удаление EienRei identity не означает автоматическое физическое удаление всех продуктовых данных без участия продуктов: должен существовать явный lifecycle/notification contract между Auth и продуктами.

При этом EienRei Auth не должен получать обратно продуктовые данные ради реализации удаления.

---

## 29. Observability и audit

Security audit и operational telemetry являются разными потоками.

### Operational telemetry

Через OpenTelemetry:

- request traces;
- latency;
- error rates;
- database metrics;
- token endpoint metrics;
- external provider latency;
- health checks.

### Security audit

Отдельно сохраняются значимые security events с защитой от случайного изменения/удаления в рамках доступной инфраструктуры.

Логи никогда не должны содержать:

- passwords;
- raw access tokens;
- raw refresh tokens;
- authorization codes;
- TOTP secrets;
- recovery codes;
- private keys/client secrets.

---

## 30. Cryptographic keys

Signing/encryption keys Auth являются критичным infrastructure secret.

Требования:

- не хранить production private keys в репозитории;
- обеспечить rotation;
- публиковать одновременно старый и новый public key в JWKS в период миграции;
- не инвалидировать все активные tokens мгновенно без необходимости;
- иметь backup/recovery procedure;
- в дальнейшем рассмотреть external secret manager/HSM/KMS, если инфраструктура вырастет.

---

## 31. Высокоуровневая архитектура

```text
                         ┌────────────────────────────┐
                         │ External Identity Sources  │
                         │                            │
                         │ Google / GitHub / Discord  │
                         │ Microsoft / Apple          │
                         │ Generic OIDC / AD FS       │
                         │ Windows AD / Kerberos      │
                         └─────────────┬──────────────┘
                                       │ federation
                                       ▼
┌────────────────────────────────────────────────────────────────┐
│                        EienRei Auth                            │
│                     https://me.eienrei.com                    │
│                                                               │
│ OpenIddict Server                Identity Broker              │
│ ├─ OAuth 2.0                     ├─ ExternalIdentity          │
│ ├─ OpenID Connect                ├─ Generic OIDC              │
│ ├─ PKCE                          ├─ AD FS                     │
│ ├─ Refresh Rotation              └─ Windows/AD                │
│ ├─ Logout                                                     │
│ ├─ Device Flow                  ASP.NET Core Identity         │
│ ├─ Client Credentials           ├─ Password                  │
│ └─ PAR (later)                  ├─ Passkeys                  │
│                                 ├─ TOTP                      │
│ Security                        └─ Recovery                  │
│ ├─ Sessions                                                   │
│ ├─ Security Events              Account / Developer UI       │
│ ├─ Step-Up                      ├─ Profile                   │
│ ├─ Risk Engine (later)          ├─ Security                  │
│ └─ Token Reuse Detection        ├─ Sessions                  │
│                                 ├─ Applications              │
│ PostgreSQL                      ├─ Activity                  │
│                                 └─ Developer Portal          │
└───────────────────────┬────────────────────────────────────────┘
                        │ OIDC / OAuth 2.0
            ┌───────────┼────────────┬──────────────┐
            ▼           ▼            ▼              ▼
         Novelier    Product B    Product C     CLI/Services
```

---

## 32. Roadmap

Функциональность вводится постепенно. Security-sensitive возможности не должны добавляться только ради feature count: каждой предшествуют тесты и threat modeling.

### v1 — Core Identity Provider

- .NET 10 + ASP.NET Core;
- OpenIddict Server;
- ASP.NET Core Identity;
- PostgreSQL;
- canonical issuer `https://me.eienrei.com`;
- local account + password;
- Authorization Code + PKCE;
- BFF-oriented web flow;
- audiences/resources;
- short-lived access tokens;
- refresh token rotation;
- session model;
- product logout / session logout / global logout;
- baseline SecurityEvent audit;
- basic account management.

### v1.1 — Modern Authentication

- passkeys/WebAuthn;
- passwordless login;
- TOTP;
- recovery codes;
- AuthenticationMethod model;
- initial step-up authentication.

### v1.2 — External Identities

- Google;
- GitHub;
- Discord;
- Microsoft;
- optional Apple/GitLab;
- secure account linking/unlinking;
- external provider security events.

### v1.3 — Security Center

- active sessions UI;
- revoke session;
- logout all other sessions;
- recent security activity;
- security notifications;
- connected applications;
- revoke application access.

### v1.4 — Identity Federation

- Generic OIDC provider;
- AD FS integration;
- enterprise provider configuration;
- first Identity Broker administration UI.

### v1.5 — Developer Platform

- Developer Portal;
- OAuth client registration UI;
- redirect/scopes/resource management;
- key/credential rotation;
- consent management.

### v2 — Enterprise and Machine Identity

- Windows Integrated Authentication / Kerberos where appropriate;
- optional LDAP/AD fallback;
- Device Authorization Flow;
- machine identities;
- Client Credentials;
- richer audit/administration.

### v2+

- PAR;
- advanced step-up policies;
- risk-based authentication;
- sender-constrained tokens;
- mTLS/DPoP where justified;
- more advanced security analytics.

---

## 33. Вопросы, которые остаются открытыми

Даже после расширения архитектуры требуют отдельного проектирования:

1. **Точные token/session lifetimes.** Значения должны выбираться после threat modeling и UX-тестирования, а не произвольно.
2. **UI technology.** Razor Pages или отдельный Nuxt frontend для `me.eienrei.com`.
3. **Pairwise subject sector policy.** Определить, вычисляется `sub` на client или на группе связанных first-party clients.
4. **Consent policy first-party приложений.** Какие scopes требуют пользовательского consent.
5. **Account deletion orchestration.** Как продукты уведомляются об удалении центральной identity без передачи бизнес-данных в Auth.
6. **Email provider и transactional notifications.** Нужен отдельный infrastructure choice.
7. **Risk data retention/privacy.** Сколько хранить IP/device/location metadata.
8. **Recovery security hold.** Какие операции блокируются и на какой срок после high-risk recovery.
9. **Enterprise federation deployment.** Нужен ли вообще прямой LDAP bind, если AD FS/OIDC/Kerberos закроют реальные сценарии.
10. **Secrets/key management.** Когда текущего Docker secret/environment подхода станет недостаточно и потребуется Vault/KMS-подобный слой.

---

## 34. Главные архитектурные принципы

1. **Identity централизована, product authorization децентрализована.**
2. **Продукты не получают пароли и refresh tokens пользователя без необходимости.**
3. **Browser не является безопасным хранилищем OAuth credentials; web-приложения ориентированы на BFF.**
4. **Access tokens короткоживущие; долгие сессии реализуются через защищённый refresh/session lifecycle.**
5. **Каждый token имеет корректный issuer и audience.**
6. **Продукты получают минимальный набор claims.**
7. **Pairwise subject identifiers уменьшают корреляцию пользователей между продуктами.**
8. **Passkeys — first-class authentication method.**
9. **External login реализуется как identity federation, а не как набор специальных исключений.**
10. **LDAP/AD — часть enterprise federation; предпочтение стандартным OIDC/Kerberos механизмам.**
11. **Session, grant и token — разные сущности с независимым lifecycle.**
12. **Любое чувствительное действие может потребовать step-up authentication.**
13. **Security events являются частью доменной модели, а не только логами приложения.**
14. **Machine identities отделены от human identities.**
15. **Новые advanced security mechanisms внедряются по threat model, а не ради сложности.**
