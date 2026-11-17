# Application layer

The application holds the use cases, split into commands and queries, and the readers' interfaces. It depends on the
domain only.

## One use case per user action, named after the intent

A use case is named after what the user does, in the business vocabulary: `TriggerAlertCommand`, `ResumeAlertQuery`,
`WatchShopQuery`, never a CRUD name (`CreateAlert`, `ReadAlert`) that says what happens to the data but not why. The
same name carries through every layer, so searching for it finds the whole feature:

| Use case              | HTTP route                             | Front service    |
| --------------------- | -------------------------------------- | ---------------- |
| `TriggerAlertCommand` | `triggerAlertRoute` (`POST /alerts`)   | `triggerAlert()` |
| `ResumeAlertQuery`    | `resumeAlertRoute` (`GET /alerts/:id`) | `resumeAlert()`  |

Files are grouped by resource: `application/commands/alert/trigger-alert.command.ts`,
`application/queries/shop/watch-shop.query.ts`. A new feature is named first: finding the name forces clarifying what
it does.

## Commands and queries

A light CQRS, with no separate read model:

- **A command** may write, even rarely (authenticating mostly reads, but extends the session). It loads aggregates
  through the repositories, calls their methods and saves them.
- **A query** only reads. On top of the repositories, it may use readers (below).

A query that starts writing becomes a command: it changes folder and suffix.

## Shape of a use case

```ts
export class CancelAlertCommand {
    constructor(private alertRepository: AlertRepository) {}

    execute = async (id: AlertId, userId: UserId) => {
        const now = new Date();

        const alert = await this.alertRepository.findById(id);
        if (!alert) return [null, 'ALERT_NOT_FOUND'] as const;
        if (alert.userId !== userId) return [null, 'ALERT_NOT_OWNED'] as const;

        const [updated, updatedCode] = alert.cancel(now);
        if (updatedCode) return [null, updatedCode] as const;
        await this.alertRepository.save(updated);

        return [updated, null] as const;
    };
}
```

- **A class whose dependencies come through the constructor** (repositories, ports, readers, plain values such as a
  sender address), and a single `execute()` method.
- **No dependency injection container**: the entry point (`index.ts`) instantiates the adapters and passes them to the
  use cases' constructors by hand. It is the only file that knows every concrete class.
- **The clock is read once**, as the first line, and passed to every entity method (see `domain.md`).
- **It returns a tuple** and forwards the domain's codes as they are (`if (code) return [null, code] as const;`), adding
  its own: not found, not owned.

## What a use case does, and doesn't

- **It orchestrates**: load, call the entities' methods, save, call the ports. Business rules stay in the entities.
- **The one rule it holds is access to a resource** ("is this alert his?", "is he an employee of this shop?"), since it
  is the one holding the resource. Such a failure is a code (`ALERT_NOT_OWNED`), that the route turns into a 403. A
  global role (admin) is checked earlier, by a presentation middleware, since it doesn't depend on any resource.
- **It never calls another use case.** Chained use cases quickly become hard to follow, and the shared part is usually
  accidental: two use cases that look alike today often diverge tomorrow. Duplicating a few lines of orchestration is
  fine. A real shared business rule belongs in an entity, where both use cases get it.
- **It takes the entity rather than its id when the caller already has it**: the presentation loads the authenticated
  session and user, so `LogoutCommand` takes the session and `RequestEmailVerificationCommand` takes the user, instead
  of loading them a second time.
- **It returns entities, never plain objects.** To return an entity with related data, it adds it with
  `Object.assign(alert, { shops })`, which keeps the class and its brand while adding the field to the type. Spreading
  it (`{ ...alert, shops }`) would turn it into a plain object, which no longer passes for an `Alert`. Turning entities
  into a response is the presentation's job (presenters).

## Readers

Repositories only return their own aggregate (see `domain.md`). A query displaying data from several aggregates at
once (the active alerts of a shop, each with its user) would then need one request per aggregate. A reader is a read
interface shaped for that display, answered in a single request by the infrastructure (a join, an aggregation
pipeline):

```ts
export interface ActiveAlertsReader {
    findManyByShopId: (shopId: ShopId, now: Date) => Promise<(Alert & { user: User })[]>;
}
```

- **Its interface lives in the application** (`application/readers/alert/active-alerts.reader.ts`), not the domain: it
  serves a screen, not a business rule.
- **It returns entities enriched with related ones**, rebuilt with `reconstitute()`, and holds no business rule.
- **Only queries use it, never commands**: a command changes an aggregate and must load it through its repository, as
  a whole and on its own.
