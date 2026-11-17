# Presentation layer

The presentation receives requests, calls the use cases and shapes the responses. It depends on the application and the
domain, never on the infrastructure. The examples use Fastify with `fastify-zod-openapi`.

```
presentation/
├── app.ts           # createApp(): Fastify instance, hooks, plugins, error handler, routes
├── server.ts        # startServer(app): listens on the internal port
├── routes/<resource>/<intent>.route.ts
├── middlewares/     # isAuthenticated, isAdmin...
├── schemas/         # Zod schemas of the responses, one per error code too
└── presenters/
```

## One file per route, named after its use case

```ts
export const cancelAlertRoute: FastifyPluginAsyncZodOpenApi<Options> = async (fastify, opts) => {
    fastify.route({
        method: 'POST',
        url: '/alerts/:id/cancel',
        schema: {
            tags: ['Alert'],
            params: z.object({ id: z.string() }),
            response: {
                200: alertSchema,
                400: badRequestSchema,
                401: unauthorizedSchema,
                403: forbiddenSchema,
                404: alertNotFoundSchema,
                409: alertInactiveSchema,
                429: tooManyRequestsSchema,
                500: internalServerErrorSchema,
            },
        } satisfies FastifyZodOpenApiSchema,
        preHandler: isAuthenticated(opts.authenticateCommand),
        handler: async (request, reply) => {
            const [alert, code] = await opts.cancelAlertCommand.execute(
                request.params.id as AlertId,
                request.session.userId,
            );
            if (code) {
                if (code === 'ALERT_NOT_OWNED') {
                    request.log.warn({ code }, 'Forbidden');
                    return reply.status(403).send({ code: 'FORBIDDEN' });
                }
                if (code === 'ALERT_NOT_FOUND') {
                    request.log.warn({ code }, 'Not found');
                    return reply.status(404).send({ code: 'ALERT_NOT_FOUND' });
                }
                if (code === 'ALERT_INACTIVE') {
                    request.log.warn({ code }, 'Conflict');
                    return reply.status(409).send({ code: 'ALERT_INACTIVE' });
                }
                return code satisfies never;
            }

            request.log.info({ alertId: alert.id }, 'Alert cancelled');
            return reply.status(200).send(AlertPresenter.toSchema(alert));
        },
    });
};
```

- **The route, its file and its plugin are named after the use case** (`cancel-alert.route.ts`, `cancelAlertRoute`),
  repeating the resource even though the folder already gives it.
- **URLs stay resource-oriented**: `POST /alerts`, `GET /alerts/:id`, `DELETE /transactions/:id`. An action that isn't
  a plain create, read, update or delete gets its own verb as a `POST` on the resource: `POST /alerts/:id/cancel`,
  `POST /email-verifications/:token/verify`.
- **Every status the route can return is declared**, `400`, `429` and `500` included, so the OpenAPI spec generated
  from the schemas is complete. A response without a body is declared `204: {}`. The generated spec is committed, and
  the frontend generates its types from it.
- **The request schema only checks the shape** (types, required fields): a failure is a `400`. Business rules belong to
  the entities (see `domain.md`). Ids come in as strings and are cast to their branded type when passed to the use case.

## Codes become responses

- **Each code of the use case is handled by an `if`, and the block ends with `return code satisfies never;`**: a new
  code returned by the use case fails the build until the route handles it. Codes sharing a response share an `if`
  (`code === 'EMAIL_INVALID' || code === 'PASSWORD_TOO_WEAK'`).
- **The status follows the nature of the failure**:

  | Failure                                          | Status | Response code                                      |
  | ------------------------------------------------ | ------ | -------------------------------------------------- |
  | Shape of the request (missing field, wrong type) | 400    | `BAD_REQUEST`                                      |
  | Not authenticated, session gone                  | 401    | `UNAUTHORIZED`                                     |
  | Resource not accessible to this user             | 403    | `FORBIDDEN`                                        |
  | Resource not found                               | 404    | precise (`ALERT_NOT_FOUND`)                        |
  | State conflict (already used, inactive)          | 409    | precise (`EMAIL_ALREADY_IN_USE`, `ALERT_INACTIVE`) |
  | No longer available (expired)                    | 410    | precise (`ALERT_EXPIRED`)                          |
  | Value breaking a business rule                   | 422    | `UNPROCESSABLE_CONTENT`                            |

