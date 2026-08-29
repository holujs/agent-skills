---
name: holu-project-setup
description: Guidance on using Holu CLI (holu) for project bootstrapping and watch-running (start). Includes templates (rest, trpc), CLI auto-detection, minimal single-file applications, and AI agent usage guidelines.
---

# Holu Project Setup & CLI

The Holu framework provides the `@holu/cli` (binary name: `holu`) to bootstrap new projects and run them with efficient incremental TypeScript compilation.

## `holu new` - Scaffolding Projects

Create a new project using starter templates:

```bash
npx @holu/cli new my-project
```

Options:
- `-t, --template <name>`: Template to use (default: `rest`). Available:
  - `rest`: Standard REST API
  - `rest-monorepo`: REST API with Yarn workspaces
  - `trpc-monorepo`: tRPC with Yarn workspaces
- `-m, --package-manager <name>`: `npm`, `yarn`, or `pnpm` (default: `npm`)
- `--skip-install`: Skip dependency installation
- `--skip-git`: Skip git initialization

## `holu start` - Development Server

The `start` command compiles the project (via TypeScript incremental builds) and restarts the Node process on changes.

```bash
yarn holu start
```

**Key Options & Auto-Detection:**
- **Config Resolution**: It auto-detects `tsconfig.build.json` in the root (or falls back to `tsconfig.json`). You can pass a directory via `-p apps/backend`.
- **`.env` Loading**: It automatically loads `.env` if it exists. Override with `--env-file <paths...>` (accepts one or more paths).
- `-e, --exec <binary>`: Execution binary (default: `node`)
- `-d, --debug [hostport]`: Run node with `--inspect`
- `--entry-file <file>`: Compiled entry file to run (relative to project root), overriding auto-detection
- `--preserve-watch-output`: Do not clear the terminal screen between compilations
- `--verbose`: Shows compilation time and progress
- `--restart-delay <ms>`: Delay before restarting process (default: 300)
- `--watch-assets <globs...>`: Watch non-TS files
- `-- [appArgs]`: Forward arguments to the child process (e.g., `holu start -- --port 8080`)

> [!WARNING]
> **Rule for AI Agents**: DO NOT run `holu start` in the background during coding. It is an interactive watch process meant for users. Only run one-off build/test commands.

## Minimal REST Application

A single-file REST application structure using `@holu/rest`.

```ts
import { controller, route, restRootModule, RestApplication, RequestContext } from '@holu/rest';

@controller()
class AppController {
  @route('GET', 'hello')
  getHello(ctx: RequestContext) {
    return 'Hello World!';
  }
}

@restRootModule({
  controllers: [AppController]
})
class AppModule {}

const app = await RestApplication.create(AppModule);
app.server.listen(3000, '0.0.0.0');
```
*Requires Node.js >= 24.0.0 and `experimentalDecorators: true`, `emitDecoratorMetadata: true` in tsconfig.*

## Minimal tRPC Application

A single-file tRPC application structure using `@holu/trpc`. Notice the use of `TrpcRouteService` to build the procedure.

```ts
import { trpcController, trpcRoute, trpcRootModule, TrpcApplication, TrpcRouteService } from '@holu/trpc';
import { z } from 'zod';

@trpcController()
class ExampleRouter {
  @trpcRoute()
  hello(routeService: TrpcRouteService) {
    return routeService.procedure.query(() => 'Hello from tRPC!');
  }

  @trpcRoute()
  createUser(routeService: TrpcRouteService) {
    return routeService.procedure
      .input(z.object({ name: z.string() }))
      .mutation(({ input }) => ({ id: 1, name: input.name }));
  }
}

@trpcRootModule({
  controllers: [ExampleRouter]
})
class AppModule {}

const app = await TrpcApplication.create(AppModule);
app.server.listen(3000, '0.0.0.0');
```
