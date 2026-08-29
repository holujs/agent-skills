---
name: holu-extensions
description: Practical guidance for creating, registering, ordering, exporting, grouping, and overriding Holu extensions. Use when writing custom extensions with stage1/stage2/stage3 hooks, injecting ExtensionManager to retrieve same-module or app-wide data (handling delay), dynamically registering providers, configuring module extensions metadata, or resolving cyclic extension dependencies.
---

# Holu Extensions

## Mental Model

Extensions run **after** Holu collects static metadata from decorators but **before** request handlers are created. Think of them as "infrastructure setup" hooks that operate on the fully assembled module graph.

- Extensions operate strictly during the application initialization stage to set up infrastructure and do not participate directly in request processing.
- Extensions prepare infrastructure: dynamically add interceptors/providers, build route metadata, set up OpenAPI docs, initialize DB connections.
- Extensions can be async and depend on each other through `ExtensionManager`.
- **Module Scope**: An extension is instantiated and executed **only in modules where it is explicitly registered** in the `extensions` array (or imported via `export: true` / `exportOnly: true`). It does not automatically execute in modules that do not register or import it.

---

## Implementing An Extension

An extension is a class decorated with `@injectable()` that implements the `Extension<T>` interface, where `T` specifies the payload return type of `stage1()` (and no other stage method). While each individual stage method is optional, an extension **must** implement at least one of the three stages (otherwise an `InvalidExtension` error is thrown).

> [!IMPORTANT]
> The stages `stage1`, `stage2`, and `stage3` are framework lifecycle hooks that are **always executed sequentially** in that order during the application bootstrap process. They are never skipped or executed out of order. Therefore, you can rely on state populated in `stage2` (like a module injector instance) to always be available in `stage3`.

```ts
import { injectable, inject, PROVIDERS_PER_APP } from '@holu/core';
import type { Extension, ExtensionManager, Injector, Provider } from '@holu/core';

@injectable()
export class MyExtension implements Extension<MyPayload | void> {
  constructor(
    private extensionManager: ExtensionManager,
    @inject(PROVIDERS_PER_APP) protected providersPerApp: Provider[],
  ) {}

  async stage1(isLastModule: boolean): Promise<MyPayload> {
    // isLastModule === true when this is the last module that imports this extension.
    // Return a payload (e.g., collected metadata) that other extensions can consume
    // via ExtensionManager.stage1().
  }
  async stage2(injectorPerMod: Injector): Promise<void> {
    // Stage 2: called after stage1 finishes for ALL modules.
    // Receives a ready module-level injector. Use this to resolve providers
    // or set data on the Context service (`injectorPerMod.get(Context)`).
  }

  async stage3(): Promise<void> {
    // Stage 3: called after stage2 finishes for ALL modules.
    // No strict role — use for final post-processing.
  }
}
```

> [!TIP]
> In `stage2` or `stage3`, instead of dynamically adding a `ValueProvider` for configurations, it is highly recommended to retrieve the core `Context` service (`injectorPerMod.get(Context)`) and store global configuration objects or extension states via `context.set(TOKEN, data)`.

### Constructor Injection Constraints

- The injector for the extension constructor is created **before any extension runs**.
- You can only inject providers statically registered (module-level or app-level) before extensions execute, or default extension providers exported via `defaultExtensionProviders` from `@holu/core`.
- You **cannot** inject providers that are dynamically added by other extensions.
- The extension and application injectors belong to separate hierarchy trees. The extension injector is short-lived and destroyed once the extensions finish initializing. Because any providers instantiated within the extension's constructor belong to this temporary injector, they are not shared with the main application.

### `isLastModule` Parameter

`stage1(isLastModule: boolean)` receives `true` only for the final module that imports this extension. Use this flag to defer aggregation work (e.g., finalizing a data structure) until all modules that import this extension have contributed.

### Accessing Extension Metadata (`extensionsMeta`)

Extension-specific configurations passed via `@featureModule({ extensionsMeta: { ... } })` or dynamic module options (e.g. `MyModule.withParams({ extensionsMeta: { ... } })`) are normalized by `ModuleNormalizer` into `normalizedModuleMeta.extensionsMeta`.

An extension retrieves `extensionsMeta` from `normalizedModuleMeta` depending on how it receives module metadata:

1. **Via `ResolvedModuleMeta` in constructor**:

   ```ts
   const myOptions = this.resolvedModuleMeta.normalizedModuleMeta.extensionsMeta[MY_EXTENSION];
   ```

2. **Via `RouteExtensionMeta` from `ExtensionManager`**:
   ```ts
   const myOptions = routeExtensionMeta.normalizedModuleMeta.extensionsMeta[MY_EXTENSION];
   ```

