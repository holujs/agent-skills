---
name: holu-rest-openapi
description: OpenAPI generation and validation for Holu REST applications (@holu/rest). Covers OpenapiModule setup, documenting routes via @oasRoute, DTO classes with @property and REQUIRED, generating schemas via getContent and getParams, and automatic request validation using ValidationModule.
---

# Holu OpenAPI & Validation

The `@holu/openapi` package provides tools to generate OpenAPI specifications for your REST API, and `@holu/openapi-validation` allows you to automatically validate incoming requests against those specifications.

## Setup OpenAPI Endpoint

Import `OpenapiModule` and configure it in your root module using `withOpts(oasObject, absolutePath?, swaggerOAuthOptions?)`.

```ts
import { restRootModule } from '@holu/rest';
import { OpenapiModule } from '@holu/openapi';

@restRootModule({
  imports: [
    OpenapiModule.withOpts(
      {
        openapi: '3.1.0',
        info: { title: 'My API', version: '1.0.0' },
      },
      'docs' // Serves UI at /docs/openapi and JSON at /docs/openapi.json
    ),
  ],
})
export class AppModule {}
```

Exposed endpoints:
- `/docs/openapi` - Swagger UI HTML
- `/docs/openapi.json` - OpenAPI 3.1.0 JSON specification
- `/docs/openapi.yaml` - OpenAPI 3.1.0 YAML specification

## Documenting Routes and DTOs

Define your DTOs as classes and decorate properties with `@property()`. Use the `REQUIRED` symbol for mandatory fields.

```ts
import { property, REQUIRED } from '@holu/openapi';

export class CreateUserDto {
  @property({ [REQUIRED]: true, description: 'User handle' })
  username: string;

  @property({ format: 'email' })
  email: string;

  // Complex types (arrays, enums) are defined in the second argument
  @property({}, { array: String })
  tags: string[];
}
```

Use `@oasRoute` in your controller instead of `@route`. Generate the OpenAPI schemas using `getContent()` (for bodies/responses) and `getParams()` / `Parameters` (for path/query).

```ts
import { controller } from '@holu/rest';
import { ctx } from '@holu/core';
import { oasRoute, getContent } from '@holu/openapi';
import { HTTP_BODY } from '@holu/body-parser';
import { CreateUserDto } from './create-user.dto.js';

@controller()
export class UserController {
  @oasRoute('POST', 'users', [], [], {
    description: 'Create a new user',
    requestBody: {
      description: 'User data',
      // Use getContent() to auto-generate the schema from the DTO
      content: getContent({ mediaType: 'application/json', model: CreateUserDto }),
    },
    responses: {
      '201': {
        description: 'Created successfully',
      },
    },
  })
  createUser(@ctx(HTTP_BODY) body: CreateUserDto) {
    // Context parameter injection is used for bodies
    return { success: true, user: body };
  }
}
```

### Documenting Parameters (`getParams` / `Parameters`)

Use the `Parameters` class to define path, query, header, or cookie parameters for a route:

```ts
import { controller } from '@holu/rest';
import { oasRoute, Parameters } from '@holu/openapi';

@controller()
export class PostController {
  @oasRoute('GET', 'posts/:postId', [], [], {
    description: 'Get a post by ID',
    parameters: new Parameters()
      .required('path', 'postId')
      .optional('query', 'includeComments')
      .getParams(),
    responses: {
      '200': { description: 'Post found' },
    },
  })
  getPost() {}
}
```

> [!TIP]
> Use `DEFAULT_OAS_OBJECT` from `@holu/openapi` as the base for your OpenAPI config to avoid repeating boilerplate fields:
> ```ts
> import { OpenapiModule, DEFAULT_OAS_OBJECT } from '@holu/openapi';
>
> OpenapiModule.withOpts({ ...DEFAULT_OAS_OBJECT, info: { title: 'My API', version: '1.0' } })
> ```

## Request Validation

The `@holu/openapi-validation` package automatically validates incoming requests (body, query, params) against the schemas. It requires `BodyParserModule` for body parsing.

```ts
import { restRootModule } from '@holu/rest';
import { OpenapiModule } from '@holu/openapi';
import { ValidationModule } from '@holu/openapi-validation';
import { BodyParserModule } from '@holu/body-parser';

@restRootModule({
  imports: [
    BodyParserModule.withOpts({}),
    OpenapiModule.withOpts({ openapi: '3.1.0', info: { title: 'My API', version: '1.0' } }),
    ValidationModule,
  ],
})
export class AppModule {}
```

When validation fails, the interceptor automatically throws a `CustomError` and returns a `400 Bad Request`. You can customize this by providing `ValidationOptions` in `providersPerMod` (e.g. to change status code to Unprocessable Entity).

> [!NOTE]
> `@oasGuard()` is a class decorator applied to Guard classes to attach security requirement metadata, not a method decorator for routes!
