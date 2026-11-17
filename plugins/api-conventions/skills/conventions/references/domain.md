# Domain layer

The domain holds the entities, value objects, ports and repository interfaces. It imports nothing from the framework,
the database driver or the validation library: it only depends on itself.

## Entities are nominal, immutable classes that validate themselves

Business rules live in the entity, written once, so they apply to every entry point (HTTP route, socket handler,
script) and can't be bypassed by building an object by hand.

```ts
export type ShopId = string & { readonly __brand: 'ShopId' };

export class Shop {
    // Never set at runtime: only makes the class nominal, so a plain object can't be passed off as a Shop without going
    // through create() or reconstitute()
    declare private readonly brand: 'Shop';

    private static readonly NAME_MAX_LENGTH = 100;

    private constructor(
        public readonly id: ShopId,
        public readonly name: string,
        public readonly createdAt: Date,
        public readonly updatedAt: Date,
    ) {}

    static create = (id: ShopId, name: string, now: Date) => {
        if (name !== name.trim() || name.length > Shop.NAME_MAX_LENGTH) return [null, 'SHOP_NAME_INVALID'] as const;

        return [new Shop(id, name, now, now), null] as const;
    };

    rename = (name: string, now: Date) => {
        // ...same rule, then:
        return [new Shop(this.id, name, this.createdAt, now), null] as const;
    };

    static reconstitute = (id: ShopId, name: string, createdAt: Date, updatedAt: Date) => {
        return new Shop(id, name, createdAt, updatedAt);
    };
}
```

- **Private constructor.** An entity is only obtained through:
  - `create()`, or a business verb (`User.register()`, `Alert.trigger()`), which validates;
  - `reconstitute()`, which rebuilds without validating an entity read back from the database. Only the database
    mappers call it, by convention.
- **`declare private readonly brand`** makes the class nominal: TypeScript compares types by shape, so without it any
  object with the same fields would pass for the entity. It is absent from the emitted JavaScript.
- **Immutable**: every field is `readonly`, and a transition (`alert.resolve(now)`, `shop.rename(name, now)`) returns a
  new instance instead of mutating. No code can change an entity without going through its rules, and an instance
  someone else holds never changes under them.
- **`now` is a parameter**, never `new Date()` inside the entity: entities created together in one use case (a user and
  its session) share the exact same instant, and entities can be tested without mocking the clock. The use case reads
  the clock once, as the first line of `execute()`.
- **Constants are `static readonly`** on the entity (`DURATION`, `TOKEN_LENGTH`, `COOLDOWN_DURATION`), public only when
  another layer needs them, with a comment when the value is a choice worth explaining.
- **A dependency the rule needs is passed to the factory**, e.g. `User.register(id, email, password, passwordHasher,
  now)`: validation and hashing then always happen in order, and no caller can hash a password it forgot to validate.

## Ids are branded types

```ts
export type UserId = string & { readonly __brand: 'UserId' };
```

Declared next to their entity. A `UserId` can't be passed where a `ShopId` is expected, nor can a plain string. The
cast (`as UserId`) only happens where an id enters the system: the id generator's output in the use case, the database
mappers, and request parameters in the routes.

## Value objects and child entities

- **A value object** (no identity, e.g. `Position`) follows the same pattern: private constructor, brand,
  `create()` returning a tuple, `reconstitute()`.
- **A child entity** that only exists through its parent (a shop's employees, a user's pending email verification) is
  part of the parent's aggregate:
  - its class lives in the parent's file and is **not exported**, so only the parent can create one;
  - the parent exposes only what the other layers need, such as `static reconstituteEmployee = Employee.reconstitute`
    for the mapper, or a constant (`User.EMAIL_VERIFICATION_TOKEN_LENGTH`);
  - there is no repository for it: the parent is loaded, changed through its methods, and saved whole, so rules
    spanning the children (exactly one owner) are enforced in one place and written in one operation.
- Separate aggregates (a user, its sessions, its transactions) only reference each other by id. Merge into one
  aggregate when there is a real joint invariant and a bounded number of children, not just because two things are
  related.

