# TanStack Start integration surface for Tanned

Research date: 2026-09-03

## Scope and version

This report answers [Map TanStack Start's integration surface for Tanned](https://github.com/aminevg/tanned/issues/9) using only TanStack's current official documentation and source.

The source snapshot is TanStack Router commit [`37877da166fe4ce055c7b85e138b6681ebd7e8b4`](https://github.com/TanStack/router/commit/37877da166fe4ce055c7b85e138b6681ebd7e8b4), committed 2026-08-31. At that commit:

- `@tanstack/react-start` is `1.168.49`;
- `@tanstack/react-router` is `1.170.32`;
- `@tanstack/start-plugin-core` is `1.171.39`;
- `@tanstack/router-generator` is `1.167.33`;
- React Start requires Vite 7 or newer or Rsbuild 2 or newer and supports React 18 or 19.

React Start `1.168.49` was also the latest Start version in the repository's published [2026-08-22 release](https://github.com/TanStack/router/releases/tag/release-2026-08-22-2258). The official overview calls Start a release candidate whose API is considered stable, while several relevant features remain explicitly experimental: custom server-function IDs, static server functions, import protection, deferred hydration, and React Server Components ([overview](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/start/framework/react/overview.md), [package metadata](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/packages/react-start/package.json)).

## Answer

Tanned's paired TSX/PHP route modules and frontend gateway are feasible on current TanStack Start without changing TanStack Router and without defining a public Tanned wire protocol.

The smallest robust design is:

1. TanStack's TSX route remains the canonical web route.
2. A Tanned build plugin discovers the PHP sidecar with the corresponding relative path, asks the Laravel integration to extract its operation names and PHPStan-derived input/output types, and emits TypeScript bindings plus a backend operation manifest.
3. Generated bindings call one or more ordinary, statically declared Start server functions. During SSR those functions execute in the Start server process; after hydration the same calls become same-origin RPC requests to the frontend gateway.
4. The Start-side handler forwards the operation identifier and input to the backend. The backend authenticates, authorizes, validates, and executes the operation. Tanned translates the result into typed values or frontend-understood failures.
5. TanStack Router continues to own URL matching, loaders, navigation, pending/error boundaries, invalidation, redirects, SSR, hydration, and prerendering.

This gives Tanned the desired frontend ownership while reusing Start's public request and rendering model. Tanned must supply PHP discovery, type extraction, generated bindings, backend dispatch, session/header forwarding, and error normalization.

Tanned should not initially hook Start's server-function compiler or Router's generator internals. A generated wrapper around a small, hand-written `createServerFn` dispatcher is less coupled to Start's compiler than generating or transforming every server function. It also avoids making PHP files part of the TanStack route tree.

## Capability map

| Area | What current TanStack provides | What Tanned should do | Important constraint |
| --- | --- | --- | --- |
| File routes | Files under `src/routes` generate a typed `routeTree.gen.ts`; nested, pathless, index, dynamic, splat, and virtual routes are supported. | Pair PHP sidecars to TSX routes by relative path and generate bindings outside the route directory or through a Tanned virtual module. | Router's physical scanner accepts only `.tsx`, `.ts`, `.jsx`, `.js`, and `.vue`, so `.php` is ignored. This is useful, but TanStack does not discover or watch PHP sidecars for Tanned. |
| Loaders | Route `beforeLoad` runs serially; matched loaders run in parallel. Loader data is typed, cached, preloaded, invalidated, and automatically dehydrated/hydrated. | Make generated `load` operations callable from normal route loaders. Preserve Router's cache keys and invalidation instead of building a second route-data lifecycle. | Loaders are isomorphic: server on initial SSR, browser on later navigation. They must call a server function or browser-safe API, never PHP/backend-only code directly. |
| Server functions | Typed same-origin RPC callable from loaders, components, event handlers, and other server functions; GET/POST, validators, serializable inputs/results, `FormData` for POST, raw `Response`, errors, redirects, not-found, middleware, request access, and streaming. | Reuse server functions as the frontend-gateway transport for generated Tanned operations. Prefer separate load/action dispatchers if method or cache semantics differ. | The API takes a single `data` input; values must remain serializable. Static imports are supported and dynamic imports are warned against. Server-function IDs and compiler behavior are build-time concerns. |
| Server routes | Raw HTTP handlers colocated with TS routes, with all HTTP methods, params, middleware, request context, and `Response` output. | Reserve these for public callbacks, webhooks, downloads, or a deliberately raw Tanned endpoint. They are not necessary for ordinary internal operations. | A server route is tied to the frontend URL path, and duplicate handler paths error. Using them for every PHP sidecar would conflate frontend URLs with backend operation identifiers. |
| Middleware and context | Global request middleware covers SSR, server routes, and server functions. Function middleware has client/server phases, validation, context, and typed context transfer. The server entry accepts typed per-request context. Start supplies same-origin CSRF middleware for server functions. | Install Tanned's forwarding/error middleware through public Start APIs, and derive trusted session data from the incoming request or backend response. | Client-sent context is untrusted and is not sent unless explicitly requested. Defining `src/start.ts` disables the automatic CSRF installation, so Tanned's setup must preserve or explicitly install it. |
| SSR and hydration | Start runs matched `beforeLoad`/loaders, renders the document, streams by default, and hydrates loader data. Per-route SSR can be full, data-only, or disabled. | Let a Tanned loader call the same generated operation on SSR and navigation. Keep backend access behind the Start server function. | A route loader itself also ships to and executes in the browser. Only the server-function handler may contain backend credentials or trusted forwarding logic. |
| Redirects | Router redirects can be thrown from `beforeLoad`, loaders, and server functions and are handled across SSR and client navigation. | Convert only explicit Tanned outcomes into Router redirects, or let route code choose destinations. Backend HTTP `Location` responses should not automatically control frontend navigation. | Tanned's product model says the frontend owns redirects. The error/outcome contract must distinguish unauthenticated, unauthorized, validation, not-found, and ordinary failures. |
| Streaming | Start streams HTML and deferred loader promises; server functions can return typed `ReadableStream`s or async generators. | Initially support ordinary serializable operation results. Add backend-to-frontend streaming only after proving backpressure, cancellation, and error translation through PHP and the gateway. | Every production adapter/proxy must preserve streaming rather than buffer the response. An upstream backend stream is not automatically a typed Start stream. |
| Prerendering | Build-time prerendering supports explicit pages, automatic static-route discovery, link crawling, filters, concurrency, retry, redirects, and custom output paths. Experimental static server functions cache build-time results as JSON assets. | A public deterministic Tanned load can call the backend while a page is prerendered, provided the backend is reachable during the build. Mark prerender eligibility explicitly. | Dynamic routes need explicit paths or discoverable links. Session-specific/private operations must never be turned into shared build artifacts. Static server functions are experimental. |
| Build integration | `tanstackStart()` composes Vite/Rsbuild plugins and exposes router directory/output options, server-function base/ID options, prerendering, entries, and dev middleware settings. | Ship a normal Tanned Vite plugin first: scan/watch PHP, invoke the backend extractor, emit deterministic generated TypeScript and manifests, and fail builds on stale/incompatible types. | There is no documented Start extension API specifically for foreign-language sidecars. Plugin ordering and multi-environment builds must be tested. |
| Production | The server entry is a standard `fetch(Request) -> Response` handler. Official docs cover Cloudflare Workers, Netlify, Railway, Nitro, Vercel, Node/Docker, Bun, and Appwrite. | Keep the Tanned gateway compatible with Web `Request`/`Response` and standard `fetch`. Make backend location and trusted proxy policy deployment configuration. | Nitro's Vite integration is documented as under active development. Edge deployments cannot assume a local PHP process or Node APIs. |

## Detailed findings

### 1. File-route generation and paired PHP sidecars

Start uses TanStack Router's file routing and automatically creates `routeTree.gen.ts` during development and builds. The generated tree is the source of route type inference. Route files export `Route = createFileRoute(...)`; the generator may update the literal route path in source ([Start routing](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/start/framework/react/guide/routing.md), [Router file routing](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/router/routing/file-based-routing.md)).

The physical scanner only considers JavaScript, TypeScript, JSX, TSX, and Vue extensions. A colocated `index.php` beside `index.tsx` therefore does not become a route or produce a duplicate path ([scanner source](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/packages/router-generator/src/filesystem/physical/getRouteNodes.ts)). That is the correct default for Tanned.

Router does define generator-plugin callbacks for initialization, post-transform work, and route-tree changes ([generator plugin type](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/packages/router-generator/src/plugin/types.ts)). However, Start's integration currently constructs its own generator-plugin list after spreading user router configuration, replacing any supplied `router.plugins` value ([Start Router Vite integration](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/packages/start-plugin-core/src/vite/start-router-plugin/plugin.ts)). Tanned should not depend on this source-level seam.

Use a separate build plugin and generated output that Router will not mistake for routes. Good initial choices are a generated directory outside `src/routes` or filenames using Router's ignored `-` prefix. A virtual module can improve ergonomics later, after multi-environment HMR is proven.

### 2. Loaders are the integration point for route data

Router's lifecycle validates params/search, runs `beforeLoad` top-down, then runs matched loaders in parallel. Loaders receive abort state, params, typed search-derived dependencies, context, and route metadata. Router supplies stale-while-revalidate caching, preloading, pending/error UI, and invalidation ([data loading](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/router/guide/data-loading.md)). Resolved loader data is automatically dehydrated on SSR and rehydrated in the browser ([SSR](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/router/guide/ssr.md)).

The important constraint is that loaders are isomorphic. They execute on the Start server for an initial SSR request and in the browser for later navigation ([execution model](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/start/framework/react/guide/execution-model.md)). A generated Tanned operation called by a loader must therefore be an isomorphic proxy. `createServerFn` already provides that behavior.

Tanned should avoid an implicit second cache. Generated calls should fit inside ordinary loaders, with route authors retaining `loaderDeps`, `staleTime`, preload, and invalidation control. A future higher-level convenience API may generate loader configuration, but it is not required for the first integration.

### 3. Server functions are the best frontend-gateway primitive

Server functions are private, same-origin RPC endpoints intended for calls within a Start frontend. The compiler removes the implementation from client bundles and replaces calls with RPC stubs. They support GET and POST, typed validation, serializable inputs and outputs, `FormData` for POST, raw `Response`s, request/response utilities, middleware, redirects, not-found, cancellation, and typed streaming ([server functions](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/start/framework/react/guide/server-functions.md)).

Start uses different RPC implementations by environment. On the browser, it fetches the server-function URL; on SSR, it resolves and invokes the server function in-process ([browser RPC source](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/packages/start-client-core/src/client-rpc/createClientRpc.ts), [SSR RPC source](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/packages/start-server-core/src/createSsrRpc.ts)). This is exactly the frontend-gateway topology Tanned needs: the route calls one API in both environments, and only the Start handler calls the backend.

The first prototype should compare two shapes:

- one statically declared dispatcher for loads and one for actions, with generated typed wrappers passing `{ operation, data }`; or
- one generated static server function per PHP operation.

The first shape is less coupled to Start's compiler. The second permits per-operation middleware, method selection, and function IDs, but requires materialized source that the compiler can discover. Official docs recommend static imports and warn that dynamic server-function imports can cause bundler problems. Custom server-function ID generation is experimental.

### 4. Server routes are useful but not the default transport

Server routes provide raw HTTP handlers inside a route's `server` property. They share Router's file naming and path params, can be placed in the same TSX file as UI, can use route/handler middleware, and return Web `Response`s ([server routes](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/start/framework/react/guide/server-routes.md)).

They are intended for externally callable endpoints; server functions are recommended for calls internal to a Start frontend. Mapping every PHP sidecar onto a server route would also bind backend operation URLs to frontend page URLs and create conflicts where several route files resolve to the same handler path. Tanned should use server routes only when the operation intentionally exposes raw HTTP semantics, such as a webhook, callback, file response, or public API.

### 5. Middleware, request context, sessions, and CSRF

Request middleware wraps SSR, server routes, and server functions. Function middleware can run on both client and server, validate data, and exchange explicitly serialized context. Global middleware is registered with `createStart` in `src/start.ts`. A typed context passed to the server entry is available throughout request middleware, routes, server functions, and the router ([middleware](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/start/framework/react/guide/middleware.md), [server entry](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/start/framework/react/guide/server-entry-point.md)).

This is sufficient for a Tanned forwarding layer to read the incoming browser request, attach correlation data, call the backend, and normalize its result. The backend remains the authority for the session and authorization. Values sent from the browser through Start context are explicitly untrusted and must not be treated as identity.

Start provides same-origin CSRF checks for server functions. The middleware is automatic only while the frontend has no custom `src/start.ts`; once Tanned or the user defines that file, it must explicitly include `createCsrfMiddleware`. This protects the browser-to-Start hop. Protection of the Start-to-backend hop, cookie forwarding, CSRF token forwarding, `Set-Cookie` propagation, and trusted-proxy headers remain Tanned/backend-integration responsibilities.

### 6. SSR, hydration, and redirects

Full SSR is the default: `beforeLoad` and loaders execute on the server, loader data is sent to the browser, components render to HTML, and the browser hydrates. A route may select full SSR, data-only SSR, or client-only behavior, with children allowed to become only more restrictive than parents ([selective SSR](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/start/framework/react/guide/selective-ssr.md)).

Router `redirect()` values may be thrown from route lifecycles and server functions, and Start carries them across the server/client boundary ([redirect API](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/router/api/router/redirectFunction.md)). Tanned can therefore offer seamless frontend redirects without accepting backend-controlled URL navigation.

The backend operation contract should not expose a raw backend redirect as the default. Tanned should request API-style responses and translate known conditions into typed outcomes. Route code or a configurable frontend policy can then throw a Router redirect to a frontend route. This preserves the settled rule that the frontend owns routes and redirects.

### 7. Streaming

Start's default stream handler incrementally streams SSR HTML. Deferred loader promises are serialized into the document and resolved through suspense as their values or errors arrive ([Router SSR](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/router/guide/ssr.md), [deferred data](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/router/guide/deferred-data-loading.md)). Server functions separately support typed `ReadableStream` results and async generators ([server-function streaming](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/start/framework/react/guide/streaming-data-from-server-functions.md)).

Tanned can reuse HTML and deferred-data streaming immediately when PHP operations return ordinary values. True backend stream pass-through requires a separate design: the PHP integration must expose a stream, the Start gateway must preserve backpressure and cancellation, the serializer must support the chunk type, and the hosting adapter must not buffer. It should not block the first release unless a target use case requires it.

### 8. Prerendering and a backend at build time

Start can prerender configured pages, automatically discover static routes, crawl links, follow a bounded number of redirects, retry failures, and choose output paths. Dynamic routes are not automatically known unless explicitly configured or found through crawling ([static prerendering](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/start/framework/react/guide/static-prerendering.md)).

A Tanned loader can call the backend during prerender because prerendering executes the Start server build. This is not “prerendering the backend”; it is executing a backend operation at frontend build time and embedding the result. It works only if the backend is reachable and the operation is deterministic and safe without a user session.

Start's experimental static server functions additionally cache build-time server-function results into static JSON and replace later browser calls with asset fetches ([static server functions](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/start/framework/react/guide/static-server-functions.md)). Tanned should not adopt that experimental API as a first-release foundation. It may later map an explicit `static`/public operation capability to it. Authenticated or tenant-specific results must never enter shared prerender artifacts.

### 9. Build-plugin seams

The `tanstackStart` plugin accepts source/entry paths, Router generator configuration, client/server base paths, server-function options, page/prerender options, SPA mode, import protection, and a Vite dev-middleware switch ([configuration schema](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/packages/start-plugin-core/src/schema.ts), [Vite-specific schema](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/packages/start-plugin-core/src/vite/schema.ts)). Its implementation is itself a list of Vite plugins and official hosting examples compose it with other plugins ([Vite implementation](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/packages/start-plugin-core/src/vite/plugin.ts)).

That makes an adjacent Tanned plugin practical. The plugin should own these steps:

- discover PHP sidecars and associate them with TSX routes;
- run the backend-specific extractor;
- generate deterministic TypeScript declarations/wrappers and a backend manifest;
- make PHP edits trigger regeneration during development;
- ensure generation finishes before the relevant client and server modules compile;
- verify generated output is current in production builds.

Start now builds multiple Vite environments and also supports Rsbuild. HMR, plugin order, virtual-module invalidation, and one-time generation across environments are prototype questions. Starting with materialized generated files is safer than depending on Start compiler transforms or internal virtual modules. Vite can be the first-release build target; Rsbuild compatibility can remain an extension point unless the product scope requires it.

### 10. Production adapters

Start's server entry uses the WinterCG-style Web API shape `fetch(Request) -> Response`. The same entry handles SSR, server functions, and server routes. Official deployment guidance covers Cloudflare Workers, Netlify, Railway, Nitro, Vercel, Node/Docker, Bun, and Appwrite ([hosting](https://github.com/TanStack/router/blob/37877da166fe4ce055c7b85e138b6681ebd7e8b4/docs/start/framework/react/guide/hosting.md)).

Tanned should stay inside this portable request model. The gateway should use standard `fetch` to call a separately addressable backend and should not assume PHP is colocated at runtime. A Node deployment may optimize a local network path, but edge deployments require HTTPS or a platform service binding. Nitro's Vite integration is currently documented as under active development, so the first production proof should use one conservative Node deployment and one non-Node fetch runtime if portability is a release claim.

## What Tanned can reuse unchanged

- TanStack's TSX route conventions and generated typed route tree.
- Router loaders, search/param typing, preloading, cache behavior, invalidation, pending states, and error boundaries.
- Start server functions for same-origin browser RPC and direct SSR invocation.
- Start request/function middleware and typed request context.
- Router redirects and not-found handling on both SSR and client navigation.
- Automatic loader dehydration/hydration and streaming SSR.
- Start prerender page discovery and output generation.
- The standard fetch server entry and existing hosting integrations.

## What Tanned must add

- A precise PHP sidecar discovery and export convention.
- Laravel/PHPStan extraction of operation names, input types, output types, and operation metadata.
- Generated TypeScript bindings and a backend dispatch manifest.
- Development regeneration and diagnostics for PHP changes.
- A Start-side dispatcher that forwards cookies/headers safely and calls the backend.
- Backend-side secure dispatch that exposes only declared operations.
- A normalized result/error model for validation, authentication, authorization, not-found, conflicts, and unexpected errors.
- Response-header and `Set-Cookie` propagation rules.
- Prerender eligibility metadata and safeguards against embedding private results.
- Deployment configuration for backend location, timeouts, trust boundaries, and observability.

## Version-sensitive risks

1. Start is still labeled RC. The high-level APIs are presented as stable, but build internals continue to change.
2. The Router generator has a plugin interface in source, but Start currently replaces user generator plugins. Do not build Tanned on it without an upstream-supported contract.
3. Server-function extraction is compiler-driven. Dynamic imports are discouraged, and custom function IDs are experimental.
4. Start's Vite implementation uses multi-environment builds. A sidecar plugin must prove that extraction runs once when appropriate and invalidates both client and server graphs correctly.
5. Static server functions are experimental and can create security problems if applied to session-dependent operations.
6. Nitro's Vite adapter is under active development. Adapter behavior around streaming and response headers must be tested, not assumed.
7. Deferred hydration, RSC, and import protection are experimental and unnecessary for the first Tanned operation model.
8. The package versions in the monorepo intentionally move independently. Tanned should test a supported version matrix rather than assume all TanStack packages share a version.

## Questions the first prototype must answer

1. Can a PHP edit regenerate bindings and update a running route without restarting Vite?
2. Should generated operations use two generic server functions (`load` GET and `action` POST) or one server function per operation?
3. Can generated files stay outside `src/routes` while retaining an import style that feels colocated?
4. What PHP declarations are sufficient for PHPStan to infer nullable values, generics/collections, unions, enums, dates, paginated results, files, and validation errors?
5. How are the incoming `Cookie`, CSRF token, origin, host, forwarding headers, and backend `Set-Cookie` headers transformed at the gateway?
6. What exact backend outcomes become values, thrown errors, not-found values, or frontend redirect decisions?
7. Does request cancellation propagate from Router to the server function and onward to the backend fetch?
8. Which operations are safe during prerender, and how does a developer explicitly declare that capability?
9. Does the selected Node production adapter preserve multiple `Set-Cookie` headers and streamed responses unchanged?
10. Can the same public generated frontend API later target another backend integration without exposing Laravel-specific concepts?

## Decision enabled by this research

Proceed with a paired-route prototype using TanStack Start, React, Vite, and Laravel. Use normal Router loaders and a generated frontend operation interface backed by static Start server-function dispatchers. Treat the PHP sidecar as Tanned input, not as a TanStack route. Do not define a public Tanned wire protocol, extend the Router generator, adopt static server functions, or promise backend streaming until the prototype answers the questions above.
