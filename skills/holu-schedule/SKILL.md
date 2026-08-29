---
name: holu-schedule
description: 'Scheduling tasks in Holu with @holu/schedule: @cron / @interval / @timeout decorators, CronExpression enum, CronOptions, SchedulerRegistry (query/start/stop/delete tasks at runtime), ScheduleModule setup, provider scope constraints (only providersPerMod / providersPerApp), graceful shutdown behavior, and dynamic task management patterns.'
---

# Holu Schedule

`@holu/schedule` provides decorator-based task scheduling — cron jobs, periodic intervals, and one-shot timeouts — for Holu applications. Scheduling is implemented as a Holu extension (`ScheduleExtension`) and requires no manual wiring beyond importing `ScheduleModule`.

## Installation

```bash
yarn add @holu/schedule
```

## Module Setup

Import `ScheduleModule` in any root or feature module. Register task classes in `providersPerMod` (or `providersPerApp`) of that same module — **not** in `providersPerRou` or `providersPerReq`.

```ts
import { rootModule } from '@holu/core';
import { ScheduleModule } from '@holu/schedule';

import { ReportTasks } from './report-tasks.service.js';
import { CleanupTasks } from './cleanup-tasks.service.js';

@rootModule({
  imports: [ScheduleModule],
  providersPerMod: [ReportTasks, CleanupTasks],
})
export class AppModule {}
```

