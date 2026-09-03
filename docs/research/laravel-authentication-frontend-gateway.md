# Laravel authentication behind a Tanned frontend gateway

Research for [Map Laravel's authentication and frontend-gateway constraints](https://github.com/aminevg/tanned/issues/3).

## Research frame

This report was prepared on 2026-09-03 against:

- Laravel 13 documentation at commit [`ccefc99`](https://github.com/laravel/docs/tree/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2), committed 2026-09-02.
- Laravel Framework [`v13.30.1`](https://github.com/laravel/framework/releases/tag/v13.30.1), released 2026-09-01.
- Laravel Fortify [`v1.39.0`](https://github.com/laravel/fortify/releases/tag/v1.39.0), released 2026-08-25.
- Laravel Sanctum [`v4.3.3`](https://github.com/laravel/sanctum/releases/tag/v4.3.3), released 2026-07-21.
- The official React starter kit at commit [`87cce87`](https://github.com/laravel/react-starter-kit/tree/87cce8705d712629ebddd70ccfbb06592ecbaac2), committed 2026-08-31.

Laravel 13 requires PHP 8.3 and receives bug fixes through Q3 2027 and security fixes through 2028-03-17. Laravel 12 remains under security support through 2027-02-24. Laravel 11 reached end of security support on 2026-03-12 ([support table](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/releases.md#L19-L31)).

## Conclusion

A frontend gateway can reuse Laravel's established authentication layer without using starter-kit views or controllers. The Tanned endpoint should run through Laravel's normal `web` middleware group and call the framework's configured guard, password broker, verification contracts, gates / policies, and validator. The gateway must forward cookies in both directions and preserve the original request's CSRF-relevant headers. All Tanned requests should explicitly prefer JSON.

Sanctum is not required for this default topology. Sanctum solves the different case in which a browser calls a Laravel API directly. Fortify is also not a required runtime dependency for the basic flows in this ticket. It remains an optional compatibility integration because official starter kits use Fortify and applications can customize Fortify's login pipeline and password-reset actions.

This yields one required Composer install for an existing Laravel project: the future Tanned Laravel package itself. Its service provider can register a reserved JSON operation endpoint and attach the existing `web` middleware group. It should depend on supported Laravel framework versions, not install Sanctum or Fortify unconditionally.

## Why the `web` session stack is the correct base

Laravel's `web` middleware group already contains cookie encryption, queued-cookie handling, session startup, request-forgery protection, and route binding ([framework source](https://github.com/laravel/framework/blob/718d17db56861e0a49f644217c8853dab1bff8ce/src/Illuminate/Foundation/Configuration/Middleware.php#L485-L492)). Laravel's default browser guard stores authentication in that session. The documented manual login flow uses `Auth::attempt(...)` and regenerates the session after success; logout removes the guard state, invalidates the session, and regenerates the CSRF token ([login](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/authentication.md#L241-L280), [logout](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/authentication.md#L489-L505)).

The frontend gateway therefore does not need a second auth token or a synchronized frontend session. It needs to act as a lossless reverse proxy for Laravel's cookies:

1. Forward the browser's complete `Cookie` header to Laravel on every operation, including SSR loader calls.
2. Relay every Laravel `Set-Cookie` header separately to the browser. Do not combine headers. This includes the encrypted session cookie, the readable `XSRF-TOKEN` cookie, remember-me cookies, cookie expiry on logout, and any cookies added by the backend.
3. Preserve `Path`, `Expires` / `Max-Age`, `HttpOnly`, `Secure`, and `SameSite`. A host-only cookie returned through the gateway naturally belongs to the frontend host. If an existing backend sets an explicit, incompatible `Domain`, Tanned must either fail configuration validation or apply an explicit domain-rewrite policy; silently broadening a cookie's domain is unsafe.
4. During SSR, propagate backend `Set-Cookie` values onto the outgoing frontend response before its headers are committed. A server-to-server fetch does not update the browser's cookie jar by itself.
5. If one frontend request makes multiple backend calls and an earlier call rotates a cookie, later calls in that request must use the rotated cookie. Login, logout, and CSRF bootstrap depend on this.

Laravel's session middleware constructs the session cookie from the configured domain, secure, HTTP-only, SameSite, and partitioned settings ([source](https://github.com/laravel/framework/blob/718d17db56861e0a49f644217c8853dab1bff8ce/src/Illuminate/Session/Middleware/StartSession.php#L220-L237)); the CSRF middleware similarly issues `XSRF-TOKEN` ([source](https://github.com/laravel/framework/blob/718d17db56861e0a49f644217c8853dab1bff8ce/src/Illuminate/Foundation/Http/Middleware/PreventRequestForgery.php#L231-L254)). Tanned should not replace either mechanism.

## CSRF through the gateway

The gateway is not a reason to disable CSRF. A hostile site can still cause a user's browser to submit a credentialed request to the public frontend origin. Laravel must receive enough information to reject it.

Laravel 13 first accepts secure requests whose browser-provided `Sec-Fetch-Site` value is `same-origin`; otherwise it falls back to matching the session token with `_token`, `X-CSRF-TOKEN`, or encrypted `X-XSRF-TOKEN`. Token mismatch becomes HTTP 419, while origin-only mode rejects invalid origins with 403 ([documentation](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/csrf.md#L37-L102), [implementation](https://github.com/laravel/framework/blob/718d17db56861e0a49f644217c8853dab1bff8ce/src/Illuminate/Foundation/Http/Middleware/PreventRequestForgery.php#L95-L180)). Laravel 12 uses the token-oriented predecessor, so Tanned should support the token path even when running on Laravel 13.

Required gateway behavior:

- Forward the browser-provided `Sec-Fetch-Site`, `Origin`, and `Referer` values without upgrading them. In particular, never synthesize `Sec-Fetch-Site: same-origin` for a server-to-server call.
- Bootstrap a session / CSRF cookie before the first state-changing operation, then URL-decode `XSRF-TOKEN` and send it to Laravel as `X-XSRF-TOKEN`. The frontend primitive can hide this sequence. Laravel documents the same mechanism for Sanctum SPAs ([Sanctum CSRF flow](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/sanctum.md#L326-L348)).
- A server action that receives browser cookies may derive `X-XSRF-TOKEN` from the incoming `XSRF-TOKEN` cookie. A background server process with no browser request must not impersonate a browser operation; it needs a separate trusted server-auth mechanism if such calls are ever supported.
- Do not exempt the reserved Tanned endpoint from Laravel's request-forgery middleware.

## Authentication flows

### Login

The basic operation can use the configured stateful guard and the application's existing user provider. On success it must regenerate the session. It should also preserve Laravel's remember-me behavior and the configured credential identifier.

Official starter kits use Fortify ([documentation](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/starter-kits.md#L352-L382)). Fortify adds more than `Auth::attempt`: its pipeline can canonicalize usernames, enforce rate limits, invoke an application-supplied authentication callback, branch to two-factor authentication, perform the guard attempt, and regenerate the session ([controller](https://github.com/laravel/fortify/blob/b1fc50707bbe007fd92165d8b7d460ab549b355a/src/Http/Controllers/AuthenticatedSessionController.php#L51-L91), [attempt action](https://github.com/laravel/fortify/blob/b1fc50707bbe007fd92165d8b7d460ab549b355a/src/Actions/AttemptToAuthenticate.php#L49-L102), [session preparation](https://github.com/laravel/fortify/blob/b1fc50707bbe007fd92165d8b7d460ab549b355a/src/Actions/PrepareAuthenticatedSession.php#L34-L43)).

Consequently, core login is transparent for applications that rely on the configured Laravel guard/provider. It is not automatically behavior-identical for an application that customized Fortify's pipeline, `authenticateUsing` callback, rate limiters, two-factor flow, or passkeys. A Fortify compatibility layer must deliberately invoke or reproduce those registered extension points; copying Fortify's current internal pipeline into Tanned would be a brittle dependency.

### Logout

Call the configured stateful guard's `logout`, then invalidate the session and regenerate its CSRF token. Relay all resulting cookies. This matches both Laravel's documented flow and Fortify's implementation ([Fortify source](https://github.com/laravel/fortify/blob/b1fc50707bbe007fd92165d8b7d460ab549b355a/src/Http/Controllers/AuthenticatedSessionController.php#L100-L109)). Navigation after logout belongs to the frontend; the backend operation should return success rather than an HTTP redirect.

### Password reset

Use the password broker selected by the existing `config/auth.php`. The broker already uses the configured user provider to locate the user and owns token creation, throttling, expiry, and validation ([documentation](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/passwords.md#L95-L132)). The reset operation must validate `token`, the configured identity field, `password`, and `password_confirmation`, update the password securely, rotate the remember token, persist the user, and dispatch `PasswordReset`, matching Laravel's documented sequence ([reset flow](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/passwords.md#L143-L197)).

The standard reset notification assumes a backend named route named `password.reset`. Tanned instead needs a fixed public frontend origin and a frontend route. Laravel officially exposes `ResetPassword::createUrlUsing(...)` for this purpose ([documentation](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/passwords.md#L216-L237), [source](https://github.com/laravel/framework/blob/718d17db56861e0a49f644217c8853dab1bff8ce/src/Illuminate/Auth/Notifications/ResetPassword.php#L85-L112)). The Tanned package can register this callback from configuration, but must not silently overwrite an application's existing callback or custom `sendPasswordResetNotification` implementation.

When Fortify is installed, its `ResetsUserPasswords` contract is an important compatibility point: official starter kits bind an application-owned action there. Tanned can optionally resolve and use that contract. Without Fortify, a generic default requires the user model to satisfy Laravel's normal reset contracts and persistence expectations.

### Email verification

Reuse Laravel's `MustVerifyEmail` contract, `sendEmailVerificationNotification`, `markEmailAsVerified`, and `Verified` event. The standard handler also checks the authenticated user's id and email hash and protects its URL with a temporary signature ([documentation](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/verification.md#L23-L116)).

Laravel's default notification generates a temporary signed URL for the backend route named `verification.verify`. The framework also exposes `VerifyEmail::createUrlUsing(...)` ([source](https://github.com/laravel/framework/blob/718d17db56861e0a49f644217c8853dab1bff8ce/src/Illuminate/Auth/Notifications/VerifyEmail.php#L77-L103)). Tanned must generate a URL at the public frontend origin and preserve the exact canonical scheme, host, path, and query used to validate the signature. An absolute Laravel signature includes the URL; forwarding the link to a different backend path and then running the normal `signed` middleware will fail. Viable designs are:

- proxy the frontend verification path to a same-path backend operation while forwarding the trusted public host and scheme; or
- define and verify a Tanned-specific signed payload at the operation boundary.

Choosing between these designs belongs to the later operation-interface work. It cannot be hidden by ordinary fetch forwarding alone.

## Authorization, validation, and error semantics

Every Tanned backend request should send `Accept: application/json`, with `Content-Type: application/json` when it has a JSON body. This makes Laravel's normal exception handler return API-shaped responses instead of backend redirects:

- unauthenticated: `401` with `{"message":"Unauthenticated."}` ([handler](https://github.com/laravel/framework/blob/718d17db56861e0a49f644217c8853dab1bff8ce/src/Illuminate/Foundation/Exceptions/Handler.php#L854-L866));
- validation failure: `422` with `message` and field-keyed `errors`, including dotted nested keys ([documentation](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/validation.md#L281-L308), [handler](https://github.com/laravel/framework/blob/718d17db56861e0a49f644217c8853dab1bff8ce/src/Illuminate/Foundation/Exceptions/Handler.php#L876-L916));
- gate / policy denial: `403` by default, but applications may intentionally return another status such as `404` ([documentation](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/authorization.md#L125-L225));
- CSRF token mismatch: `419`; Laravel 13 origin-only mismatch: `403`;
- expired session: generally `401` on an authenticated read or `419` on a state-changing request whose CSRF token no longer matches. Laravel's Sanctum docs explicitly warn clients to handle both ([documentation](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/sanctum.md#L342-L350)).

Tanned should expose these as typed frontend outcomes, not infer backend navigation. A `401` or `419` can trigger a frontend-defined login route. `403` and intentional `404` must remain distinguishable. Validation must preserve the full error arrays and dotted keys rather than flattening them into one message.

`Accept: application/json` does not rewrite an explicit redirect response returned by application code. Tanned operations should therefore return data or typed outcomes and treat arbitrary 3xx responses as unsupported unless a future primitive deliberately models them. Laravel's session `intended` URL and flashed errors are backend-navigation concepts and cannot be transparently preserved while the frontend owns routing.

## Network and deployment constraints

### Recommended default: same origin to the browser

The browser should communicate only with the TanStack Start origin. The frontend gateway calls Laravel over a private or otherwise protected upstream. In this arrangement:

- browser requests are same-origin;
- browser CORS configuration is unnecessary;
- Laravel and the frontend may use unrelated internal deployment hostnames;
- Laravel session cookies remain opaque and are relayed through the frontend origin;
- SSR and hydrated requests can use the same Tanned frontend API.

This is a gateway deployment, not a direct cross-origin SPA deployment. The gateway itself becomes security-sensitive infrastructure: it must restrict the upstream, remove hop-by-hop headers, reject ambiguous request framing, set size / timeout limits, preserve multiple cookie headers, and avoid exposing an unrestricted open proxy.

### Direct browser-to-Laravel is a different mode

If the browser bypasses the gateway and calls Laravel directly, Sanctum's documented SPA mode applies. It uses Laravel's session cookies but requires the SPA and API to share the same top-level domain, requires `Accept: application/json` plus `Origin` or `Referer`, classifies first-party domains from `sanctum.stateful`, and requires credentialed CORS and suitable cookie-domain configuration ([documentation](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/sanctum.md#L266-L324)). Its stateful middleware chooses the session stack by matching `Referer` / `Origin` and forces HTTP-only, `SameSite=Lax` sessions ([source](https://github.com/laravel/sanctum/blob/fee27a573d1a013af3721d86153a65e0b11927e6/src/Http/Middleware/EnsureFrontendRequestsAreStateful.php#L19-L96)).

Therefore, unrelated top-level frontend and backend domains are not a transparent Sanctum SPA configuration. `SameSite=None`, partitioned cookies, browser privacy policies, and third-party-cookie blocking make that a separate compatibility project. Tanned should not promise it in the first release.

### Trusted proxies, hosts, and generated links

The gateway should send sanitized `Forwarded` or `X-Forwarded-*` values that describe the public frontend URL. Laravel must trust only the actual gateway / load-balancer addresses and the precise header set used; otherwise client-controlled forwarding headers can corrupt scheme, host, URL generation, IP-based rate limits, and audit data. Laravel documents `trustProxies` and its header mask in `bootstrap/app.php` ([documentation](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/requests.md#L818-L859)).

Laravel uses the request `Host` when generating absolute URLs and accepts every host by default. Deployments should reject unknown hosts at the web server or configure `trustHosts`; Laravel calls this particularly important for password reset ([trusted-host docs](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/requests.md#L861-L889), [password-reset warning](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/passwords.md#L71-L78)). A fixed, validated Tanned public frontend origin is safer for queued notifications than deriving links from an incoming host.

## Minimum Tanned Laravel package and configuration surface

### Composer/package behavior

`composer require tanned/laravel` should be the only new install required in a normal Laravel project. The package should:

- support maintained Laravel majors, initially `^12.0 || ^13.0`; this implies PHP 8.2 for Laravel 12 and PHP 8.3 for Laravel 13;
- auto-discover one service provider;
- register a reserved JSON operation endpoint through the existing `web` middleware group;
- use the configured stateful guard and password broker instead of introducing its own user table, guard, token, or session store;
- normalize known Laravel exceptions while retaining their status and safe payload;
- register frontend URL callbacks for the default password-reset and email-verification notifications only when the application has not already customized them;
- provide optional Fortify integration when Fortify is already installed, without requiring it;
- not require Sanctum for gateway mode.

The package should avoid hard-coding the Laravel 12 `ValidateCsrfToken` or Laravel 13 `PreventRequestForgery` class. Attaching the application's `web` group preserves the version-appropriate middleware and any application customization.

### Required configuration

The irreducible configuration is small but not zero:

- canonical public frontend origin, including scheme and non-default port;
- the reserved backend operation path / prefix;
- guard override only when the desired guard is not Laravel's configured default;
- password-broker override only when the desired broker is not Laravel's configured default;
- trusted gateway addresses and forwarded-header policy, preferably in deployment-specific Laravel / web-server configuration;
- an explicit cookie-domain rewrite only for existing projects whose `SESSION_DOMAIN` cannot be used by the frontend host;
- frontend route paths for password reset and email verification, unless Tanned establishes conventions or obtains them from its generated route manifest.

Everything else should be inferred from Laravel configuration. Tanned should ship a diagnostic command that checks the endpoint's middleware, stateful guard, session persistence, cookie domain / flags, public origin, trusted hosts / proxies, password broker, verification model contract, and conflicting notification URL callbacks before production.

## Existing-project compatibility boundaries

The initial compatibility claim can be strong for applications using Laravel's built-in session guard and standard framework contracts, including applications originally created from a starter kit. The claim must be narrower than “all starter-kit auth behavior is automatically retained.”

Not automatically transparent:

- custom or non-stateful guards that do not implement the normal session behavior;
- Fortify custom authentication pipelines, `authenticateUsing`, custom response bindings, two-factor authentication, passkeys, or application-specific rate limits unless the optional Fortify integration explicitly supports them;
- the official WorkOS AuthKit starter-kit variants, social login / OAuth callbacks, SSO, or other external identity providers;
- application overrides of exception rendering that change `401`, `403`, `419`, or `422` response shapes;
- custom session-cookie encryption exclusions, incompatible explicit cookie domains, or infrastructure that cannot forward multiple `Set-Cookie` headers;
- absolute signed URLs whose public host / scheme / path changes between generation and verification;
- custom reset / verification notifications that bypass Laravel's URL callback hooks;
- arbitrary backend redirects, flashed “intended” destinations, and view-specific error bags;
- SSR or prerender caching of authenticated responses. Personalized loader results must remain request-scoped and non-cacheable.

Fortify's `views: false` disables its GET view routes but leaves mutation routes registered ([documentation](https://github.com/laravel/docs/blob/ccefc99cd9f671ef3c60b2e1b3ee3188df8fadf2/fortify.md#L114-L126), [routes](https://github.com/laravel/fortify/blob/b1fc50707bbe007fd92165d8b7d460ab549b355a/routes/routes.php#L27-L100)). If an existing starter-kit project should expose only Tanned operations, it must also disable Fortify route registration with `Fortify::ignoreRoutes()` and let Tanned provide the corresponding operations. That setting is distinct from disabling Fortify views.

The current official React starter-kit template requires Fortify and contains selectable feature blocks for password reset, email verification, two-factor authentication, and passkeys ([Composer dependency](https://github.com/laravel/react-starter-kit/blob/87cce8705d712629ebddd70ccfbb06592ecbaac2/composer.json#L8-L20), [feature configuration](https://github.com/laravel/react-starter-kit/blob/87cce8705d712629ebddd70ccfbb06592ecbaac2/config/fortify.php#L166-L188)). The installer may remove optional blocks, but 2FA and passkeys are visible compatibility exclusions for an initial basic-auth release, not hypothetical future edge cases.

## Resulting decision constraints

The later Tanned operation and gateway design should treat these as requirements:

1. Gateway mode uses Laravel `web` sessions and request-forgery protection; Sanctum is optional and belongs to direct-browser mode.
2. Cookie and header propagation is part of Tanned's correctness contract, including SSR response mutation and intra-request cookie rotation.
3. Backend operations produce JSON / typed outcomes. The frontend alone chooses routes and redirect destinations.
4. The package uses Laravel's configured guard, provider, password broker, verification contract, gates / policies, validators, and events.
5. The package owns or coordinates frontend-origin reset and verification URLs; absolute signed verification URLs require an explicit design.
6. Fortify compatibility is capability-based. Basic guard/provider compatibility can be automatic; Fortify pipeline, MFA, passkey, and custom-action compatibility must be declared and tested individually.
7. Initial support should target Laravel 12 and 13, not the unsupported Laravel 11 line.
