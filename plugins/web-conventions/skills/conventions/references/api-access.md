# API access

## Generated types

`types.gen.ts` is generated from the API's OpenAPI spec (`openapi-typescript`, through an `api:generate` script). It
describes every route: URL, parameters, body, and each response with its status. The HTTP client uses it to type every
call, so an unknown URL, an incomplete body or an unknown error code doesn't compile.

Both the spec and the generated types are committed. After adding, removing or changing a route on the API, regenerate
the spec there, then the types here. The CI fails when either no longer matches the code.

## The HTTP client

`config/http.ts` exports a single `openapi-fetch` client, typed with the generated `paths`, on `/api` (the same origin).
Besides calling the API, it:

- **aborts a request after 10 seconds**, so that a bad network ends in an error instead of an endless wait;
- **treats any response that isn't JSON as an unreachable API** (except a `204`): a proxy's error page during a restart
  carries no error code, so it is reported like a network failure;
- **reports a lost session**: any response with the `UNAUTHORIZED` code calls a callback the session provider
  registers (see `session-and-errors.md`). The code is checked, not the status: a `401` on login means wrong
  credentials, not a lost session.

## Services

```ts
export const login = async (email: string, password: string) => {
    const response = await http.POST('/auth/login', { body: { email, password } }).catch(() => null);
    if (!response) return [null, 'NETWORK_ERROR'] as const;
    if (response.error) return [null, response.error.code] as const;

    return [UserMapper.toEntity(response.data), null] as const;
};

export const logout = async () => {
    const response = await http.POST('/auth/logout').catch(() => null);
    if (!response) return 'NETWORK_ERROR' as const;
    if (response.error) return response.error.code;

    return null;
};
```

- **One function per API route**, named like the route and the use case behind it (`login`, `getUserTransactions`,
  `verifyEmail`), grouped by resource in `services/<resource>.ts`. Plain functions: they hold no state, and the client
  is a module, so a class would only group them.
- **The same result format as the API**: `[value, null]` or `[null, code]`, or `'CODE' | null` when there is no value.
  The code is the API's error code, typed from the spec for that very route, plus `NETWORK_ERROR` when the request
  doesn't complete (network, timeout, non-JSON response).
- **The service converts both ways**: it takes arguments in the entities' types and converts them for the request, and
  maps the response into entities. Pages never see the generated types.

## Entities and mappers

```ts
export type User = Readonly<{
    id: string;
    email: string;
    createdAt: Date;
    updatedAt: Date;
}>;

export const UserMapper = {
    toEntity: (schema: components['schemas']['User']): User => ({
        id: schema.id,
        email: schema.email,
        createdAt: new Date(schema.createdAt),
        updatedAt: new Date(schema.updatedAt),
    }),
};
```

- **Entities are `Readonly` types, not classes**: they hold no logic, and plain objects fit React, where state is
  replaced rather than mutated and spreading or copying a class instance would lose its methods.
- **They differ from the API's schemas where the API's format isn't the one to work with**, dates first: the API sends
  ISO strings, the entities hold `Date`s.
- **A mapper converts a schema into an entity**, field by field, once, in the services.