> [!IMPORTANT]
> Task classes **must** be registered in `providersPerMod` or `providersPerApp`. Registration in `providersPerRou` or `providersPerReq` is explicitly forbidden — the extension will emit a warning and skip those providers. See [Provider Scope Constraints](#provider-scope-constraints) for details.

## Declaring Tasks

Create an `@injectable()` service and decorate its methods with `@cron`, `@interval`, or `@timeout`. Each decorator attaches metadata that the `ScheduleExtension` reads during bootstrap.

```ts
import { injectable, Logger } from '@holu/core';
import { cron, interval, timeout, CronExpression } from '@holu/schedule';

@injectable()
export class ReportTasks {
  constructor(private logger: Logger) {}

  @cron(CronExpression.EVERY_DAY_AT_MIDNIGHT, { name: 'daily-report' })
  generateDailyReport(): void {
    this.logger.log('info', 'Generating daily report...');
  }

  @interval('cache-refresh', 30_000)
  refreshCache(): void {
    this.logger.log('info', 'Refreshing cache...');
  }

  @timeout('startup-warmup', 5_000)
  warmUp(): void {
    this.logger.log('info', 'Post-startup warm-up complete.');
  }
}
```

## Decorator Reference

### `@cron(cronTime, options?)`

Runs the method on a cron schedule. Uses the [`cron`](https://www.npmjs.com/package/cron) package under the hood.

| Parameter  | Type                         | Description                                         |
| ---------- | ---------------------------- | --------------------------------------------------- |
| `cronTime` | `string \| Date \| DateTime` | Standard 5- or 6-field cron expression, or a `Date` |
| `options`  | `CronOptions` (optional)     | See table below                                     |

#### `CronOptions`

| Property            | Type      | Default | Description                                                                             |
| ------------------- | --------- | ------- | --------------------------------------------------------------------------------------- |
| `name`              | `string`  | uuid    | Unique name. Required to retrieve the job via `SchedulerRegistry.getCronJob(name)`.     |
| `timeZone`          | `string`  | —       | IANA timezone string (e.g., `'America/New_York'`). Mutually exclusive with `utcOffset`. |
| `utcOffset`         | `number`  | —       | UTC offset in minutes. Alternative to `timeZone`.                                       |
| `unrefTimeout`      | `boolean` | `false` | When `true`, the underlying timeout is unreffed so the process can exit naturally.      |
| `waitForCompletion` | `boolean` | `false` | When `true`, prevents overlapping executions — waits for the previous tick to finish.   |
| `disabled`          | `boolean` | `false` | When `true`, the job is registered but **not started** automatically.                   |
| `threshold`         | `number`  | `250`   | Millisecond threshold to skip missed ticks on slow/busy hardware.                       |
| `initialDelay`      | `number`  | —       | Delay in milliseconds before the cron job first starts after bootstrap.                 |

> [!WARNING]
> `waitForCompletion` and `threshold` are declared in `CronOptions` but are **not currently passed through** to the underlying `CronJob` by `SchedulerOrchestrator`. If you rely on these options, be aware they will have no effect at runtime until this is fixed in the framework.

**6-field cron syntax** (seconds-level precision):

```
 ┌────────────── second (0-59)   [optional]
 │  ┌─────────── minute (0-59)
 │  │  ┌──────── hour (0-23)
 │  │  │  ┌───── day of month (1-31)
 │  │  │  │  ┌── month (1-12)
 │  │  │  │  │  ┌─ day of week (0-7, 0 and 7 = Sunday)
 *  *  *  *  *  *
```

```ts
@cron('*/30 * * * * *', { name: 'every-30-seconds' }) // every 30 seconds (6 fields)
@cron('0 9 * * 1-5',    { name: 'weekday-9am' })       // Mon-Fri at 09:00 (5 fields)
@cron(new Date('2027-01-01T00:00:00Z'), { name: 'new-year' })
```

### `@interval(nameOrTimeout: string | number, timeout?: number)`

Runs the method repeatedly every `timeout` milliseconds via `setInterval`. If only `nameOrTimeout` is provided as a number, it acts as the timeout (anonymous task).

```ts
// Signature overloads:
@interval(timeout: number)
@interval(name: string, timeout: number)

// Usage examples:
@interval(3000)                  // anonymous, fires every 3 s
@interval('cache-refresh', 3000) // named, fires every 3 s
```

- When no name is provided, the interval is tracked internally with a random UUID and **cannot** be retrieved by name via `SchedulerRegistry`.
- Named intervals can be stopped at runtime with `SchedulerRegistry.deleteInterval(name)`.

### `@timeout(nameOrTimeout: string | number, timeout?: number)`

Runs the method **once** after `timeout` milliseconds via `setTimeout`. If only `nameOrTimeout` is provided as a number, it acts as the timeout (anonymous task).

```ts
// Signature overloads:
@timeout(timeout: number)
@timeout(name: string, timeout: number)

// Usage examples:
@timeout(5_000)                   // anonymous, fires once after 5 s
@timeout('startup-warmup', 5_000) // named
```

- Named timeouts can be cancelled at runtime with `SchedulerRegistry.deleteTimeout(name)`.

## `CronExpression` Enum

`CronExpression` provides a curated set of named expressions to avoid hard-coding cron strings:

```ts
import { CronExpression } from '@holu/schedule';

@cron(CronExpression.EVERY_5_SECONDS)
@cron(CronExpression.EVERY_HOUR)
@cron(CronExpression.EVERY_DAY_AT_MIDNIGHT)
@cron(CronExpression.EVERY_WEEKDAY)
@cron(CronExpression.MONDAY_TO_FRIDAY_AT_9AM)
```

Key members (supports both 5-field standard and 6-field seconds-precision expressions) — these are just a few of the 50+ predefined expressions available:

| Member                                 | Cron String         |
| -------------------------------------- | ------------------- |
| `EVERY_SECOND`                         | `* * * * * *`       |
| `EVERY_5_SECONDS`                      | `*/5 * * * * *`     |
| `EVERY_MINUTE`                         | `*/1 * * * *`       |
| `EVERY_HOUR`                           | `0 0-23/1 * * *`    |
| `EVERY_DAY_AT_MIDNIGHT`                | `0 0 * * *`         |
| `EVERY_DAY_AT_NOON`                    | `0 12 * * *`        |
| `EVERY_WEEKDAY`                        | `0 0 * * 1-5`       |
| `EVERY_WEEKEND`                        | `0 0 * * 6,0`       |
| `EVERY_1ST_DAY_OF_MONTH_AT_MIDNIGHT`   | `0 0 1 * *`         |
| `EVERY_QUARTER`                        | `0 0 1 */3 *`       |
| `EVERY_YEAR`                           | `0 0 1 1 *`         |
| `MONDAY_TO_FRIDAY_AT_9AM`              | `0 0 09 * * 1-5`    |
| `EVERY_30_MINUTES_BETWEEN_9AM_AND_5PM` | `0 */30 9-17 * * *` |

## `SchedulerRegistry` — Runtime Task Management

`SchedulerRegistry` is registered at application scope (`providersPerApp`) and is injectable anywhere in the application — services, controllers, etc.

```ts
import { injectable } from '@holu/core';
import { SchedulerRegistry } from '@holu/schedule';

@injectable()
export class TaskManagerService {
  constructor(private registry: SchedulerRegistry) {}

  listAll() {
    return {
      cronJobs: Array.from(this.registry.getCronJobs().keys()),
      intervals: this.registry.getIntervals(),
      timeouts: this.registry.getTimeouts(),
    };
  }

  stopCron(name: string): void {
    if (this.registry.doesExist('cron', name)) {
      this.registry.deleteCronJob(name); // stops and removes
    }
  }

  pauseAndResumeCron(name: string): void {
    const job = this.registry.getCronJob(name); // throws if not found
    void job.stop();
    // ... later:
    job.start();
  }

  stopInterval(name: string): void {
    if (this.registry.doesExist('interval', name)) {
      this.registry.deleteInterval(name); // calls clearInterval internally
    }
  }

  cancelTimeout(name: string): void {
    if (this.registry.doesExist('timeout', name)) {
      this.registry.deleteTimeout(name); // calls clearTimeout internally
    }
  }
}
```

### `SchedulerRegistry` API

| Method                                | Description                                                                        |
| ------------------------------------- | ---------------------------------------------------------------------------------- |
| `doesExist(type, name)`               | Returns `true` if a task of the given type with that name is registered.           |
| `getCronJob(name): CronJob`           | Returns the raw `CronJob` instance (from the `cron` package). Throws if not found. |
| `getCronJobs(): Map<string, CronJob>` | Returns all registered cron jobs.                                                  |
| `deleteCronJob(name): void`           | Stops the cron job and removes it from the registry. Throws if not found.          |
| `getIntervals(): string[]`            | Returns names of all registered intervals.                                         |
| `getInterval(name): any`              | Returns the raw interval handle. Throws if not found.                              |
| `deleteInterval(name): void`          | Calls `clearInterval` and removes the entry. Throws if not found.                  |
| `getTimeouts(): string[]`             | Returns names of all registered timeouts.                                          |
| `getTimeout(name): any`               | Returns the raw timeout handle. Throws if not found.                               |
| `deleteTimeout(name): void`           | Calls `clearTimeout` and removes the entry. Throws if not found.                   |
| `addCronJob(name, job): void`         | Manually registers a pre-built `CronJob`. Use for programmatic job creation.       |
| `addInterval(name, intervalId): void` | Manually registers a raw interval handle.                                          |
| `addTimeout(name, timeoutId): void`   | Manually registers a raw timeout handle.                                           |

> [!TIP]
> Always call `doesExist()` before `getCronJob()` / `getInterval()` / `getTimeout()` when the existence of the task is not guaranteed — the getters throw an `Error` if the name is not found.

#### Programmatic Task Registration Example

If you need to dynamically create and register a task at runtime (rather than using class decorators):

```ts
import { CronJob } from 'cron';

// Inside a service or controller method:
const customJob = new CronJob('* * * * *', () => {
  console.log('Running programmatic cron job...');
});
this.registry.addCronJob('dynamic-cron', customJob);
customJob.start();
```

## Graceful Shutdown

`SchedulerOrchestrator` implements the Holu `BeforeShutdown` lifecycle hook. On application shutdown it automatically:

1. Clears all pending `initialDelay` timers.
2. Stops and deletes all registered cron jobs.
3. Clears and deletes all intervals.
4. Clears and deletes all timeouts.

No manual cleanup is required in application code.

## Disabled Jobs and `initialDelay`

Use `disabled: true` to register a cron job without starting it. Start it manually when ready:

```ts
@cron(CronExpression.EVERY_HOUR, { name: 'heavy-report', disabled: true })
runHeavyReport(): void { /* ... */ }
```

```ts
// Elsewhere, e.g. triggered by a route or admin action:
const job = this.registry.getCronJob('heavy-report');
job.start();
```

Use `initialDelay` to defer the first execution after bootstrap:

```ts
@cron(CronExpression.EVERY_5_MINUTES, {
  name: 'delayed-poller',
  initialDelay: 10_000, // starts only after 10 seconds post-bootstrap
})
pollExternalApi(): void { /* ... */ }
```

> [!IMPORTANT]
> If `disabled: true` is combined with `initialDelay`, the job is registered but never started automatically — `initialDelay` is silently ignored in that case.

## Provider Scope Constraints

Scheduled tasks are application-lifetime singletons that must be instantiated once, not per-route or per-request.

| Provider scope    | Scheduling supported? | Notes                                                                             |
| ----------------- | --------------------- | --------------------------------------------------------------------------------- |
| `providersPerApp` | Yes                   | Scanned only in the module where `isLastModule === true` during extension stage1. |
| `providersPerMod` | Yes                   | Scanned in the module that imports `ScheduleModule`.                              |
| `providersPerRou` | No — warning logged   | Tasks cannot be route-scoped singletons.                                          |
| `providersPerReq` | No — warning logged   | Tasks cannot be request-scoped singletons.                                        |

> [!IMPORTANT]
> `SchedulerRegistry` and `SchedulerOrchestrator` are registered at `providersPerApp`. All tasks from all modules share a single registry and are all cleaned up together on shutdown.

## Internals: Bootstrap Lifecycle

Understanding how `ScheduleExtension` operates helps avoid subtle bugs:

1. **`stage1`** — scans `providersPerMod` (and `providersPerApp` for the last module) for classes that have `@cron`, `@interval`, or `@timeout` method decorators. Emits warnings for route/request-scoped classes.
2. **`stage2`** — receives the ready module injector. Resolves each scanned class via the injector, reads its decorator metadata via `Reflector.collectMeta()`, and registers each decorated method with `SchedulerOrchestrator` (which stores them as pending).
3. **`stage3`** — calls `SchedulerOrchestrator.mountJobs()`, which activates all pending timeouts (`setTimeout`), intervals (`setInterval`), and cron jobs (`CronJob.from(...)`). This runs once for the entire application.

The `ScheduleExtension` is registered with `exportOnly: true` in `ScheduleModule`, meaning it runs only in the modules that import `ScheduleModule`, not in `ScheduleModule` itself.

## Troubleshooting

| Symptom                                              | Cause / Fix                                                                                                         |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Task method never executes                           | Class not registered in `providersPerMod` or `providersPerApp` of a module that imports `ScheduleModule`.           |
| Warning: `Cannot register schedule on "Foo@bar"...`  | Task class is in `providersPerRou` or `providersPerReq`. Move it to `providersPerMod`.                              |
| `getCronJob(name)` throws `No Scheduler found`       | Name not provided in `CronOptions`, or the task was already deleted. Use `doesExist()` first.                       |
| `addCronJob(name, job)` throws `Duplicate Scheduler` | Two decorated methods share the same `name`. Each name must be unique across the entire application.                |
| Cron job not stopped on shutdown                     | Ensure Holu shutdown hooks are properly triggered. `SchedulerOrchestrator.beforeShutdown()` handles cleanup.     |
| Task method throws an exception                      | The exception is caught and logged automatically. It will not crash the Node.js process.                            |
| `initialDelay` not working                           | Check that `disabled` is not also set to `true` — combining both prevents automatic start.                          |
| Anonymous tasks not retrievable by name              | Tasks without a `name` are assigned a random UUID at runtime. Always provide a `name` for tasks you need to manage. |

## End-to-End Example

Here is a complete example of defining scheduled tasks, registering them in a module, and dynamically interacting with them via a REST controller:

```ts
import { injectable } from '@holu/core';
import { controller, restRootModule, route } from '@holu/rest';
import { ScheduleModule, cron, interval, timeout, SchedulerRegistry } from '@holu/schedule';

@injectable()
export class MyScheduledTasks {
  @cron('*/5 * * * * *', { name: 'cron-job' })
  handleCron() {
    console.log('Cron job runs every 5 seconds');
  }

  @interval('interval-job', 3000)
  handleInterval() {
    console.log('Interval runs every 3 seconds');
  }

  @timeout('timeout-job', 2000)
  handleTimeout() {
    console.log('Timeout runs once after 2 seconds');
  }
}

@controller()
export class ScheduleController {
  constructor(private registry: SchedulerRegistry) {}

  @route('GET', 'tasks')
  listTasks() {
    return {
      intervals: this.registry.getIntervals(), // Returns array of interval names
      timeouts: this.registry.getTimeouts(), // Returns array of timeout names
      cronJobs: Array.from(this.registry.getCronJobs().keys()), // Returns array of cron job names
    };
  }

  @route('POST', 'tasks/stop-cron')
  stopCron() {
    this.registry.deleteCronJob('cron-job');
    return { message: 'Cron job stopped and deleted' };
  }
}

@restRootModule({
  imports: [ScheduleModule], // Required to enable scheduling
  controllers: [ScheduleController],
  providersPerMod: [MyScheduledTasks], // Scheduled tasks must be instantiated at the module or app level
})
export class AppModule {}
```
