---
name: holu-rest-body-parser
description: HTTP request body parsing for Holu REST applications (@holu/rest). Covers BodyParserModule configuration via withOpts, accessing JSON/text/URL-encoded bodies via @ctx(HTTP_BODY), and handling multipart/form-data via MulterParser.
---

# @holu/body-parser

The `@holu/body-parser` package parses incoming HTTP request bodies (JSON, URL-encoded, raw text, and multipart/form-data).

## Registration

Register the `BodyParserModule` in your root or feature module using `withOpts` to pass `BodyParserConfig` options.

```ts
import { restRootModule } from '@holu/rest';
import { BodyParserModule } from '@holu/body-parser';

@restRootModule({
  imports: [
    BodyParserModule.withOpts({
      acceptMethods: ['POST', 'PUT', 'PATCH'],
      jsonOptions: { limit: '500kb' },
      urlencodedOptions: { extended: true },
    })
  ],
  // If used in a feature module, ensure it is exported so importing modules can use it
  // exports: [BodyParserModule]
})
export class AppModule {}
```

To disable the body parser for a specific controller or route, override the `BodyParserConfig` with empty accepted methods:
```ts
import { controller } from '@holu/rest';
import { BodyParserConfig } from '@holu/body-parser';

@controller({
  providersPerRou: [{ token: BodyParserConfig, useValue: { acceptMethods: [] } }]
})
export class CustomController {}
```

## Accessing JSON / Text / URL-Encoded Bodies

Parsed bodies are attached to `ctx.body` on the `RequestContext`. However, the recommended DI approach is to use `@ctx(HTTP_BODY)` to inject the body directly into your controller method parameters.

```ts
import { controller, route } from '@holu/rest';
import { ctx } from '@holu/core';
import { HTTP_BODY } from '@holu/body-parser';

@controller()
export class UserController {
  @route('POST', 'users')
  async createUser(@ctx(HTTP_BODY) body: any) {
    return { success: true, data: body };
  }
}
```

## Handling Multipart/Form-Data (File Uploads)

For file uploads, the module wraps `@ts-stack/multer`. You can use a request-scoped `MulterParser` or a route-scoped `RouteScopedMulterParser`.

Files in `@ts-stack/multer` are streaming (`file.stream` is a `node:stream.Readable`) and should be piped to storage.

### Example: Request-Scoped `MulterParser`

`MulterParser` is injected into the controller method and does not require `ctx` in its methods.

```ts
import { controller, route } from '@holu/rest';
import { MulterParser } from '@holu/body-parser';

@controller()
export class UploadController {
  @route('POST', 'upload/avatar')
  async uploadAvatar(parse: MulterParser) {
    // Parse a single file named "avatar"
    const parsedForm = await parse.single('avatar');
    
    // parsedForm contains 'textFields' and 'file' (or 'files')
    // Note: property is textFields, NOT body!
    const file = parsedForm.file;
    const fields = parsedForm.textFields;

    return { 
      message: 'Upload successful',
      filename: file?.originalName, // Note: camelCase originalName
      fields
    };
  }
}
```

### Example: Route-Scoped `RouteScopedMulterParser`

For better performance, use a route-scoped controller and inject `RouteScopedMulterParser`. Its methods require `ctx` as the first argument.

```ts
import { controller, route, RequestContext } from '@holu/rest';
import { RouteScopedMulterParser } from '@holu/body-parser';

@controller({ scope: 'route' })
export class FastUploadController {
  constructor(private multer: RouteScopedMulterParser) {}

  @route('POST', 'fast-upload')
  async upload(ctx: RequestContext) {
    const parsedForm = await this.multer.array(ctx, 'documents', 5);
    return { count: parsedForm.files.length };
  }
}
```

> [!WARNING]
> Do NOT use `@ctx(HTTP_BODY)` for multipart/form-data requests before parsing them with Multer. The body will be unparsed until the multer parser processes it.
