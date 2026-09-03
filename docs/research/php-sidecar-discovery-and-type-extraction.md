# PHP sidecar discovery and type extraction

Research for [Assess PHP sidecar discovery and type extraction](https://github.com/aminevg/tanned/issues/7), conducted 2026-09-03.

## Decision summary

Tanned can support PHP files colocated with TanStack route files without asking the backend to register one route per frontend route and without adding the frontend tree to the backend's Composer autoload configuration.

The recommended shape is a build-time sidecar compiler and a runtime Laravel dispatcher:

1. Scan the configured TanStack routes directory for exact TypeScript/PHP pairs.
2. Statically parse each PHP file and accept only explicitly exported operation methods.
3. Resolve each operation's declared input and return types into a small Tanned intermediate representation (IR).
4. Generate three deterministic artifacts from that IR: an allowlisted PHP dispatch manifest, TypeScript declarations/callable bindings, and a machine-readable manifest for tooling.
5. Register one internal Laravel endpoint through an auto-discovered service provider. The endpoint looks up an opaque operation ID in the generated manifest, loads only that trusted file, resolves the sidecar through Laravel's container, hydrates only its declared transport input, and calls the selected method.

Composer should load the Tanned package and the sidecars' dependencies, not discover the sidecars. The generated manifest should load sidecar files explicitly. This keeps installation close to `composer require` plus configuration, works with arbitrary TanStack filenames, and avoids stale Composer classmaps during development.

For the first prototype, declared native/PHPDoc types should be the contract. Do not promise whole-body return-type inference or exact TypeScript types derived solely from Laravel validation rules. PHPStan is useful as an optional compiler backend and as a signature checker, but its semantic type graph still needs a Tanned-specific JSON/TypeScript mapping. The fastest credible starting point is a Tanned provider/extension over Spatie TypeScript Transformer 3, while prototyping the same fixture set through a PHPStan collector before choosing the long-term analysis engine.

## Proposed architecture

### Source convention

Given a frontend route:

```text
frontend/routes/users/index.tsx
```

Tanned discovers only the exact colocated sidecar:

```text
frontend/routes/users/index.php
```

The pairing algorithm should operate on paths, not PHP namespaces. TanStack file names may contain routing syntax such as `$id`, `_layout`, or parentheses that should not be forced into PSR-4 class names.

A first prototype should use one named, final class per sidecar because all currently viable reflection pipelines handle that shape well. The syntax below is illustrative rather than a settled public API:

```php
<?php

namespace App\Tanned\Users;

use Tanned\Action;
use Tanned\Load;

final class IndexSidecar
{
    #[Load]
    public function load(UsersQuery $input, UserRepository $users): UsersPageData
    {
        // ...
    }

    #[Action]
    public function delete(DeleteUserInput $input, UserService $users): DeleteUserResult
    {
        // ...
    }
}
```

Attributes make remote exposure opt-in. A public helper method is not automatically callable. The compiler can require zero or one transport input parameter in a fixed position and treat later class-typed parameters as Laravel dependencies. The precise syntax should be decided by a later interface prototype.

Anonymous returned classes or a file that returns a map of closures could remove namespace/class boilerplate. They are worth a later syntax experiment, but should not be the first extraction test: PHPStan can see anonymous classes in AST scope, while Spatie's `PhpNodeCollection::addByFile()` explicitly expects exactly one class, interface, or enum per file, and Laravel cannot constructor-inject an anonymous class through a stable class name. Method injection would still work.

### Generated artifacts

Use one normalized IR as the single compiler output, then render runtime and frontend artifacts from it. This is an internal build contract, not a public Tanned wire protocol.

```json
{
  "schemaVersion": 1,
  "contentHash": "...",
  "operations": {
    "users/index:load": {
      "kind": "load",
      "source": { "file": "frontend/routes/users/index.php", "line": 13 },
      "handler": { "class": "App\\Tanned\\Users\\IndexSidecar", "method": "load" },
      "input": { "$ref": "UsersQuery" },
      "output": { "$ref": "UsersPageData" }
    }
  },
  "schemas": {}
}
```

The PHP artifact should contain the same allowlist in executable form. The TypeScript artifact can expose an operation registry such as `Operation<Input, Output, Kind>` and generated bindings. The API's exact spelling remains a separate product decision.

Generation should be sorted and deterministic, write temporary files and rename atomically, include the compiler/schema version, remove only files recorded in the previous manifest, and fail CI when checked/generated output is stale. Spatie TypeScript Transformer's manifest already demonstrates content hashes, skipped unchanged writes, and deletion of previously generated files ([versioned manifest documentation](https://github.com/spatie/typescript-transformer/blob/3.3.1/docs/advanced/the-manifest-file.md)).

### Laravel runtime

Laravel package discovery can register a Tanned service provider from the package's `composer.json`; Laravel documents this as the standard automatic package setup ([Laravel 13 package discovery](https://laravel.com/docs/13.x/packages#package-discovery)). The provider can register one internal dispatch route and its middleware. No product/page routes are created by the backend.

On a request, the dispatcher should:

1. Resolve the operation ID only from the generated allowlist.
2. Verify the stored relative sidecar path resolves beneath the configured route root.
3. `require_once` that trusted path if the class is not already loaded.
4. Resolve the named sidecar class with `app()->make()`.
5. Decode and hydrate only the one declared transport input.
6. Invoke the method while Laravel fills the remaining dependencies.
7. Normalize the declared result to JSON and normalize known Laravel exceptions to the frontend primitives.

Laravel's container provides zero-configuration resolution for concrete dependencies and an explicit `call` method for invoking an object method or closure with method injection ([Laravel 13 service container](https://laravel.com/docs/13.x/container#method-invocation-and-injection)). Tanned must not pass the client's raw object as the named-parameter array to `Container::call`: a client field whose name matches a service parameter could otherwise override a container-provided dependency. Hydrate the transport DTO separately, pass only that object explicitly, and let the container resolve the rest.

The internal endpoint should run through the backend's normal session, authentication, authorization, CSRF, throttling, and exception middleware as configured. Laravel's CSRF mechanism accepts the encrypted `XSRF-TOKEN` cookie via `X-XSRF-TOKEN`, which the frontend gateway can forward ([Laravel 13 CSRF documentation](https://laravel.com/docs/13.x/csrf#x-xsrf-token)).

### Composer's role

Composer offers PSR-4, PSR-0, classmap, and eager `files` autoloading ([Composer schema: autoload](https://getcomposer.org/doc/04-schema.md#autoload)). None is a good primary sidecar discovery mechanism:

| Mechanism | Benefit | Problem for colocated route sidecars |
| --- | --- | --- |
| PSR-4 | New classes are found during development without regenerating a map. | Namespace/class names must correspond to paths. TanStack route tokens do not naturally form PHP identifiers, and a dependency package cannot silently add the project's frontend tree to the root package's PSR-4 map. |
| Classmap | Finds classes whose file paths do not follow PSR-4; optimized production lookup is fast. | Adding/removing sidecars requires `composer dump-autoload`; authoritative maps deliberately stop filesystem fallback. It still does not express the TypeScript/PHP pairing or operation allowlist. See [Composer's optimization behavior](https://getcomposer.org/doc/articles/autoloader-optimization.md). |
| `files` | No class naming requirement. | Eagerly executes every listed file whenever Composer loads and is generated at install/update time. This is wrong for route-local handlers. |
| Runtime `ClassLoader::addPsr4` | A package could add a namespace dynamically. | It retains the naming mismatch and interacts poorly with authoritative production classmaps. |

Therefore, Composer should autoload `tanned/laravel` and ordinary backend dependencies. The sidecar compiler owns discovery. The runtime manifest owns explicit file loading. A backend may opt into a normal PSR-4 namespace for editor support, but Tanned must not require it.

## Type extraction options

### Option A: native reflection plus PHPDoc parsing

Native reflection is simple and stable for native parameter/return types, defaults, attributes, visibility, and doc comments. It requires the class to be loaded, which executes the file, and it does not resolve PHPDoc aliases, generics, array shapes, or Laravel-specific semantics by itself.

This is adequate only if Tanned defines a deliberately small declaration subset and parses PHPDoc itself. It is not enough for the desired DTO/union/generic experience.

### Option B: PHP-Parser or BetterReflection with a Tanned type resolver

[nikic/PHP-Parser](https://github.com/nikic/PHP-Parser/tree/v5.8.0) parses PHP into an AST with source locations and namespace resolution. [Roave BetterReflection](https://github.com/Roave/BetterReflection/tree/6.72.0) reflects classes without loading them and exposes declarations, docblocks, and method ASTs; its own README says it is intended for static tooling, not runtime use.

This avoids executing sidecars during a build and gives Tanned full control. The cost is substantial: Tanned would need to resolve PHPDoc, aliases, inheritance, templates, Laravel wrappers, serializers, and custom extension types. These libraries are good primitives, but building the complete semantic layer immediately would duplicate mature work.

### Option C: a PHPStan collector/compiler extension

PHPStan's reflection layer exposes functions, classes, methods, parameters, variants, and return types through `ReflectionProvider` and `ParametersAcceptor` ([reflection documentation](https://phpstan.org/developing-extensions/reflection)). Its type system represents unions, intersections, constants, array/object shapes, templates, conditional types, and many other PHPDoc constructs ([PHPDoc type reference](https://phpstan.org/writing-php-code/phpdoc-types), [type-system guidance](https://phpstan.org/developing-extensions/type-system)). With Larastan enabled, the analysis also understands more Laravel conventions.

A supported integration can register a [Collector](https://phpstan.org/developing-extensions/collectors) for sidecar class/method nodes. Collectors receive the AST node and `Scope`, can inspect resolved types, return only scalar/array data suitable for worker transfer, and feed a `CollectedDataNode` rule in the main process. That final rule could validate and atomically emit Tanned IR. PHPStan 2.2 also allows rules to emit collected data directly.

Advantages:

- Best access to configured PHPStan/Larastan types, PHPDoc aliases, stubs, and project extensions.
- Static: the sidecar need not execute merely to extract its contract.
- Diagnostics can point to exact unsupported boundary types.
- The collector mechanism participates in PHPStan's parallel analysis and result cache.

Costs and hard limits:

- PHPStan provides a semantic PHP type graph, not JSON Schema or TypeScript. Tanned must still define and maintain every wire-type conversion.
- A method with no declared/PHPDoc return type remains unsuitable as a stable remote contract. Inferring a public response schema from every return expression and serializer path is a different, much larger analyzer.
- Dynamic return extensions generally resolve call sites; they do not make an underspecified endpoint declaration into a portable schema.
- Emitting files as a side effect of `phpstan analyse` is unusual. Parallel execution, cache hits, analysis failures, and users who do not run PHPStan must be tested carefully.
- Only `@api` surfaces are covered by PHPStan's extension backward-compatibility promise ([PHPStan extension BC promise](https://phpstan.org/developing-extensions/backward-compatibility-promise)). `ReflectionProvider`, `Collector`, `ParametersAcceptor`, and `ContainerFactory` are marked `@api` in PHPStan 2.2.12, but many analyzer internals are not. Tanned must avoid direct use of `Analyser`, `FileAnalyser`, and concrete internal types unless it pins PHPStan tightly. See the [2.2.12 ReflectionProvider source](https://github.com/phpstan/phpstan-src/blob/2.2.12/src/Reflection/ReflectionProvider.php), [Collector source](https://github.com/phpstan/phpstan-src/blob/2.2.12/src/Collectors/Collector.php), and [ContainerFactory source](https://github.com/phpstan/phpstan-src/blob/2.2.12/src/DependencyInjection/ContainerFactory.php).

PHPStan is therefore a viable high-fidelity backend, but not automatically the smallest or safest first implementation.

### Option D: extend Spatie TypeScript Transformer 3

Spatie TypeScript Transformer 3 already combines most compiler infrastructure Tanned needs:

- configurable directory/class discovery and transformers;
- `phpstan/phpdoc-parser`, BetterReflection, and `spatie/php-structure-discoverer` dependencies;
- TypeScript AST nodes for aliases, objects, unions/intersections, generics, conditional and mapped types, imports, parameters, and function-call expressions;
- provider and writer extension points;
- watch mode with PHP reflection wrappers that can be replaced by BetterReflection nodes after edits;
- a generated-file manifest with hashes and stale-file cleanup;
- a Laravel package whose beta controller provider generates callable route metadata and request/response types.

The released dependency set is visible in [typescript-transformer 3.3.1's composer.json](https://github.com/spatie/typescript-transformer/blob/3.3.1/composer.json). The [PHP-node documentation](https://github.com/spatie/typescript-transformer/blob/3.3.1/docs/watch-mode/php-nodes.md) allows a provider to add a specific file, with the restriction that it contain exactly one class/interface/enum. The [Laravel controller provider](https://github.com/spatie/laravel-typescript-transformer/blob/3.3.0/docs/controllers.md) already extracts method request/response types, arrays/shapes, collections, Laravel Data, pagination, and wrapped responses. Its documented limits are also relevant: FormRequest request types and Laravel Resource response types were not supported in 3.3.0, and unresolved responses degrade to `object`.

This is the best starting point for an end-to-end prototype. Tanned can implement its own sidecar discovery, operation provider, IR writer, and runtime PHP-manifest writer while reusing the type nodes and watch/generation machinery. Before adopting it as a core dependency, test whether its reflection/PHPDoc type fidelity covers the contract matrix below and whether its experimental watch mode is reliable in the intended monorepo/deployment layout. Tanned should fail on unsupported exported types instead of silently emitting `object`, `unknown`, or `any`.

### Recommendation across options

Prototype Option D and Option C behind the same tiny IR:

- Use Spatie's pipeline to reach a working colocated route, Laravel dispatch, and generated TypeScript call quickly.
- Implement a small PHPStan collector over the same signature fixtures to measure what additional fidelity it provides, especially for aliases, generics, Larastan-aware types, and anonymous-class syntax.
- Choose one extractor after measuring correctness, incremental latency, dependency weight, and public-API stability. Do not make both production requirements.
- Keep the IR and type-to-wire rules owned by Tanned so the extractor can change without changing frontend usage.

## Transport type rules

PHP types and JSON values are not identical. The first release should publish and enforce a conservative boundary rather than claim every PHPStan type is transferable.

| PHP declaration | Recommended TypeScript/wire treatment |
| --- | --- |
| `null`, `bool`, `int`, `float`, `string` | Direct scalar mapping; document that JSON/JavaScript cannot exactly represent all 64-bit integers. |
| Literal scalars and backed enums | TypeScript literal unions; encode enum backing values, not PHP object structure. |
| `list<T>` | `T[]`. |
| `array<string, T>` | `Record<string, T>`; reject or explicitly convert non-string/non-sequential JSON keys. |
| Sealed array shapes | Object/tuple types with optional keys preserved. |
| `A\|B` | TypeScript union only when both alternatives have supported serializers. Prefer a discriminator for object unions. |
| DTO class | Supported only through a registered, deterministic serializer/schema adapter. Public PHP properties alone do not prove the JSON shape. |
| Concrete generic wrapper such as `Page<UserData>` | Supported through a registered wrapper mapping. |
| Unbound templates, conditional return types, `static`, arbitrary intersections | Reject at an exported operation boundary unless a registered mapper resolves them completely. |
| `DateTimeInterface`, UUID/value objects, money | Require a registered encoding and corresponding TypeScript type; do not guess from the PHP class. |
| `iterable`, generators, resources, closures, callables | Reject unless materialized by an explicit adapter. |
| Eloquent models and base `JsonResponse`/`Response` | Treat as opaque and reject by default; hidden/appended fields, casts, resources, and conditional serialization make class reflection insufficient. |
| `void`/`never` | Explicit no-content or non-returning operation semantics, not an inferred JSON value. |
| uploaded files/binary streams | A later transport primitive; not ordinary JSON DTO fields. |

Optionality and nullability must remain distinct: a missing property is not the same as a present `null`. Defaulted method parameters may be optional at the call boundary; nullable types permit `null`. Recursive DTOs need cycle detection and named references in the IR.

Generics need special care. PHPStan can retain templates and resolve them at a call site, but a generated remote operation has no PHP call site. Only concrete exported instantiations or registered generic wrappers should cross the boundary. An exported `function identity<T>(T $value): T` has no finite, independently deployable wire schema and should fail generation.

## Validation and authorization

Laravel Form Requests combine `authorize()` and `rules()`, are resolved by the container, and are validated before a controller method runs ([Laravel 13 Form Request documentation](https://laravel.com/docs/13.x/validation#form-request-validation)). JSON validation failures are normalized to HTTP 422 with dot-notated field keys ([Laravel 13 validation response format](https://laravel.com/docs/13.x/validation#validation-error-response-format)). Tanned should preserve those runtime semantics and expose them through its frontend primitives.

Validation rules are not a complete static input type system:

- rules may be strings, arrays, objects, and arbitrary custom classes;
- `rules()` may receive container dependencies;
- closures can add rules conditionally through `sometimes`, `Rule::forEach`, or `Rule::excludeIf`;
- authorization and database-dependent `exists`/`unique` checks are constraints, not TypeScript types;
- middleware may trim strings or convert empty strings to `null` before validation;
- validation does not define the serializer or response shape.

The Laravel docs demonstrate [conditional `sometimes` closures](https://laravel.com/docs/13.x/validation#conditionally-adding-rules), [per-element `Rule::forEach`](https://laravel.com/docs/13.x/validation#accessing-nested-array-data), and [custom rule objects](https://laravel.com/docs/13.x/validation#using-rule-objects). Statically executing `rules()` during generation is also unsafe because it may require a live request, authenticated user, database, or environment-specific service.

Recommended rule:

- An operation has an explicit typed input DTO (or an explicit array shape) that defines the transport shape.
- A Form Request or validator defines runtime authorization and constraints.
- Tanned may derive UI metadata or check simple shape consistency from well-known validation rules, but this is best-effort and never replaces the DTO contract.
- The compiler should diagnose a clear contradiction, such as a required validation field absent from a sealed input shape, when it can prove one.

Spatie Laravel Data 4 is a strong optional adapter because one class can define hydration/serialization and can generate TypeScript, including nullable, optional, lazy, nested, and collection types ([Laravel Data TypeScript integration](https://spatie.be/docs/laravel-data/v4/advanced-usage/typescript)). It should not be mandatory: Tanned's Laravel promise includes existing codebases that use ordinary DTOs and Form Requests.

## Security implications

The dispatcher creates a remote-call boundary inside the backend. Treat compiler output as a capability allowlist.

- Never accept a class name, method name, or filesystem path from the request. Accept only an opaque operation ID and resolve it in the generated manifest.
- Export operations explicitly with an attribute or similarly unambiguous declaration. Do not export every public method discovered in a colocated file.
- Enforce `realpath` containment beneath configured roots, reject symlink escapes during generation, and store project-relative paths rather than developer-machine absolute paths.
- Reject duplicate operation IDs, duplicate sidecars, multiple classes when the selected syntax requires one, non-public/static/magic methods, variadics, by-reference parameters, and unsupported transport types.
- Separate the client payload from container injection. Never allow JSON keys to override framework services or the current request.
- Apply the backend's normal middleware and require each mutation to perform its normal authorization. Colocation is not authorization.
- Preserve Laravel's CSRF behavior for session-authenticated mutations. The frontend gateway must forward relevant cookies and headers in both SSR and browser navigation.
- Normalize production errors; do not return PHP class names, filesystem paths, stack traces, SQL, or serializer internals.
- Put a size/depth limit on decoded payloads and serialized responses. Detect object graph cycles.
- Generate in trusted build/development environments only. Do not run arbitrary sidecar discovery or PHPStan analysis in response to public HTTP traffic.
- In long-running workers such as Octane, load an immutable production manifest per worker lifecycle; development reload behavior must not leak old class definitions.

A stale frontend/backend pair is also a correctness risk. Include a content/schema hash in both generated sides, expose it in development diagnostics, and define a clear `unknown operation` response. Whether production deployments require a handshake is a later deployment decision.

## Maintained tools assessed

Versions are the latest GitHub releases observed on 2026-09-03.

| Tool | Version and release date | What Tanned should reuse or learn | Important limit |
| --- | --- | --- | --- |
| Composer | [2.10.3, 2026-08-27](https://github.com/composer/composer/releases/tag/2.10.3) | Package loading and the backend dependency graph. | Its autoload maps do not express route pairing or safe remote exports. |
| Laravel Framework | [13.30.1, 2026-09-01](https://github.com/laravel/framework/releases/tag/v13.30.1) | Package discovery, container resolution/calls, middleware, sessions, validation, authorization, exception handling. | Tanned must supply dispatch and serialization rules. |
| PHPStan | [2.2.12, 2026-08-31](https://github.com/phpstan/phpstan/releases/tag/2.2.12) | High-fidelity semantic types, reflection, collectors, diagnostics, project extensions. | No JSON/TypeScript schema generator; internal analyzer APIs are not all stable. |
| Larastan | [3.11.0, 2026-09-01](https://github.com/larastan/larastan/releases/tag/v3.11.0) | Laravel-aware PHPStan semantics when using the PHPStan backend. | Adds dependency/config coupling and cannot prove runtime serializer shapes in general. |
| Spatie TypeScript Transformer | [3.3.1, 2026-08-28](https://github.com/spatie/typescript-transformer/releases/tag/3.3.1) | Best first compiler foundation: discovery, type nodes, writers, watch mode, manifest. | Type fidelity and experimental watch behavior need the prototype matrix. |
| Spatie Laravel TypeScript Transformer | [3.3.0, 2026-06-19](https://github.com/spatie/laravel-typescript-transformer/releases/tag/3.3.0) | Laravel provider pattern and beta typed controller operations. | Route/controller-centric; documented lack of FormRequest and Resource support at this release. |
| Laravel Wayfinder | [0.1.21, 2026-08-04](https://github.com/laravel/wayfinder/releases/tag/v0.1.21) | Reference for Composer package discovery, Artisan generation, Vite-triggered builds/watch, callable TypeScript route functions, and generated-file layout. | Beta and centered on Laravel-registered routes/controllers; functions resolve URL/method rather than Tanned sidecar payload/result contracts. See its [versioned README](https://github.com/laravel/wayfinder/blob/v0.1.21/README.md). |
| Scramble | [0.13.42, 2026-08-21](https://github.com/dedoc/scramble/releases/tag/v0.13.42) | Strong reference for AST-based Laravel request/response inference, validation, resources, and extensible OpenAPI 3.1 generation. | Backend-route/OpenAPI centered and still pre-1.0; adopting OpenAPI would add a larger contract than Tanned currently needs. See [repository overview](https://github.com/dedoc/scramble/tree/v0.13.42). |
| Scribe | [5.11.0, 2026-06-08](https://github.com/knuckleswtf/scribe/releases/tag/5.11.0) | Reference for extracting FormRequest/validation metadata and extensible endpoint strategies. | Documentation-first; it may call endpoints for example responses, which Tanned should not do during contract generation. See [Scribe's maintained documentation](https://scribe.knuckles.wtf/laravel/). |
| Spatie Laravel Data | [4.23.0, 2026-05-08](https://github.com/spatie/laravel-data/releases/tag/4.23.0) | Optional typed DTO, validation, hydration, serialization, and TypeScript adapter. | Must remain optional to support ordinary Laravel codebases. |
| BetterReflection | [6.72.0, 2026-07-22](https://github.com/Roave/BetterReflection/releases/tag/6.72.0) | Safe build-time reflection without loading classes; anonymous-class/closure inspection. | Slower than native reflection and not a complete type-to-wire layer. |
| PHP-Parser | [5.8.0, 2026-07-04](https://github.com/nikic/PHP-Parser/releases/tag/v5.8.0) | Sidecar syntax/discovery checks, source locations, name resolution, deterministic AST inspection. | Syntax tree only; semantic type resolution is Tanned's responsibility. |
| Spatie PHP Structure Discoverer | [2.4.4, 2026-06-15](https://github.com/spatie/php-structure-discoverer/releases/tag/2.4.4) | Fast discovery by directory/interface/attribute; already used by Spatie's transformer. | Its cached object serialization has a reported Laravel 13 issue as of this research; a Tanned JSON manifest avoids that representation. |

## End-to-end prototype acceptance matrix

Build one small Laravel 13 + TanStack Start repository with colocated frontend routes and test these questions before fixing the public API.

### 1. Discovery and lifecycle

- Pair `index.tsx`/`index.php`, dynamic route tokens such as `$user.tsx`/`$user.php`, nested layouts, and route groups.
- Add, edit, rename, move, and delete a sidecar while the frontend dev server and Laravel backend run.
- Detect orphan PHP files, duplicate/colliding route IDs, more than one exported class, invalid PHP during an edit, and symlinks outside the configured root.
- Prove production dispatch with `composer install --classmap-authoritative` and no project-level autoload mapping for the frontend directory.
- Compare full and incremental generation time for 10, 100, and 1,000 synthetic sidecars.
- Verify generated artifacts are byte-for-byte deterministic across absolute checkout paths and operating systems.

### 2. Laravel execution

- Constructor and method injection of concrete services and bound interfaces.
- The current request, authenticated user/session, policies/gates, database transactions, and a Form Request or equivalent validator.
- Success, unauthenticated, forbidden, not found, validation 422, domain conflict, and unhandled error responses.
- CSRF success/failure through the frontend gateway, including SSR and hydrated browser calls.
- A load operation and a mutating action; confirm the backend defines only the internal dispatcher route.
- Normal PHP-FPM and a long-running Octane worker.

### 3. Security failures

- Request an unknown operation ID, another sidecar's unexported method, a magic method, an arbitrary FQCN, path traversal, and a symlink escape.
- Send payload keys equal to injected parameter names and prove they cannot replace container services.
- Tamper with/mismatch the generated manifest and confirm a closed failure with useful development diagnostics and safe production output.
- Exercise payload depth/size limits and recursive output cycles.

### 4. Type fixtures

Compile and run round-trip tests for:

- primitives, literals, nullability, optional/defaulted fields, and backed enums;
- lists, string maps, tuples, sealed/nested array shapes, and dot-notated validation errors;
- nested DTOs, DTO inheritance, renamed/hidden/optional fields, and a recursive DTO;
- supported object unions with and without discriminators;
- concrete generic wrappers such as `Page<UserData>` and an intentionally unbound template;
- Laravel collections and paginators;
- `DateTimeInterface`, UUID/value objects, Eloquent models, API Resources, Laravel Data, `JsonResponse`, `void`, files, streams, and generators;
- native types versus PHPDoc, local/imported PHPStan aliases, stubs, and a Larastan extension-provided type;
- a conditional/dynamic return type and a missing return declaration.

Every fixture must either generate a precise TypeScript type and round-trip through the actual serializer or fail with a source-located compiler error. `any`, `unknown`, and generic `object` are not acceptable silent fallbacks for an exported operation.

### 5. Extractor comparison

Run the same fixtures through:

1. a Tanned provider built on Spatie TypeScript Transformer 3; and
2. a PHPStan collector that writes the same IR.

Measure clean build latency, one-file incremental latency, memory, behavior when the application has PHPStan errors unrelated to Tanned, and compatibility with the project's PHPStan/Larastan configuration. Record which cases produce different semantic types. This evidence is enough to choose an extractor without binding the frontend API to it.

### 6. Generated frontend contract

- Compile the generated module under strict TypeScript.
- Call load/action operations from ordinary TanStack Start loaders/actions without handwritten auth, CSRF, URL, or validation-error plumbing.
- Prove an input signature change and an output signature change produce useful frontend compile failures.
- Prove tree-shaking or per-route generation does not pull every operation into each browser chunk.
- Verify the frontend and PHP manifests carry the same content/schema hash.

## Prototype exit decision

Proceed with the architecture if the prototype proves all of the following:

- no root Composer autoload edit is needed;
- the generated allowlist is secure and works with normal Laravel middleware/container behavior;
- supported input/output declarations round-trip exactly through JSON;
- unsupported declarations fail clearly;
- add/move/delete feedback is fast enough for route-file development;
- the chosen analyzer uses maintained public extension surfaces or is isolated behind the Tanned IR;
- the frontend can consume operations consistently without knowing which PHP extractor produced them.

If Spatie's pipeline passes the matrix, use it for the first release and keep a PHPStan rule for contract enforcement. If only the PHPStan collector resolves required real-world declarations accurately, use PHPStan as the build-time compiler backend and pin/test its supported APIs. If neither can make serialization agree with generated types without extensive inference, narrow the first-release contract to explicitly marked DTOs and registered wrapper/serializer adapters rather than weakening types to `unknown`.
