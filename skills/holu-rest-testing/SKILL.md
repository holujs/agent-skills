---
name: holu-rest-testing
description: Testing utilities for Holu REST applications (@holu/rest). Covers isolated DI container setup, mocking providers via TestRestApplication and TestRestPlugin, HTTP request testing with SuperTest, and Unit Testing via Injector.
---

# Testing Holu Applications

Holu provides robust testing utilities to bootstrap applications in test environments and easily mock dependencies. The primary package for REST testing is `@holu/rest-testing`.

## 1. End-to-End (E2E) Testing with SuperTest

To test REST applications, you use `TestRestApplication` with the `TestRestPlugin` from `@holu/rest-testing`.

```ts
import request from 'supertest';
import type { HttpServer } from '@holu/rest';
import { TestRestApplication, TestRestPlugin } from '@holu/rest-testing';

import { AppModule } from './app.module.js';

describe('AppController (e2e)', () => {
  let app: TestRestApplication;
  let server: HttpServer;
  let testAgent: ReturnType<typeof request>;

  beforeAll(async () => {
    // Bootstrap the test application and get the HTTP server
    app = TestRestApplication.createTestApp(AppModule, { path: 'api', logLevel: 'off' })
      .$use(TestRestPlugin);

    server = await app.getServer(); // getServer() is async and must be awaited
    testAgent = request(server);
  });

  afterAll(async () => {
    // TestRestApplication inherits close() from BaseApplication — use it for graceful shutdown
    await app?.close();
  });

  it('/api/hello (GET)', async () => {
    const response = await testAgent.get('/hello').expect(200);
    expect(response.text).toBe('Hello World!');
  });
});
```

## 2. Mocking Dependencies

Holu provides methods on `TestRestApplication` to override providers using a flat array of Providers or the `ProviderBuilder` fluent API.

### Overriding Module/App Providers

You can override providers (like Database Services) in the module registry using `overrideModuleMeta()`. It accepts a flat array of providers or a `ProviderBuilder`.

```ts
import { TestRestApplication } from '@holu/rest-testing';
import { ProviderBuilder } from '@holu/core';

import { AppModule } from './app.module.js';
import { DatabaseService } from './db.service.js';

const mockDatabaseService = {
  findUsers: jest.fn().mockResolvedValue([{ id: 1, name: 'Test User' }]),
};

const server = await TestRestApplication.createTestApp(AppModule)
  .overrideModuleMeta([
    { token: DatabaseService, useValue: mockDatabaseService }
  ])
  // Or using ProviderBuilder:
  // .overrideModuleMeta(new ProviderBuilder().useValue(DatabaseService, mockDatabaseService))
  .getServer();
```

### Overriding Route/Request Scoped Providers

For providers injected per-request or per-route inside a controller (e.g., via `@restModule` or `@restRootModule` extensions), use `overrideExtensionRestMeta()` from `TestRestPlugin`.

```ts
import { TestRestApplication, TestRestPlugin } from '@holu/rest-testing';
import { AppModule } from './app.module.js';
import { ExternalApiService } from './external.service.js';

const mockExternalApi = {
  fetchData: jest.fn().mockResolvedValue('mock data'),
};

const server = await TestRestApplication.createTestApp(AppModule)
  .$use(TestRestPlugin)
  // Modifies route/request providers across the REST application
  .overrideExtensionRestMeta([
    { token: ExternalApiService, useValue: mockExternalApi }
  ])
  .getServer();
```

### Marking Modules as External

To isolate third-party providers (prevent application-exported `providersPerApp` from leaking into external modules), use `markModuleAsExternal()`. Note: this sets `isExternal = true` but does *not* completely exclude the module from the test environment.

```ts
const app = TestRestApplication.createTestApp(AppModule)
  .markModuleAsExternal(SomeThirdPartyModule);
```

### Adding Providers to Specific Modules

Use `addProvidersToModule()` when you need to **add** new providers to a specific module rather than globally overriding existing ones. Pass the module reference and an array of providers:

```ts
const server = await TestRestApplication.createTestApp(AppModule)
  .addProvidersToModule(UsersModule, [
    { token: CacheService, useValue: mockCacheService }
  ])
  .getServer();
```

> [!TIP]
> For overriding extension groups beyond REST routes (e.g., custom extension groups), use the generic `overrideExtensionMeta()` method instead of `overrideExtensionRestMeta()`.

## 3. Unit Testing

For isolated unit testing of services, guards, interceptors, and controllers without booting the whole app, use `Injector.resolveAndCreate` from `@holu/core`.

```ts
import { jest } from '@jest/globals';
import { Injector } from '@holu/core';
import { Logger } from '@holu/core';

import { MyService } from './my.service.js';

describe('MyService (unit)', () => {
  let service: MyService;
  let mockLogger: any;

  beforeEach(() => {
    mockLogger = { info: jest.fn() };
    const injector = Injector.resolveAndCreate([
      MyService,
      { token: Logger, useValue: mockLogger },
    ]);
    service = injector.get(MyService);
  });

  it('should log on init', () => {
    service.doSomething();
    expect(mockLogger.info).toHaveBeenCalled();
  });
});
```