- **The response code is chosen by the route, not copied from the use case.** A 422 is always generic: the client
  validates the same rules before sending, so a 422 reveals a client bug, not a user mistake. Codes that would reveal
  too much are merged: an unknown email and a wrong password both answer `INVALID_CREDENTIALS`.
- **Every error body is `{ code }`**, with one Zod schema per code, holding a literal and registered for the spec
  (`.meta({ id })`).
- **The global error handler only maps framework errors** by their `statusCode` (400 for a malformed body or a failed
  schema, 413, 415, 429), with one `if` per status, then falls back to a 500 that logs the error. Business codes never
  reach it. The not-found handler answers `404 ROUTE_NOT_FOUND`.

## Presenters

```ts
export const AlertPresenter = {
    toSchema: (alert: Alert): AlertSchema => ({
        id: alert.id,
        shopIds: [...alert.shopIds],
        expiresAt: alert.expiresAt.toISOString(),
        // ...every field
    }),
};
```

No entity is sent as is: a presenter converts it explicitly, field by field. Response schemas declare dates as strings
(`z.iso.datetime()`, or `z.iso.date()` for a calendar day), so the presenter converts them itself instead of leaving it
to JSON serialization, and the response is validated against what is actually sent.

## Authentication

- **An opaque random token in a cookie**, not a JWT: a session can then be revoked at once (logout, ban), which a
  signed token can't before it expires. The cookie is `HttpOnly`, `Secure` (browsers treat `http://localhost` as a
  secure context, so development works too), `SameSite=Strict`, path `/`.
- **Only the token's hash is stored** (SHA-256 through a `TokenHasher` port), never the token itself: a copy of the
  database then can't be used to act as anyone. This holds for every token the server only has to verify (sessions,
  email verification, password reset). The clear token only exists in the cookie and in memory.
- **The session carries its token and its user on the fly**: `AuthenticateCommand`, `LoginCommand` and
  `RegisterCommand` all return `Object.assign(session, { token, user })`, rather than the session, the token and the
  user side by side. The same shape then flows everywhere, and `reply.setSessionCookie(session)` reads what it needs
  from it.
- **The `isAuthenticated` preHandler** reads the cookie and calls `AuthenticateCommand`. It answers `401 UNAUTHORIZED`
  itself for any failure (no cookie, unknown or expired session, deleted user), sets `request.session`, replaces
  `request.log` with a child logger carrying `sessionId` and `userId`, and renews the cookie, since sessions slide.
  `request.session` is declared on every request (a Fastify limitation): reading it in a route without the preHandler
  fails at runtime.
- **A global role** (admin) is checked by another preHandler placed after it. Access to one resource is checked by the
  use case (see `application.md`).

## Logs

- **Every outcome is logged where the response is sent**, right above `reply.send()`:
  - an expected failure logs a `warn` with the status's generic message (`'Not found'`, `'Conflict'`) and the precise
    `{ code }`;
  - a successful write logs an `info` naming the business event (`'Alert cancelled'`, `'User registered'`) with the ids
    involved. A successful read logs nothing more than the access log.
- **Messages are fixed strings**, never interpolated: the varying parts go in fields, so logs can be searched and
  grouped.
- **The resource the log is about comes first** among the fields (`{ sessionId, userId }`, `{ alertId }`).
- **One access log per request**, in an `onResponse` hook: method, URL, route, status, duration, IP and user agent.
  Fastify's own request logs are disabled.
- **Every request has an id**, a random UUID unless the client sent a `request-id` header, bound to every log of the
  request and echoed back in a `request-id` response header, so a client can report it.
- **When a route authenticates the user itself** (login, register), it replaces `request.log` with the same child
  logger as `isAuthenticated`, so the access log carries the ids too.
