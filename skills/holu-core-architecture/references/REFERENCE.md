# Holu Core Architecture — Technical Reference

This guide contains detailed reference material, API definitions, type shapes, and execution examples for Module, Dependency Injection (DI), and Reflector mechanics. Use it on demand for deep diagnostics, advanced configurations, and integrations.

## Part 1: Dependency Injection References

### Reading `Resolution path` Errors

When DI cannot resolve a token, it throws an error with a `Resolution path`. Understanding this path is critical for diagnosing hierarchy misconfigurations.

#### Format

```
Error: No provider for [MissingToken in injectorName]!
Resolution path: [TokenA in injectorX >> injectorY] -> [MissingToken in injectorY]
```

- **`TokenA in injectorX >> injectorY`** — DI searched for `TokenA` starting in `injectorX`, did not find it there, and continued upward through `>>` until it found it in `injectorY`.
- **`->`** — separates the found provider from the dependency it requires. The dependency search starts from the injector where the provider was found (not from where the request originated).
- **`MissingToken in injectorY`** — the dependency `MissingToken` was searched starting from `injectorY` (upward) and was not found anywhere.

#### Injector Naming

Holu auto-names injectors as `injector1` (highest/app level), `injector2`, `injector3`, etc. (lower = higher number). Pass an explicit second argument to `Injector.resolveAndCreate(providers, 'MyName')` for clearer diagnostics. In a full Holu application the levels are named `App`, `Mod`, `Rou`, `Req`.

#### Diagnostic Examples

**Example 1 — dependency placed in child injector:**

```ts
// Setup
const parent = Injector.resolveAndCreate([Service], 'parentInjector');
const child  = parent.resolveAndCreateChild([{ token: Config, useValue: {...} }], 'childInjector');

child.get(Service);
// Error: No provider for [Config in parentInjector]!
// Resolution path: [Service in childInjector >> parentInjector] -> [Config in parentInjector]
```