In all cases, `extensionsMeta` is accessed directly as a property of `normalizedModuleMeta` (`normalizedModuleMeta.extensionsMeta`). Keep each extension's metadata stored under its own dedicated key. For full runnable class implementations and patterns for passing and consuming `extensionsMeta`, see [references/REFERENCE.md](references/REFERENCE.md#extensionsmeta-usage-pattern).

---

## Registering An Extension

Add extensions to the `extensions` array of any module decorator (`@featureModule`, `@restModule`, etc.).

> [!IMPORTANT]
> Extensions are strictly module-scoped. Registering an extension in a module (including the root module) executes it **only** within that specific module instance, unless it is exported to importing modules using `export: true` or `exportOnly: true`.

### Direct Class Registration

Simplest form — runs only in the current module with no ordering constraints:

```ts
@restModule({
  extensions: [MyExtension],
})
export class AppModule {}
```

### Config Object Registration

Use the config object form for ordering, exporting, grouping, or overriding:

```ts
@restModule({
  extensions: [
    {
      extension: MyExtension,
      beforeExtensions: [ConsumerExtension], // MyExtension runs before ConsumerExtension
      afterExtensions: [ProducerExtension], // MyExtension runs after ProducerExtension
      groups: [RestRouteExtension], // MyExtension joins the RestRouteExtension group
      export: true, // also runs in every module that imports this module
    },
  ],
})
export class FeatureModule {}
```

`export` and `exportOnly` are mutually exclusive. `beforeExtensions`, `afterExtensions`, and `groups` are **not** available when using `overrideExtension` (see below).

---

## Configuration Properties Reference

### Ordering

| Property           | Meaning                                          |
| ------------------ | ------------------------------------------------ |
| `beforeExtensions` | This extension runs **before** each listed class |
| `afterExtensions`  | This extension runs **after** each listed class  |

Both accept an array of `ExtensionClass` values.

### Export Behaviour

| Property           | Runs in host module? | Runs in importing modules? |
| ------------------ | -------------------- | -------------------------- |
| _(default)_        | Yes                  | No                         |
| `export: true`     | Yes                  | Yes                        |
| `exportOnly: true` | **No**               | Yes                        |

> [!TIP]
> When registering extensions that are only intended to run in importing modules (such as in wrapper, library, or shared modules), always prefer `exportOnly: true` over `export: true`. This prevents the extension from executing unnecessarily in the host module itself, which can lead to redundant execution or runtime errors due to missing configuration/routes in the host.

### Overriding

Replace an imported extension with a custom one. The override form **only** accepts `extension` and `overrideExtension` — `beforeExtensions`, `afterExtensions`, `groups`, `export`, and `exportOnly` are **not** allowed in this form:

```ts
extensions: [{ extension: MyExtension, overrideExtension: ExternalExtension }];
```

`MyExtension` is registered under the token `ExternalExtension`, so all existing consumers transparently receive `MyExtension`.

---

## Extension Groups

An **extension group** allows multiple extensions to contribute data under the class token of a lead extension. Analogous to HTTP interceptor groups, an extension group represents a specific type of work performed over application metadata.

```ts
extensions: [
  {
    extension: OpenApiRouteExtension,
    groups: [RestRouteExtension], // joins the RestRouteExtension group
    export: true,
  },
];
```

> [!NOTE]
> An extension group does **not** require a special token class or dedicated group entity. Any ordinary extension class (such as `RestRouteExtension`) acts as the group token simply when other extensions list it in their `groups` array.

### Key Rules & Requirements for Extension Groups

#### 1. Shared Base Interface Contract (Critical Requirement)

All extensions belonging to a specific group **must** return data from `stage1()` that adheres to a **shared base interface**.

- **Contract Guarantee**: Consumers that call `extensionManager.stage1(LeadExtension)` rely on the base interface shape to safely iterate and process elements in `groupData`.
- **Extensibility**: Group members can extend the base interface with additional properties (e.g., `OpenApiRouteExtension` returning extended route metadata with OpenAPI schemas), but must **never** narrow or break the base interface contract.

#### 2. Automatic Execution Ordering via Group Membership

When an extension joins a group via `groups: [LeadExtension]`, it automatically inherits the group's positioning relative to other extensions in the execution graph:

- If `LeadExtension` was declared with `beforeExtensions: [TargetExtension]`, any extension joining `groups: [LeadExtension]` is automatically placed in the execution queue **after** `LeadExtension` and **before** `TargetExtension` (for example, since `RestRouteExtension` runs before `DispatcherExtension`, any extension joining `groups: [RestRouteExtension]` will automatically run after `RestRouteExtension` and before `DispatcherExtension`).
- You do **not** need to explicitly repeat `beforeExtensions: [TargetExtension]` on member extensions. This allows third-party modules to seamlessly integrate into existing extension pipelines without manual ordering configuration.

