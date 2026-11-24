# Session and errors

## Loading the session

At startup, `AuthProvider` asks the API for the signed-in user (`me()`) and renders nothing else until it has the
answer:

- a user: they are signed in;
- `UNAUTHORIZED`: there is no session, which is not an error;
- any other code (network, server, rate limit): an error page, with a retry button but no way back. These codes say
  nothing about the session, so they must not be mistaken for being signed out.

Only then is the rest of the application rendered. No page ever deals with a user "being loaded": `useAuth().user` is a
user or `null`, nothing else. The signed-in user is the only data shared between screens, and the context exposes it
with `setUser()`, which login, registration and logout call.

## Guards

Routes are grouped under guards, which read `useAuth()`:

| Guard          | Lets through          | Otherwise                                               |
| -------------- | --------------------- | ------------------------------------------------------- |
| `GuestGuard`   | A visitor, signed out | Redirect to `/`                                         |
| `AuthGuard`    | A signed-in user      | Redirect to `/login`                                    |
| `AdminGuard`   | An administrator      | Error page `FORBIDDEN` (always nested in `AuthGuard`)   |

They are display conveniences only: the API protects the data, on every request.

## Losing the session

A session can disappear while the application is open: signed out in another tab, deleted on the server. The HTTP
client checks every response: when its code is `UNAUTHORIZED`, it calls the callback `AuthProvider` registered with
`onUnauthorized()`. The user becomes `null`, and `AuthGuard` redirects to the login page, wherever the user was.

## Errors are codes

- **A page stores the code, never a message**: `{ status: 'error'; code: ErrorCode }`. A specific case is handled right
  after the service call, where the code is still precisely typed for that route.
- **`ErrorCode`** (`constants/errors.ts`) is every error code of the API, extracted from the generated schemas, plus the
  front-only ones: `NETWORK_ERROR`, and `ROUTE_NOT_FOUND` for the router's own 404 page.
- **The message shown comes from `ERROR_MESSAGES[code]`**, a `Record<ErrorCode, string>` in one place for the whole
  application: a new API code that has no message fails the build.
- **An error preventing the whole page from showing renders `<ErrorPage code={code} />`.** It offers a retry button,
  which reloads the application, for transient errors only (`NETWORK_ERROR`, `INTERNAL_SERVER_ERROR`,
  `TOO_MANY_REQUESTS`): the others would just happen again.

## Validating and cleaning the input

The API accepts only clean input and checks its business rules itself, answering a generic `422` when one is broken. So
the frontend owns the input experience:

- **It checks the same rules before sending** (required fields, lengths, formats), to show a precise message next to
  the right field. A `422` from the API then reveals a frontend bug, not a user mistake.
- **It cleans the input as it is typed or before sending**: an email lowercased, the spaces around a name trimmed. The
  API refuses such values instead of fixing them silently.
- A rule changed on the API must be changed here too, or users meet `422`s they can't understand.
