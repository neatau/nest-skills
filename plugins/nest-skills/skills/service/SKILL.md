---
name: service
description: >-
  Create and manage NestJS DI services (@Injectable providers within a module —
  NOT microservices). Use when the user asks to create a new service, add a
  service to a module, refactor a service's dependency injection, decide whether
  a service needs a FactoryProvider, or inject config/clients into a service.
  Enforces conventions for file naming and placement, config-via-options (a
  service must never inject the whole ConfigService), when a FactoryProvider is
  warranted, and how dependencies are declared.
---

# NestJS Service Creation & Management

Follow these conventions when creating or modifying an `@Injectable()` service
(a DI-layer provider inside a module — not a `@nestjs/microservices` transport).

See `reference.md` in this skill folder for two complete, copy-ready worked
examples (a plain service and a service with a FactoryProvider). Read it before
writing code if you are unsure of the exact shape.

## Files and placement

A service lives under `<module>/services/<name>/` and is split by concern:

| File | Contains | When |
|---|---|---|
| `<name>.service.ts` | The `@Injectable()` `<Name>Service` class | Always |
| `<name>.service.provider.ts` | The `FactoryProvider` | Only if non-standard DI is needed (see below) |
| `<name>.service.types.ts` | Service-level types, incl. `<Name>ServiceOptions` | When the service has its own types/options |
| `<name>.service.support.ts` | Closely related helper functions that don't belong as methods | When such helpers exist |

Example: an `ArticleService` in the `news` module →
`news/services/article/article.service.ts`.

## Core rules

1. **Never inject application-wide config.** A service must not receive the
   first-party `ConfigService` (or any whole-app config object) via its
   constructor. If it needs configuration, define a narrow
   `<Name>ServiceOptions` type in `<name>.service.types.ts` and supply it from a
   FactoryProvider, which is the only place allowed to read `ConfigService`.
2. **Injected dependencies are `private readonly`.** Every constructor
   dependency is declared `private readonly`.
3. **Only introduce a FactoryProvider when DI is non-standard** (next section).
   Otherwise register the plain class and let Nest construct it.

## Decision: does this service need a FactoryProvider?

Create `<name>.service.provider.ts` **only if any** of these apply:

- **Config via options** — the service needs specific application configuration,
  delivered as a `<Name>ServiceOptions` object built in the factory.
- **Constructed clients** — it depends on clients/instances created in the
  factory: axios instances, SDK clients, or infrastructure connections (Redis,
  RabbitMQ, …).
- **Async setup** — a dependency needs async initialisation (opening a TCP
  connection, database/socket handshake). `useFactory` can be `async`.
- **`@Inject()`-token dependencies** — it depends on things that would otherwise
  require `@Inject()` in the constructor; move them into the factory so the
  service constructor stays clean.

If **none** apply, use a plain `@Injectable()` class registered directly in the
module's `providers` array — do **not** add a provider file.

## Procedure

1. Resolve the target module and service name → path
   `<module>/services/<name>/`.
2. Write `<name>.service.ts` with the `@Injectable()` class; all deps
   `private readonly`.
3. Run the decision checklist above.
   - **No FactoryProvider:** register the class in the module's `providers` (and
     `exports` if other modules use it).
   - **FactoryProvider needed:** add `<name>.service.types.ts` (with
     `<Name>ServiceOptions` if config is involved) and `<name>.service.provider.ts`.
     In the factory, read `ConfigService`/build clients/await async setup, then
     `return new <Name>Service(options, ...deps)`. Register the *provider* in the
     module (`provide` is the class token); export the class token if needed.
4. Extract closely related non-method helpers into `<name>.service.support.ts`.
5. Keep the constructor free of `@Inject()` — if a token dep appears, that is a
   signal to move it into a FactoryProvider.
