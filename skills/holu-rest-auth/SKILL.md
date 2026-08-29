---
name: holu-rest-auth
description: Authentication and authorization patterns for Holu REST applications (@holu/rest). Covers token-based auth (@holu/jwt), OAuth integrations (@holu/authjs), session cookies (@holu/session-cookie), and building guards implementing CanActivate.
---

# Holu Authentication & Authorization

Holu provides dedicated modules for handling authentication and session management.

## 1. Building Guards (`CanActivate`)

To protect routes, create a Guard class decorated with `@guard()` from `@holu/rest`. It implements `CanActivate`.

```ts
import { guard, CanActivate, RequestContext } from '@holu/rest';

@guard()
export class AuthGuard implements CanActivate {
  async canActivate(ctx: RequestContext): Promise<boolean | Response> {
    const token = ctx.rawReq.headers.authorization;
    if (!token) {
      // Returning false automatically sends a 401 response and halts execution
      return false;
      // Alternatively, return a Web API Response object for custom status/body
      // return new Response('Forbidden', { status: 403 });
    }
    
    ctx.auth = { userId: 123 }; // Pass data to the controller
    return true; // Allows execution
  }
}
```

Apply it to a route:
```ts
@route('GET', 'profile', [AuthGuard])
getProfile(ctx: RequestContext) {
  return { user: ctx.auth };
}
```

## 2. JWT Authentication (`@holu/jwt`)

Use `@holu/jwt` for stateless token-based authentication.

### Setup
```ts
import { restRootModule } from '@holu/rest';
import { JwtModule } from '@holu/jwt';

@restRootModule({
  imports: [
    JwtModule.withOpts({
      secret: 'your-super-secret-key',
      signOptions: { expiresIn: '1h' },
    })
  ]
})
export class AppModule {}
```

### Usage
Use `JwtService` to sign and verify tokens (`signWithSecret`, `verifyWithSecret`).

```ts
import { JwtService, JWT_PAYLOAD } from '@holu/jwt';
import { guard, CanActivate, RequestContext, controller, route } from '@holu/rest';
import { ctx } from '@holu/core';

@guard()
export class JwtGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  async canActivate(context: RequestContext) {
    const token = context.rawReq.headers.authorization?.split(' ')[1];
    if (!token) return false;

    try {
      const payload = await this.jwtService.verifyWithSecret(token);
      // Set payload via the generic Context to be injected later
      context.set(JWT_PAYLOAD, payload);
      return true;
    } catch {
      return false;
    }
  }
}

@controller()
export class ProfileController {
  @route('GET', 'profile', [JwtGuard])
  getProfile(@ctx(JWT_PAYLOAD) payload: any) {
    return { message: 'Authenticated!', user: payload };
  }
}
```

## 3. Auth.js / NextAuth (`@holu/authjs`)

The `@holu/authjs` module provides OAuth and credential authentication.

### Setup
```ts
import { restRootModule, controller, route } from '@holu/rest';
import { AuthjsModule, AuthjsInterceptor, AuthjsGuard } from '@holu/authjs';
import github from '@holu/authjs/providers/github';

// Must define the Auth.js protocol endpoint route
@controller()
class AuthController {
  @route('GET', 'auth/:action/:providerId?', [], [AuthjsInterceptor])
  @route('POST', 'auth/:action/:providerId?', [], [AuthjsInterceptor])
  auth() {}
}

@restRootModule({
  imports: [
    AuthjsModule.withConfig({
      providers: [
        github({ clientId: process.env.GH_ID, clientSecret: process.env.GH_SECRET }),
      ],
      secret: process.env.AUTH_SECRET,
    }),
  ],
  controllers: [AuthController]
})
export class AppModule {}
```

### Protecting Routes
Apply `AuthjsGuard` to your routes. The guard populates `ctx.auth` with the session. For manual retrieval in public routes, use `getSession(ctx, config)` imported from `@holu/authjs`.

```ts
@controller()
export class DashboardController {
  @route('GET', 'dashboard', [AuthjsGuard])
  getDashboard(ctx: RequestContext) {
    // If the route is hit, the user is authenticated
    return { user: ctx.auth };
  }
}
```

## 4. Session Cookies (`@holu/session-cookie`)

For simple session IDs without full JWTs.

```ts
import { restRootModule } from '@holu/rest';
import { SessionCookieModule, SessionCookie } from '@holu/session-cookie';

@restRootModule({
  imports: [
    SessionCookieModule.withOpts({
      cookieName: 'sessionId',
      maxAge: 86_400_000, // Important: Value is in milliseconds! (1 day)
      httpOnly: true,
    })
  ]
})
export class AppModule {}
```

In your controller, inject `SessionCookie` to manage the ID:
```ts
import { controller, route, RequestContext } from '@holu/rest';

@controller()
export class CartController {
  constructor(private sessionCookie: SessionCookie) {}

  @route('POST', 'cart')
  addToCart(ctx: RequestContext) {
    if (!this.sessionCookie.id) {
      this.sessionCookie.id = crypto.randomUUID();
    }
    return { sessionId: this.sessionCookie.id };
  }
}
```
