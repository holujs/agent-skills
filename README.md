# Holu Agent Skills

A collection of [Agent Skills](https://github.com/vercel-labs/skills) for AI coding assistants (such as Claude Code, Gemini CLI, Cursor, Copilot, Codex, and others) working with [Holu](https://holujs.github.io/en/).

[Holu](https://holujs.github.io/en/) is a TypeScript Node.js web framework built around decorators, modules, dependency injection, extensions, metadata reflection, and explicit module composition.

## 🚀 Installation

Install all available skills at once into your project using the `skills` CLI:

```bash
npx skills add https://github.com/holujs/agent-skills --skill '*' -y
```

## 📦 Available Skills

| Skill | Description |
| --- | --- |
| **[holu-core-architecture](skills/holu-core-architecture/SKILL.md)** | Core pillars of Holu: Modules, Dependency Injection (DI) hierarchy, provider scopes (`providersPerApp`, `providersPerMod`, `providersPerReq`, etc.), and custom decorators. |
| **[holu-project-setup](skills/holu-project-setup/SKILL.md)** | Bootstrapping projects with `@holu/cli` (`holu`), project templates (REST, tRPC), minimal single-file apps, and development workflows. |
| **[holu-rest](skills/holu-rest/SKILL.md)** | HTTP request lifecycle in `@holu/rest`, sequence of execution (interceptors, guards, routing, controllers), and custom request dispatchers. |
| **[holu-extensions](skills/holu-extensions/SKILL.md)** | Building, registering, ordering, exporting, grouping, and overriding Holu extensions (stage 1–3 initialization hooks). |
| **[holu-schedule](skills/holu-schedule/SKILL.md)** | Task scheduling via `@holu/schedule` (`@cron`, `@interval`, `@timeout`), `SchedulerRegistry`, provider scopes, and graceful shutdown patterns. |
| **[holu-rest-testing](skills/holu-rest-testing/SKILL.md)** | Testing utilities for Holu REST applications, isolated DI container setup, mocking providers, and SuperTest integration via `@holu/rest-testing`. |
| **[holu-rest-openapi](skills/holu-rest-openapi/SKILL.md)** | Generating OpenAPI documentation with `@holu/openapi` and automatic request/response validation with `@holu/openapi-validation`. |
| **[holu-rest-auth](skills/holu-rest-auth/SKILL.md)** | Authentication and authorization for REST applications using JWTs (`@holu/jwt`), NextAuth.js (`@holu/authjs`), and session cookies (`@holu/session-cookie`). |
| **[holu-rest-body-parser](skills/holu-rest-body-parser/SKILL.md)** | Parsing REST HTTP request bodies (JSON, text, URL-encoded) and handling file uploads with `MulterParser` via `@holu/body-parser`. |

## 💡 How It Works

Agent skills provide authoritative guidance, architectural rules, and code patterns directly to AI assistants. When an AI agent encounters a Holu codebase with these skills enabled, it leverages them to produce idiomatic, type-safe, and architecturally accurate code.

## 📜 License

[MIT](LICENSE)