## Ports and repository interfaces

A port is an interface saying what the inner layers need from the outside world. The infrastructure implements it
(see `infrastructure.md`), and nothing inside knows how.

- **They all live in the domain** (`domain/repositories/`, `domain/ports/`), even those only use cases call. One place
  to look, as in DDD. Some ports are used by the entities themselves (`User.register()` takes a `PasswordHasher`), and
  those must be in the domain anyway. The one exception is readers, which serve display and live in the application
  (see `application.md`).
- **A repository is a collection of aggregates, as the domain sees it**: it creates, finds, saves and deletes entities.
  How long the data lives or what stores it is not the domain's concern: a room kept by a video service for the length
  of a stream is as valid a repository as a database table. A repository only exposes what the domain needs, not a
  full set of operations.
- **Any other port** covers an action on an external service with nothing to find again afterwards: hashing a
  password, generating an id or a token, sending an email.
- **Named in the domain's language**, never the technology's: `findByEmailVerificationToken`, `findManyNearPosition`,
  not `findOneByQuery`. No driver, ORM or framework type appears in a signature.
- **Parameter types derive from the entity** (`User['id']`, `NonNullable<User['emailVerification']>['token']`), so they
  follow it when it changes.
- **Finding** returns the entity or `null` (an array for several). An id that isn't even well-formed is just not found,
  never an exception: it comes from the outside world, like any other unknown id.
- **A repository only ever returns its own aggregate**, never one combined with another (an alert with its user). Each
  aggregate is loaded and saved on its own, so its boundary stays clear. Its implementation may read several tables to
  build that one aggregate, but a read spanning aggregates, e.g. for display, goes through a reader (see
  `application.md`).
- **Saving** writes the whole aggregate, children included.
- **An expected failure is a return value here too.** When a constraint of the store expresses a business rule, the
  write reports it instead of the use case checking first: `create(user)` returns `'EMAIL_ALREADY_IN_USE' | null`,
  enforced by a unique index. One operation instead of two, and no race between the check and the write.

## Expected failures are return values, exceptions are for the unexpected

- A function that can fail returns `[value, null]` or `[null, 'CODE']`, with `as const`, so the code's type is the exact
  union of possible failures.
- A function with nothing to return on success (a check, a write) returns `'CODE' | null`. A function that can't fail
  returns its value directly.
- Private validators return a code or `null`, and the caller forwards it: `if (code) return [null, code] as const;`.
- Codes are `UPPER_SNAKE_CASE` string literals, never `Error` subclasses. They name the subject first, then what is
  wrong: `EMAIL_INVALID`, `PASSWORD_TOO_WEAK`, `ALERT_EXPIRED`, `USER_NOT_FOUND`, never `INVALID_EMAIL`. The codes of
  one resource then sort together. Generic HTTP codes keep their status name (`BAD_REQUEST`, `UNAUTHORIZED`).
- **Exceptions are only for what can't be recovered**: a programmer error (a state that should be impossible, e.g. a
  token of the wrong length, which means the generator is miswired) or a technical failure. They are never caught to be
  turned into a business code; they reach the global error handler and become a 500.

Why: an exception doesn't appear in a function's type, so nothing tells the caller which failures to handle, and a new
one compiles fine and ends as a 500 in production. Tuples were chosen over `{ success, data | code }` objects, whose
unions of object literals broke type narrowing in places, and over `Result` libraries, which add a dependency for what
two lines of destructuring cover.

## Entities accept only clean input

An entity validates values and refuses anything not already clean: an email with uppercase letters or a name with
surrounding spaces is rejected, never normalized. What is stored is then exactly what the client sent, and a dirty
value reveals a client bug instead of being silently fixed. Cleaning the input is the client's job.

The entity checks **values** (a malformed email: 422) and **state** (an alert already resolved: 409 or 410). Checking
the **shape** of a request (missing field, wrong type: 400) is the route schema's job, see `http.md`.