#### 3. Group Lookup Behavior & Asymmetry in `ExtensionManager`

Calling `extensionManager.stage1()` with an extension class yields different results depending on whether it is queried as a lead extension or as a specific member:

- **Lead Extension (Group) Lookup**: `await extensionManager.stage1(LeadExtension)` returns aggregated `groupData` containing the payload from `LeadExtension` **and** all member extensions registered in its group (`groups: [LeadExtension]`).
- **Direct Member Lookup**: `await extensionManager.stage1(SpecificMemberExtension)` returns `groupData` containing **only** the payload from `SpecificMemberExtension`.
- **Index Alignment**: Elements in `groupData` map 1-to-1 by array index to debug entries in `groupDebugMeta`:
  ```ts
  groupData[i] === groupDebugMeta[i]?.payload;
  ```

#### 4. Group Configuration Rules

- **Multiple Groups**: An extension can belong to multiple groups by passing multiple class tokens: `groups: [ExtensionA, ExtensionB]`.
- **Override Constraint**: If an extension is registered using `overrideExtension`, the `groups` property (along with `beforeExtensions`, `afterExtensions`, `export`, and `exportOnly`) is **not allowed**.

For a detailed step-by-step example demonstrating group formation, lead extension inclusion, and expansion, see [Extension Group Formation & Lead Extension Membership Example](references/REFERENCE.md#extension-group-formation--lead-extension-membership-example).

---

## Using ExtensionManager

`ExtensionManager` manages extension dependencies, caches stage1 results, and detects cyclic dependencies.

> [!WARNING]
> An extension **cannot** request its own `stage1` data via `ExtensionManager.stage1()` (e.g., `await this.extensionManager.stage1(CurrentExtension)` or `await this.extensionManager.stage1(CurrentExtension, this)`). Requesting itself creates a self-referential cyclic dependency and will throw a cyclic dependency error.

Inject it in the constructor:

```ts
constructor(private extensionManager: ExtensionManager) {}
```

### Same-Module Data

Retrieves the `stage1` output of `TargetExtension` (or its group) within the current module only:

```ts
const meta = await this.extensionManager.stage1(TargetExtension);
// meta.groupData — array of T values returned by each extension in the group/class
// meta.delay     — always false for same-module calls
```

> [!NOTE]
> If `TargetExtension` is not imported into the current module, this expression will not return data.

### App-Wide Data (Cross-Module)

Pass `this` as the second argument to receive aggregated data from modules where `TargetExtension` is registered:

```ts
const meta = await this.extensionManager.stage1(TargetExtension, this);
```

A separate instance of your extension is created for each module where it is imported. Passing `this` instructs the framework to accumulate `stage1` data across modules where `TargetExtension` is imported (e.g., if an application has 10 modules and `TargetExtension` is imported in only 2 of them, data is accumulated from those 2 modules).

#### Handling `delay`

When requesting app-wide data you **must** check `meta.delay`. If `true`, not all modules that import `TargetExtension` have completed `stage1` yet — return early; the framework will call your extension's `stage1(true)` method again later during a dedicated `perAppHandling` phase (after all standard module processing completes):

```ts
const meta = await this.extensionManager.stage1(TargetExtension, this);
if (meta.delay) {
  return; // not ready yet — framework will re-invoke this extension (perAppHandling)
}

// meta.groupDataPerApp: AppExtensionGroupMeta<T>[]
//   Each entry = result from one module that registered TargetExtension
meta.groupDataPerApp.forEach((perModMeta) => {
  // perModMeta.groupData: T[]  — payloads from that module
  perModMeta.groupData.forEach((payload) => {
    // process payload
  });
});
```

For the full `ExtensionGroupMeta` type shape, see [references/REFERENCE.md](references/REFERENCE.md).

---

## Dynamic Provider Registration

Extensions can dynamically push into provider arrays that are still being assembled. Providers can be pushed to:

1. **Application-level metadata** (`providersPerApp` injected via `@inject(PROVIDERS_PER_APP)`): Affects the entire application across all modules.
2. **Module-level metadata** (`normalizedModuleMeta.providersPerMod`, `providersPerRou`, `providersPerReq`): Affects the entire current module.
3. **Controller/Route-level metadata** (`controllersMeta[i].providersPerRou` or `controllersMeta[i].providersPerReq`): Affects only the specific controller or route (e.g. selectively adding interceptors as in `BodyParserExtension`).

| Stage    | Allowed provider levels               |
| -------- | ------------------------------------- |
| `stage1` | Any: app, module, route, request      |
| `stage2` | Only **below** module: route, request |

### Typical Pattern: Reading Config, Pushing Interceptors

> [!IMPORTANT]
> If you need to read statically declared module-level configuration in `stage1`, you must build a temporary injector. This requires injecting the `PROVIDERS_PER_APP` array in the constructor via `@inject(PROVIDERS_PER_APP)` so you can resolve it.

```ts
async stage1(): Promise<void> {
  const meta = await this.extensionManager.stage1(RestRouteExtension);

  meta.groupData.forEach((routeExtensionMeta) => {
    const { providersPerMod } = routeExtensionMeta.normalizedModuleMeta;

    // Pushing to controller/route-level metadata specifically:
    routeExtensionMeta.controllersMeta.forEach(({ providersPerRou, providersPerReq }) => {
      // Build a temporary injector hierarchy to inspect config
      const injectorPerApp = Injector.resolveAndCreate(this.providersPerApp, 'App');
      const injectorPerMod = injectorPerApp.resolveAndCreateChild(providersPerMod);
      const config = injectorPerMod.get(MyConfig, null);

      if (config?.enableFeature) {
        // Pushing to individual controller/route providersPerReq array (controller-scoped)
        providersPerReq.push({
          token: HTTP_INTERCEPTORS,
          useClass: MyInterceptor,
          multi: true,
        });
      }
    });
  });
}
```

Always ensure your extension is ordered **after** the extension that produces the metadata (e.g., after `RestRouteExtension`) and **before** the extension that consumes it (e.g., before `DispatcherExtension`).

### Transferring State & Resources to the Application (`ValueProvider`)

Since the extension's internal injector is temporary and destroyed post-bootstrap, any state or runtime resource initialized by an extension (e.g., database connection pools, SDK clients, external service configurations, compiled caches) must be transferred to the application's long-lived DI hierarchy.

To pass a ready object or resource instance to the application:

1. Initialize or connect to the resource asynchronously inside `stage1()` or `stage2()`.
2. Push a `ValueProvider` (`{ token: 'some-token', useValue: initializedInstance }`) into the appropriate provider hierarchy array (`providersPerApp`, `providersPerMod`, `providersPerRou`, or `providersPerReq`).

```ts
@injectable()
export class DbExtension implements Extension<void> {
  constructor(@inject(PROVIDERS_PER_APP) protected providersPerApp: Provider[]) {}

  async stage1(): Promise<void> {
    // 1. Asynchronously create/initialize the resource during bootstrap
    const dbClient = await createDbConnection({/* config */});

    // 2. Register the existing instance into application DI via ValueProvider
    this.providersPerApp.push({ token: DbClient, useValue: dbClient });
  }
}
```

Any application service or controller can then inject `DbClient` via standard DI.

---

## Registration Decision Guide

| Goal                                    | Configuration                                           |
| --------------------------------------- | ------------------------------------------------------- |
| Run only in this module                 | `extensions: [MyExtension]`                             |
| Control execution order                 | Object form with `beforeExtensions` / `afterExtensions` |
| Run in this module AND importers        | `export: true`                                          |
| Run in importers only (not host)        | `exportOnly: true`                                      |
| Contribute to an existing data pipeline | `groups: [TargetExtension]`                             |
| Replace an imported extension           | `{ extension: MyExt, overrideExtension: OriginalExt }`  |

---

## Troubleshooting

| Symptom                                    | Fix                                                                                                          |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------ |
| Extension does not run in importing module | Add `export: true` or `exportOnly: true` in host module                                                      |
| Extension runs too early                   | Add `afterExtensions: [ProducerExtension]`                                                                   |
| Extension runs too late                    | Add `beforeExtensions: [ConsumerExtension]`                                                                  |
| Group data is empty or missing             | Verify `groups: [TargetExtension]` uses the correct class token                                              |
| `groupDataPerApp` is incomplete            | Ensure `this` is passed as second arg to `extensionManager.stage1()` and `delay` is handled                  |
| Circular dependency error                  | Check the error log — `ExtensionManager` prints the full loop path; use `afterExtensions` to break the cycle |
| `UndeclaredExtensionDependency` error      | Extension A called `extensionManager.stage1(B)` but B was not listed in A's `afterExtensions`                |

---

## Further Reading

For full type definitions (`Extension<T>`, `ExtensionGroupMeta<T>`, `AppExtensionGroupMeta<T>`, `ExtensionConfig` union), see [references/REFERENCE.md](references/REFERENCE.md).