Reading this: `Service` was not found in `childInjector`, was found in `parentInjector`. From `parentInjector`, DI searched for `Config` — also in `parentInjector` and its ancestors — but found nothing. Fix: move `Config` to `parentInjector` or add `Service` to `childInjector` (so it uses the child's `Config`).

**Example 2 — Holu application level names:**

```
Error: No provider for [Config in Mod >> App]!
Resolution path: [Service in Req >> Rou >> Mod] -> [Config in Mod >> App]
```

Reading this: `Service` was searched in `Req`, `Rou`, and finally found in `Mod`. From `Mod`, DI searched for `Config` through `Mod` and `App` but found nothing. Fix: register `Config` in `providersPerMod` or `providersPerApp`.

### Complete Provider Type Reference

```ts
import { Class } from '@holu/core';

type Provider =
  | Class<any> // TypeProvider (shorthand)
  | { token: any; useValue?: any; multi?: boolean } // ValueProvider
  | { token: any; useClass: Class<any>; multi?: boolean } // ClassProvider
  | { token?: any; useFactory: [Class<any>, Function]; multi?: boolean } // ClassFactoryProvider
  | { token?: any; useFactory: (...args: any[]) => any; deps: any[]; multi?: boolean } // FunctionFactoryProvider
  | { token: any; useToken: any; multi?: boolean }; // TokenProvider
```

Named provider types importable from `@holu/core`:

```ts
import { TypeProvider, ValueProvider, ClassProvider, FactoryProvider, TokenProvider } from '@holu/core';
```

#### `useFactory` token is optional

For factory providers, `token` is optional because DI can use the factory function or the class method itself as a token.

### Injector Hierarchy Simulation

Full four-level hierarchy simulation matching a real Holu application:

```ts
import { Injector, Provider } from '@holu/core';

const providersPerApp: Provider[] = [];
const providersPerMod: Provider[] = [];
const providersPerRou: Provider[] = [];
const providersPerReq: Provider[] = [];

const injectorPerApp = Injector.resolveAndCreate(providersPerApp, 'App');
const injectorPerMod = injectorPerApp.resolveAndCreateChild(providersPerMod, 'Mod');
const injectorPerRou = injectorPerMod.resolveAndCreateChild(providersPerRou, 'Rou');
const injectorPerReq = injectorPerRou.resolveAndCreateChild(providersPerReq, 'Req');

injectorPerApp === injectorPerMod.parent; // true
```

Holu repeats this procedure for each module, route, and HTTP request.

### `Context.getInScope()` vs `Context.get()`

Both methods retrieve values by key, but they differ in traversal scope:

- **`ctx.get(key)`** — retrieves the value from the current `Context` instance only.
- **`ctx.getInScope(key, injector)`** — traverses **up** the injector hierarchy, checking each `Context` instance at each level, returning the first match found. Useful when `Context` instances exist at multiple injector levels (e.g., parent module sets a value, child module reads it).

```ts
const parent = Injector.resolveAndCreate([Context], 'parent level');
const child = parent.resolveAndCreateChild([Context], 'child level');

parent.get(Context).set('key1', 'value1');
child.get(Context).set('key2', 'value2');

child.get(Context).getInScope('key1', child); // 'value1'  (found in parent Context)
child.get(Context).getInScope('key2', child); // 'value2'  (found in child Context)
```

### `@inject` + `@input` Full Example

```ts
import { injectable, inject, input, Injector } from '@holu/core';

@injectable()
class Dependency1 {
  constructor(@input public inputParameter: string) {}
}

@injectable()
class Service1 {
  constructor(@inject(Dependency1, 'input-data') public dependency1: Dependency1) {}
}

const injector = Injector.resolveAndCreate([Service1, Dependency1]);
const service1 = injector.get(Service1);
service1.dependency1.inputParameter; // 'input-data'
```

Key rules:

- `@input` (without parentheses) is shorthand for `@inject(input)`.
- When a second argument is passed to `@inject()`, **no cache** is created for that dependency — a new instance is created each time.
- `input` itself can be used as a token in `deps` arrays of function factories.

### ParentParams Alternative Patterns

When extending a parent class in an `@injectable()` class, the recommended approach in `SKILL.md` uses `@ts-expect-error`. If type suppression is not desired, these alternative type-safe approaches can be used:

#### Option B — `@inject` decorator (type-safe, no suppression)

```ts
import { ParentParams, injectable, inject } from '@holu/core/di';

@injectable()
class Child extends Parent {
  constructor(
    @inject(ParentParams) parentParams: ConstructorParameters<typeof Parent>,
    public childParam1: ChildParam1,
  ) {
    super(...parentParams);
  }
}
```

#### Option C — inline type assertion (type-safe, no suppression)

```ts
@injectable()
class Child extends Parent {
  constructor(
    parentParams: ParentParams,
    public childParam1: ChildParam1,
  ) {
    super(...(parentParams as ConstructorParameters<typeof Parent>));
  }
}
```

### Parameter Validation & Transformation (Pipes) via Factory Providers

In Holu, route parameter validation and transformation (analogous to NestJS pipes) can be implemented using custom `FactoryProvider` logic paired with `@input` or the `input` token.

When `@inject(Token, 'paramName')` is used on a controller method parameter:

1. The second argument (`'paramName'`) is passed to the factory as an `input` parameter.
2. The DI injector skips caching for this dependency, executing the factory function/method afresh for each injection site.
3. The custom factory logic (written by the developer) receives `input`, reads raw parameter data from request context (`PATH_PARAMS`, `QUERY_PARAMS`, or `Context`), performs custom validation (throwing `BadRequestError` if invalid), and returns the transformed value.

#### 1. `ClassFactoryProvider` Pipe Pattern

Use an `@injectable()` class with `@factoryMethod()`, receiving context via `@ctx(PATH_PARAMS)` and target property name via `@input`:

```ts
import { injectable, factoryMethod, inject, ctx, input, AnyObj } from '@holu/core';
import { controller, route, PATH_PARAMS } from '@holu/rest';
import { BadRequestError } from './errors.js';

interface PostParams {
  postId: number;
  commentId: number;
}

@injectable()
export class ParseIntParamPipe {
  @factoryMethod()
  transform(@ctx(PATH_PARAMS) pathParams: AnyObj, @input paramName: string): number {
    const rawValue = pathParams?.[paramName];
    const parsedNumber = Number(rawValue);

    if (rawValue === undefined || Number.isNaN(parsedNumber)) {
      throw new BadRequestError(`Path parameter "${paramName}" must be a number. Received: "${rawValue}"`);
    }

    return parsedNumber;
  }
}

@controller()
export class ClassPipeController {
  @route('GET', 'posts/:postId/comments/:commentId')
  getComment(
    @inject<keyof PostParams>(ParseIntParamPipe, 'postId') postId: number,
    @inject<keyof PostParams>(ParseIntParamPipe, 'commentId') commentId: number,
  ) {
    return { postId, commentId };
  }
}

// Module registration (providersPerReq):
// { token: ParseIntParamPipe, useFactory: [ParseIntParamPipe, ParseIntParamPipe.prototype.transform] }
```

#### 2. `FunctionFactoryProvider` Pipe Pattern

Use a standalone function factory with `deps: [Context, input]`:

```ts
import { Context, inject, input } from '@holu/core';
import { controller, route, PATH_PARAMS } from '@holu/rest';
import { BadRequestError } from './errors.js';

interface ProductParams {
  categoryId: number;
  productId: number;
}

export function parseIntParamPipeFn(ctx: Context, paramName: string): number {
  const pathParams = ctx.get(PATH_PARAMS);
  const rawValue = pathParams?.[paramName];
  const parsedNumber = Number(rawValue);

  if (rawValue === undefined || Number.isNaN(parsedNumber)) {
    throw new BadRequestError(`Path parameter "${paramName}" must be a number. Received: "${rawValue}"`);
  }

  return parsedNumber;
}

@controller()
export class FunctionPipeController {
  @route('GET', 'categories/:categoryId/products/:productId')
  getProduct(
    @inject<keyof ProductParams>(parseIntParamPipeFn, 'categoryId') categoryId: number,
    @inject<keyof ProductParams>(parseIntParamPipeFn, 'productId') productId: number,
  ) {
    return { categoryId, productId };
  }
}

// Module registration (providersPerReq):
// { deps: [Context, input], useFactory: parseIntParamPipeFn }
```

## Part 2: Module References

### Full Module Metadata Shapes

`@holu/core` defines metadata via the `FeatureModuleOptions` class (passed to `rootModule` and `featureModule`):

```ts
// Source: @holu/core — decorators/module-decorator-options.ts
class FeatureModuleOptions<T extends AnyObj = AnyObj> {
  // ModRefId = StaticModule | DynamicModule
  imports?: (ModRefId | ForwardRefFn<StaticModule>)[];
  exports?: any[];
  providersPerApp?: ProviderBuilder | (Provider | ForwardRefFn<Provider>)[];
  providersPerMod?: ProviderBuilder | (Provider | ForwardRefFn<Provider>)[];
  providersPerRou?: ProviderBuilder | (Provider | ForwardRefFn<Provider>)[];
  providersPerReq?: ProviderBuilder | (Provider | ForwardRefFn<Provider>)[];
  extensions?: (ExtensionConfig | ExtensionClass)[];
  extensionsMeta?: T; // generic — any object shape, not Record<string, unknown>
  resolvedCollisionsPerMod?: [any, ModRefId | ForwardRefFn<StaticModule>][];
  resolvedCollisionsPerRou?: [any, ModRefId | ForwardRefFn<StaticModule>][];
  resolvedCollisionsPerReq?: [any, ModRefId | ForwardRefFn<StaticModule>][];
}
```

`rootModule` adds one more field:

```ts
resolvedCollisionsPerApp?: [any, ModRefId | ForwardRefFn<StaticModule>][];
```

`@holu/rest` and `@holu/trpc` extend the metadata with their own fields (`appends`, `controllers`) — these are **not** part of `FeatureModuleOptions`.

### `DynamicModule` Shape

`DynamicModule` is composed of interfaces in `@holu/core` (`DynamicModuleBase` and `DynamicModuleOptions`), and `DynamicModuleWithAspect` extends `DynamicModule`:

```ts
// Source: @holu/core — decorators/module-decorator-options.ts
interface DynamicModuleBase<M extends AnyObj = AnyObj> {
  id?: string; // optional module identity string for disambiguation
  module: StaticModule<M> | ForwardRefFn<StaticModule<M>>;
}

interface DynamicModuleOptions<E extends AnyObj = AnyObj> extends Partial<ProvidersByLevel> {
  // ProvidersByLevel provides: providersPerApp, providersPerMod, providersPerRou, providersPerReq
  exports?: any[];
  extensionsMeta?: E; // generic, not Record<string, unknown>
}

// The final exported type used in imports[]:
interface DynamicModule<M extends AnyObj = AnyObj> extends DynamicModuleBase<M>, DynamicModuleOptions {
  aspectOptions?: DynamicAspectOptionsMap; // present when used with aspect decorators
}
```

Note: there is **no** index signature (`[key: string]: unknown`) in `DynamicModule`. Extra properties are not generically allowed.

`path` is **not** a field of `DynamicModule` from `@holu/core`. When using `@holu/rest`, the object passed to `imports[]` is typed as `RestDynamicOptions`, which extends `DynamicModuleOptions` and adds route-specific fields:

```ts
// Source: @holu/rest — init/rest-aspect-raw-meta.ts
// RestDynamicOptions = RestDynamicPathOptions | RestDynamicAbsolutePathOptions
interface RestDynamicPathOptions extends BaseRestModuleOptions {
  path?: string;
  absolutePath?: never; // mutually exclusive with absolutePath
}
interface RestDynamicAbsolutePathOptions extends BaseRestModuleOptions {
  absolutePath?: string;
  path?: never; // mutually exclusive with path
}
interface BaseRestModuleOptions extends DynamicModuleOptions {
  exports?: any[];
  guards?: GuardItem[];
}
```

Use `path` for a relative route prefix, `absolutePath` for an absolute one. They cannot be set simultaneously.

When a module is imported with `DynamicModule`, the framework treats the object identity as the module's key. This is why re-exporting must use the **same object reference**.

### `getTokens()` Behavior

`getTokens(providers: Provider[]): Token[]` extracts the token from each provider in the array:

- `Class` → the class itself is the token
- `{ token, useValue | useClass | useFactory | useToken }` → extracts `token`
- Multi-providers (`{ token, useValue, multi: true }`) → token is included once

Use it whenever a provider array is constructed with object-form providers and you want to export all of them without manually listing tokens.

### Import Order vs. Collision Resolution

Import order does **not** determine which provider wins in a collision. Holu detects any ambiguity between two imported modules that export different providers for the same token and throws a collision error, regardless of import order. Always resolve collisions explicitly via `resolvedCollisionsPer*`.

### Route Prefix Composition

Path prefixes compose from outer to inner:

- Root module mounts a feature module at `api/v1` via `imports: [{ path: 'api/v1', module: ApiModule }]`
- `ApiModule` mounts `UsersModule` at `users` via `imports: [{ path: 'users', module: UsersModule }]`
- Controller route `GET :id` inside `UsersModule` → final URL: `GET /api/v1/users/:id`

The same composition applies for `appends`.

### Provider Scope Decision Guide

Use this decision tree when choosing provider scope:

1. Must be a single instance for the entire app lifetime? → `providersPerApp`
2. Must be a single instance per module (e.g., module-level config)? → `providersPerMod`
3. Must be a single instance per matched route (e.g., route-level state)? → `providersPerRou`
4. Must be fresh per HTTP request (e.g., request context, user session)? → `providersPerReq`

### Common Collision Error Message Pattern

```
Error: Collision was found in DataModule (providersPerMod) for token LogService.
This collision can be resolved in one of the following modules: AppModule...
```

- Identify the scope: `providersPerMod` → use `resolvedCollisionsPerMod`
- Identify the token: `LogService`
- Choose which module's provider to prefer: `[LogService, PreferredModule]`
- Add to the consuming module:
  ```ts
  resolvedCollisionsPerMod: [[LogService, PreferredModule]],
  ```

## Part 3: Aspect Decorators and ModuleAspectHandler

### Roles, Rules & Propagation

#### Roles of Aspect Decorators

An aspect decorator can serve three roles:

1. **Root Module Decorator** (e.g., `@restRootModule`): Declares a root module and extends the behavior/metadata of `@rootModule`. This decorator's transformer function returns an `ModuleAspectHandler` subclass instance with its `moduleRole` property set to `'root'`.
2. **Feature Module Decorator** (e.g., `@restModule`): Declares a feature module and extends the behavior/metadata of `@featureModule`. This decorator's transformer function returns an `ModuleAspectHandler` subclass instance with its `moduleRole` property set to `'feature'`.
3. **Modifier Decorator** (e.g., `@restAspect`): Modifies/extends an already declared root or feature module. This decorator's transformer function returns an `ModuleAspectHandler` subclass instance with its `moduleRole` property set to `undefined`. Several modifier decorators can be applied to a single class (stacked).

#### Decorator Usage Rules

- A class decorated with a root or feature module aspect decorator (role `'root'` or `'feature'`) **does not** require standard `@rootModule` or `@featureModule` decorators.
- A class decorated only with a modifier decorator (role `undefined`) **must** be accompanied by a module decorator (standard or aspect-based). Otherwise, a `MissingModuleDecorator` error is thrown.

#### Parent Module Aspects Propagation

- **Static feature modules:** If a child module is a static class decorated with `@featureModule` (meaning it has no custom aspect decorators of its own), it automatically inherits the parent module's module aspects (such as `restAspect` or `trpcAspect`) during scanning.
  - This inheritance automatically imports the parent module aspect's `hostModule` (e.g., `RestModule`) into the child module, ensuring that extensions (like `BodyParserExtension`) do not crash due to missing route/context providers.
  - **Exception:** This automatic propagation is **skipped** for external modules (meaning modules imported from `node_modules`) to avoid circular imports and missing dependency resolution errors in third-party or framework core modules (like `ContextModule`).
- **Dynamic feature modules:** Dynamic modules that do not have custom decorators can similarly inherit parent hooks when they are imported, automatically populating their initialization options and importing the corresponding host modules.

### Custom Decorator Creation Syntax

Use `Reflector.makeClassDecorator(transformAspectOptions, name, parentAspectDecorator)` to create an aspect decorator:

```ts
import {
  ModuleAspectHandler,
  ModuleAspectDecorator,
  Reflector,
  StaticAspectOptions,
  DynamicModuleOptions,
  BaseNormalizedModuleMeta,
  NormalizedModuleMeta,
  RootModuleOptions,
} from '@holu/core';
// ...

/**
 * An object with this type will be passed directly to the aspect decorator - @someAspect({ one: 1, two: 2 })
 */
interface MyStaticAspectOptions extends StaticAspectOptions<DynamicAspectOptions> {
  one?: number;
  two?: number;
}

/**
 * The methods of this class will normalize and validate the module metadata.
 */
class SomeModuleAspect extends ModuleAspectHandler<MyStaticAspectOptions> {
  // ...
}

/**
 * An object with this type will be passed in the module metadata as dynamic module.
 */
interface DynamicAspectOptions extends DynamicModuleOptions {
  path?: string;
  num?: number;
}

/**
 * Module aspects transform an object of MyStaticAspectOptions into an object of that type.
 */
interface MyNormalizedModuleMeta extends BaseNormalizedModuleMeta {
  normalizedModuleMeta: NormalizedModuleMeta;
  aspectDecoratorOptions: RootModuleOptions;
}

function transformAspectOptions(data?: MyStaticAspectOptions): ModuleAspectHandler<MyStaticAspectOptions> {
  const metadata = Object.assign({}, data);
  const moduleAspect = new SomeModuleAspect(metadata);
  moduleAspect.moduleRole = undefined;
  // OR moduleAspect.moduleRole = 'root';
  // OR moduleAspect.moduleRole = 'feature';
  return moduleAspect;
}

// Creating the aspect decorator
const someAspect: ModuleAspectDecorator<MyStaticAspectOptions, DynamicAspectOptions, MyNormalizedModuleMeta> =
  Reflector.makeClassDecorator(transformAspectOptions);

// Using aspect decorator
@someAspect({ one: 1, two: 2 })
export class SomeModule {}
```

### `ModuleAspectHandler` Methods and Properties

- `moduleRole?: 'root' | 'feature'`: Set to `'root'` or `'feature'` to make the aspect decorator act as a complete module decorator (meaning standard decorators are not required).
- `hostModule?: StaticModule`: If specified, the module class representing the host module will be automatically imported into any module class decorated with this aspect decorator.
- `hostAspectOptions?: T`: Raw options to pass as aspect parameters for the host module. When `hostModule` is normalizer-scanned, this allows attaching metadata to the host module class without directly decorating it (avoiding circular imports).
- `normalize(normalizedModuleMeta)`: Normalizes and validates raw options, returning a normalized metadata object that is saved in `normalizedModuleMeta.normalizedAspectMetaMap`.
- `getModulesToScan(meta)`: Returns an array of `ModRefId` modules that should also be scanned (e.g., appended modules in REST).
- `exportAppProviders(config)`: Invoked at bootstrap to collect and export application-level providers.
- `importModulesShallow(config)`: Invoked during shallow imports step to collect routes, paths, controllers, and guards.
- `importModulesDeep(config)`: Invoked during deep imports step to resolve provider dependencies.
- `getProvidersToOverride(meta)`: Returns array of arrays of providers that are overridable.

### Grouping Aspect Decorators via `decoratorId`

When creating a substitute decorator (with `'root'` or `'feature'` role) using `Reflector.makeClassDecorator()`, you **must** pass the base modifier decorator (e.g. `restAspect` or `someAspect`) as the third argument. This third argument serves as the `decoratorId`. It tells Holu that these decorators belong to the same group, enabling the framework to correctly collect, normalize, and associate metadata with the proper group context during initialization.

### Separation of Module Logic and Substitute Decorators via hostModule

Separating the decorator's metadata definition from the host feature module is necessary to avoid circular dependencies (decorating the host module with itself would create an import loop):

1. Create a standard feature module (e.g. `MyLibModule`) containing all necessary providers, middlewares, and extensions.
2. In your custom `ModuleAspectHandler` subclass, specify `override hostModule = MyLibModule`.
3. Create the base modifier decorator (e.g. `aspectMy`) which serves as the group parent decorator.
4. Create the transformer function that returns the hooks instance and sets `hooks.moduleRole = 'feature'` (or `'root'`).
5. Create the substitute custom decorator (e.g. `myFeatureModule`) using `Reflector.makeClassDecorator()`, passing the transformer as the first argument, its name as the second, and the base modifier decorator (`aspectMy`) as the third argument (the parent).
6. When developers apply this substitute decorator (e.g. `@myFeatureModule`), the framework recognizes it as a module decorator (requiring only one decorator on the class instead of two) and automatically imports `MyLibModule`.

Example:

```ts
import { featureModule, ModuleAspectHandler, Reflector } from '@holu/core';

// 1. Standard module containing actual logic/providers
@featureModule({
  providersPerReq: [MyService],
  exports: [MyService],
})
export class MyLibModule {}

// 2. Custom hooks setting hostModule
class MyModuleAspect extends ModuleAspectHandler {
  override hostModule = MyLibModule;
}

// 3. Creating the base modifier decorator (serves as the group parent)
export const aspectMy = Reflector.makeClassDecorator((data) => new MyModuleAspect(data), 'aspectMy');

// 4. Creating the transformer that sets moduleRole = 'feature'
function transformFeatureMeta(data?: any) {
  const hooks = new MyModuleAspect(data);
  hooks.moduleRole = 'feature'; // Makes it a substitute module decorator
  return hooks;
}

// 5. Creating the substitute decorator, passing aspectMy as the 3rd argument
export const myFeatureModule = Reflector.makeClassDecorator(transformFeatureMeta, 'myFeatureModule', aspectMy);

// 6. Using only one decorator on the class (automatically imports MyLibModule)
@myFeatureModule()
export class MyFeatureModule {}
```

### Parameter Merging and plain Dynamic Module

When importing a dynamic module in the context of an aspect decorator:

1. The dynamic module's custom options (like `path` or `guards`) are merged into the `dynamicModule.aspectOptions` Map under the aspect decorator's token.
2. If `Module1` itself is a plain `@featureModule` (not decorated with `@restAspect` or `@restModule`), the framework automatically retrieves the default hook class for the decorator from the application's register, clones it, registers it in the module's `moduleAspectMap` list, and calls `normalize()`.
3. This ensures that custom options (such as REST routing prefixes and route guards) are correctly applied to plain feature modules during import.

## Part 4: Metadata Reflector References

### Technical Types Reference

These are core interfaces and classes used by `Reflector` (primarily imported from `@holu/core/di` or `@holu/core`):

#### 1. `DecoratorMeta<Value>`

Stores the identifier of a decorator and the transformed value it returned:

```ts
class DecoratorMeta<Value = any> {
  constructor(
    public decorator: AnyFn, // The decorator factory function
    public value: Value, // The output returned by the decorator transformer
    public decoratorId?: AnyFn, // Optional group/type identifier for typeguards
    public declaredInDir?: string, // Directory in which the decorator was evaluated (class-level only)
  ) {}
}
```

#### 2. `MergedClassMeta<DecorValue, Proto>`

The return type of `Reflector.collectMeta(Cls)`. It maps every decorated property of the prototype to its metadata and provides constructor metadata:

```ts
type MergedClassMeta<DecorValue = any, Proto extends object = object> = {
  [P in keyof Proto]: MergedClassPropMeta<DecorValue>;
} & {
  constructor: MergedClassPropMeta<DecorValue>;
} & {
  [Symbol.iterator]: () => Generator<string | symbol>; // yields decorated property names
};
```

#### 3. `MergedClassPropMeta<DecorValue>`

Contains metadata about a constructor, property, or method. Supports inheritance hierarchies by storing metadata chains:

```ts
class MergedClassPropMeta<DecorValue = any> extends ClassPropMeta<DecorValue> {
  constructor(
    type: Class, // Property type or class constructor type
    decorators: DecoratorMeta<DecorValue>[], // Flattened array of decorators (merged from chain)
    params: (ParameterMeta | null)[], // Constructor/Method parameters metadata (merged)
    public decoratorChain: Map<Class, DecoratorMeta<DecorValue>[]>, // Key: Class, Value: class-specific decorators
    public paramChain: Map<Class, (ParameterMeta | null)[]>, // Key: Class, Value: class-specific parameters
  ) {}
}
```

#### 4. Parameter Metadata Types

```ts
type ParameterItem<Value = any> = DecoratorMeta<Value> | InjectionToken<any> | Class;
type ParameterMeta<Value = any> = [Class, ...ParameterItem<Value>[]] | [...ParameterItem<Value>[]] | [];
```

### Creating Custom Decorators

Use the static factory methods of `Reflector` to create decorators.

#### 1. Class Decorator

```ts
import { Reflector } from '@holu/core/di';

// Without transformer (returns arguments as an array)
const classLevelA = Reflector.makeClassDecorator();

// With transformer (transforms arguments into a structured object)
interface MyConfig {
  name: string;
  debug?: boolean;
}
const classLevelB = Reflector.makeClassDecorator((config: MyConfig) => config);

@classLevelB({ name: 'users-service', debug: true })
class UsersService {}
```

##### Class Decorator Arguments (Reflector.makeClassDecorator)

`Reflector.makeClassDecorator()` accepts up to three arguments:

1. **`transformer`**: (Optional) A function that transforms the decorator arguments into a structured object (e.g., `(config) => config`).
2. **`name`**: (Optional) A string containing the name of the decorator.
3. **`decoratorId`**: (Optional) An identifier (typically another decorator factory function) to group related class decorators together. This is a general feature of `Reflector.makeClassDecorator()` that allows different class decorators to share a common ID, facilitating their collection and inspection under the same group. For example, in aspect-decorators, the substitute decorators (like `restModule`) pass the base modifier decorator (like `restAspect`) as the `decoratorId` so Holu knows they belong to the same group.

_Note: Class decorator factories capture the directory where they are executed, which is used by Holu's module discovery to resolve relative paths._

#### 2. Property / Method Decorator

```ts
import { Reflector } from '@holu/core/di';

const propertyLevel = Reflector.makePropDecorator((role: string) => ({ role }));

class Controller {
  @propertyLevel('admin')
  secureData() {}
}
```

#### 3. Parameter Decorator

```ts
import { Reflector } from '@holu/core/di';

const paramLevel = Reflector.makeParamDecorator((token: any) => ({ token }));

class Handler {
  constructor(@paramLevel('CONFIG_TOKEN') config: any) {}
}
```

#### 4. Custom Decorator with Complex Signatures

When writing a decorator that accepts multiple argument lists, use an interface to define its call signatures and let `Reflector.make*Decorator` transform the arguments:

```ts
import { Reflector } from '@holu/core/di';

interface RouteOptions {
  method: string;
  path: string;
}

interface RouteDecorator {
  (path: string): MethodDecorator;
  (method: string, path: string): MethodDecorator;
}

const route: RouteDecorator = Reflector.makePropDecorator((arg1: string, arg2?: string) => {
  if (arg2) {
    return { method: arg1, path: arg2 } satisfies RouteOptions;
  }
  return { method: 'GET', path: arg1 } satisfies RouteOptions;
}, 'route');
```

### Collecting Metadata

Retrieve decorator metadata using `Reflector.collectMeta()` or `Reflector.getClassLevelMeta()`.

#### Reading Class-Level Decorators

To get decorators directly attached to a class:

```ts
import { Reflector } from '@holu/core/di';
import { controller } from '@holu/rest';

// Returns all decorators on the class
const decorators = Reflector.getClassLevelMeta(MyController);

// Filter by a type guard
const controllerDecorators = Reflector.getClassLevelMeta(MyController, (decor) => decor.decorator === controller);
if (controllerDecorators) {
  const metadataValue = controllerDecorators[0].value; // contains the decorator options
}
```

#### Reading Full Class Metadata Iterator

`Reflector.collectMeta(Cls)` returns a `MergedClassMeta` object which can be iterated to get decorated properties.

```ts
import { Reflector } from '@holu/core/di';

const metadata = Reflector.collectMeta(MyService);
if (metadata) {
  // Read constructor metadata
  const ctorMeta = metadata.constructor;
  console.log(ctorMeta.decorators); // Array of DecoratorMeta on the constructor
  console.log(ctorMeta.params); // Array of constructor parameters and their decorators

  // Iterate over decorated property/method keys
  for (const propName of metadata) {
    const propMeta = metadata[propName];
    console.log(`Property ${String(propName)} has type:`, propMeta.type);
    console.log(`Decorators:`, propMeta.decorators);
  }
}
```

#### Class Decorator with decoratorId (Grouping Decorators)

When creating multiple class decorators that belong to a single logical group, you can pass a reference to the base decorator as the third argument (`decoratorId`). This makes it easy to filter/inspect them collectively at runtime:

```ts
import { DecoratorMeta, Reflector } from '@holu/core/di';

// 1. Create a base decorator (acts as the Group ID)
export const base = Reflector.makeClassDecorator((data?: any) => data, 'base');

// 2. Create related decorators, passing `base` as the third argument (decoratorId)
export const decorator1 = Reflector.makeClassDecorator((data?: any) => data, 'decorator1', base);
export const decorator2 = Reflector.makeClassDecorator((data?: any) => data, 'decorator2', base);

// 3. Apply the decorators
@decorator1({ one: 1 })
class SomeModule {}

// 4. Retrieve and inspect metadata by grouping ID
const decorators = Reflector.getClassLevelMeta(
  SomeModule,
  (decor): decor is DecoratorMeta<{ one: number }> => decor.decoratorId === base,
);
if (decorators && decorators.length > 0) {
  console.log(decorators[0].value); // { one: 1 }
}
```

### Inheritance and Chains

`Reflector` resolves metadata throughout the entire class inheritance hierarchy. If a child class extends a parent class and overrides properties or constructor parameters, `Reflector` tracks this in the `decoratorChain` and `paramChain` fields of `MergedClassPropMeta`.

```ts
import { Reflector } from '@holu/core/di';

const customDecor = Reflector.makeClassDecorator((val?: string) => val);

@customDecor('parent-meta')
class Parent {
  constructor(a: string) {}
}

@customDecor('child-meta')
class Child extends Parent {
  constructor(a: string, b: number) {
    super(a);
  }
}

const childMeta = Reflector.collectMeta(Child);
if (childMeta) {
  const ctor = childMeta.constructor;

  // Flattened decorators/params for the child
  console.log(ctor.decorators); // Contains child's decorators

  // Inheritance chains
  console.log(ctor.decoratorChain); // Map<Class, DecoratorMeta[]> containing Parent & Child decorators
  console.log(ctor.paramChain); // Map<Class, ParameterMeta[]> containing Parent & Child parameter metadata
}
```

#### Detailed Example: Parsing `collectMeta` Output

When `Reflector.collectMeta(Child)` is executed, the following data structure is generated:

```ts
import { Reflector } from '@holu/core/di';

class ServiceA {}
class ServiceB {}

class Parent {
  constructor(a: ServiceA) {}
}

class Child extends Parent {
  constructor(a: ServiceA, b: ServiceB) {
    super(a);
  }
}

const metadata = Reflector.collectMeta(Child);
/*
metadata structure:
{
  constructor: MergedClassPropMeta {
    type: Child,
    decorators: [],
    params: [
      [ServiceA], // Parameter 0: Type is ServiceA, no decorators
      [ServiceB]  // Parameter 1: Type is ServiceB, no decorators
    ],
    decoratorChain: Map {
      [class Parent] => [],
      [class Child] => []
    },
    paramChain: Map {
      [class Parent] => [ [ServiceA] ],
      [class Child] => [ [ServiceA], [ServiceB] ]
    }
  }
}
*/
```

### Programmatic Metadata Writing (Experimental & Non-Public)

Holu provides experimental helper methods to attach metadata programmatically without using decorators directly (useful for code generators, dynamic routing setups, or testing environments).

> [!IMPORTANT]
> These methods (`setClassMeta`, `setPropertyMeta`, `setParameterMeta`) are **`protected static`** in the `Reflector` class. They are not part of the public API.
> To use them, you must subclass `Reflector` to expose them publicly or access them via a type assertion/cast:

```ts
import { Reflector } from '@holu/core/di';
import { controller, route } from '@holu/rest';

// Exposing protected methods via subclassing:
class CustomReflector extends Reflector {
  static override setClassMeta(...args: Parameters<typeof Reflector.setClassMeta>) {
    return super.setClassMeta(...args);
  }
  static override setPropertyMeta(...args: Parameters<typeof Reflector.setPropertyMeta>) {
    return super.setPropertyMeta(...args);
  }
  static override setParameterMeta(...args: any[]) {
    return super.setParameterMeta(...args);
  }
}

class CustomController {
  tellHello() {
    return 'Hello!';
  }
}

// 1. Programmatically write class-level decorator
CustomReflector.setClassMeta(CustomController, controller, { routePrefix: 'custom' });

// 2. Programmatically write method-level decorator and compile design:type
CustomReflector.setPropertyMeta(
  CustomController,
  'tellHello',
  Function, // property type
  route,
  'GET',
  'hello', // decorator arguments
);
```

## Part 5: Application Lifecycle & Shutdown References

### `BaseApplication` Shutdown Sequence

When `app.close(signal)` is called (manually or via intercepted process signal), `BaseApplication` executes the following sequence:

```
[Signal / app.close()]
          │
          ▼
1. isShuttingDown = true & cleanShutdownListeners()
          │
          ▼
2. Gather instantiated singletons from injectorPerApp & module injectors
          │
          ▼
3. Run BeforeShutdown hooks (runShutdownHooks: 'beforeShutdown')
          │
          ▼
4. Execute customShutdown(signal) (overridden by RestApplication, custom apps)
          │
          ▼
5. Run OnShutdown hooks (runShutdownHooks: 'onShutdown')
          │
          ▼
6. Flush logs & process.exit(0) (if signal present)
```

### Service Shutdown Hooks Example (`BeforeShutdown` & `OnShutdown`)

```ts
import { BeforeShutdown, OnShutdown, injectable } from '@holu/core';

@injectable()
export class DatabaseService implements BeforeShutdown, OnShutdown {
  beforeShutdown(signal?: string) {
    console.log(`Received signal ${signal}. Stopping background jobs...`);
  }

  async onShutdown(signal?: string) {
    console.log('Closing database connection pool...');
    await this.pool.end();
  }
}
```

### Overriding `customShutdown` in Subclasses

`customShutdown(signal?: string)` is a protected extension method in `BaseApplication` designed for package-level application classes (`RestApplication`, gRPC/WebSocket wrappers) to perform custom resource cleanup between `beforeShutdown` and `onShutdown` hooks.

#### Implementation in `@holu/rest`:

```ts
export class RestApplication extends BaseApplication {
  protected override async customShutdown(signal?: string): Promise<void> {
    if (this.server) {
      await new Promise<void>((resolve, reject) => {
        // Stops receiving new connections and drains active sockets
        this.server.close((err) => (err ? reject(err) : resolve()));
      });
    }
  }
}
```

#### Custom Application Subclass Example:

```ts
import { BaseApplication } from '@holu/core';

export class CustomApp extends BaseApplication {
  protected override async customShutdown(signal?: string): Promise<void> {
    // Custom framework/package level cleanup (e.g. stopping gRPC server, flushing telemetry)
    await this.myCustomServer.stop();
  }
}
```

### Active Instance Discovery Mechanics

`getActiveInstances()` scans:

1. `this.injectorPerApp` (Application scope injector).
2. All module-level injectors from `this.moduleRegistry.getInjectorsPerMod().values()`.

It retrieves instantiated singleton instances via `injector.getInstances()`. Services registered in `providersPerReq` or `providersPerRou` or singletons that were never instantiated during the app runtime are excluded.

### Error Handling in Shutdown Hooks

`runShutdownHooks` executes all matching hooks concurrently using `Promise.allSettled()`:

```ts
protected async runShutdownHooks(
  instances: any[],
  hookName: 'beforeShutdown' | 'onShutdown',
  signal?: string,
): Promise<void> {
  const hooks = instances
    .filter((instance) => instance && typeof instance[hookName] == 'function')
    .map(async (instance) => instance[hookName](signal));

  const results = await Promise.allSettled(hooks);
  results.forEach((result) => {
    if (result.status == 'rejected') {
      this.log.shutdownError(this, hookName, result.reason);
    }
  });
}
```

Errors thrown in `beforeShutdown` or `onShutdown` methods do not interrupt other services' shutdown hooks; they are logged via `this.log.shutdownError()`.

## Part 6: LogMediator & Log Buffering

`LogMediator` is the foundational abstraction in Holu for structured, strongly-typed, and context-aware logging. It separates log message formatting from application/extension logic and enables log buffering during application startup.

### 1. Strongly-Typed Module Log Mediators

Rather than injecting `Logger` directly and scattering string templates across extensions or services, Holu modules create a dedicated subclass of `LogMediator`.

#### Creating a Custom Log Mediator

1. Create a class extending `LogMediator` decorated with `@injectable()`.
2. Define typed methods for each log message. Each method typically accepts `self: object` (to extract `self.constructor.name`) and any relevant contextual parameters.
3. Inside each method, format the message string and call `this.setLog(inputLogLevel, msg)`.

```ts
import { injectable, LogMediator, InputLogLevel } from '@holu/core';

@injectable()
export class OpenapiLogMediator extends LogMediator {
  /**
   * `trace: ${className}: waiting for last module with ${className}.`
   */
  dataAccumulation(self: object) {
    const className = self.constructor.name;
    this.setLog('trace', `${className}: return false, waiting for last module with ${className}.`);
  }

  /**
   * `trace: ${className}: config file (with XOasObject type) not found, applying default.`
   */
  oasObjectNotFound(self: object) {
    const className = self.constructor.name;
    this.setLog('trace', `${className}: config file (with XOasObject type) not found, applying default.`);
  }

  /**
   * `warn: ${className}: ${msg}`
   */
  customWarning(self: object, msg: string) {
    const className = self.constructor.name;
    this.setLog('warn', `${className}: ${msg}`);
  }
}
```

#### Registering & Injecting the Log Mediator

Register the log mediator in the module's `providersPerMod` (or `providersPerApp`) and inject it into extensions, controllers, or services:

```ts
// In module definition:
@featureModule({
  providersPerMod: [OpenapiLogMediator],
})
export class OpenapiModule {}

// In extension or service:
@injectable()
export class OpenapiCompilerExtension implements Extension<void> {
  constructor(protected log: OpenapiLogMediator) {}

  async stage1(): Promise<void> {
    this.log.dataAccumulation(this);
    // ...
  }
}
```

### 2. Log Buffering Mechanics (`bufferLogs` & `buffer`)

During application bootstrap, extensions and modules write logs **before** the user's `OutputLogLevel` or custom `LoggerConfig` may be fully resolved. To handle this cleanly, `LogMediator` provides static log buffering capabilities.

#### Static Buffer Properties

```ts
export abstract class LogMediator {
  static bufferLogs?: boolean = false;
  static buffer: LogEntry[] = [];
}
```

- **`LogMediator.bufferLogs`**: When set to `true` (typically by `AppInitializer` / `RestApplication.bootstrap(AppModule, { bufferLogs: true })`), `this.setLog()` does **not** write directly to the logger.
- **`LogMediator.buffer`**: Instead, `this.setLog()` pushes structured `LogEntry` items into this shared static array:

```ts
interface LogEntry {
  isExternal: boolean;
  showExternalLogs: boolean;
  moduleName: string;
  inputLogLevel: InputLogLevel;
  outputLogLevel: OutputLogLevel;
  date: Date;
  msg: string;
}
```

#### Automatic Formatting in `setLog`

When `bufferLogs` is `false`, `setLog()` automatically formats log output using module metadata:

```ts
this.logger.log(inputLogLevel, `[${this.moduleInfo.moduleName}]: ${msg}`);
```

#### Flushing Buffered Logs (`flush()`)

Once application providers and extensions are initialized and the final `OutputLogLevel` is established, `systemLogMediator.flush()` is called:

```ts
// In SystemLogMediator:
flush() {
  const { buffer } = LogMediator;
  this.writeLogs(buffer);
  buffer.splice(0); // Clears the static buffer
  LogMediator.hasDiffLogLevels = false;
}
```

`writeLogs()` iterates over all buffered `LogEntry` objects, updates their `outputLogLevel` to the resolved log level, filters out external module logs if `showExternalLogs === false`, and writes each entry using a child logger instance.

### 3. Overriding System Log Messages

Framework-level log messages emitted during bootstrap are defined in `SystemLogMediator` (which extends `LogMediator`). Applications can customize or translate framework log messages by extending `SystemLogMediator` and registering the subclass as a provider for `SystemLogMediator`:

```ts
@injectable()
export class CustomSystemLogMediator extends SystemLogMediator {
  override startingHolu(self: object) {
    const className = self.constructor.name;
    this.setLog('debug', `${className}: Запуск застосунку Holu...`);
  }
}

// In AppModule:
@rootModule({
  providersPerApp: [{ token: SystemLogMediator, useClass: CustomSystemLogMediator }],
})
export class AppModule {}
```
