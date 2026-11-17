# Infrastructure layer

The infrastructure implements the domain's ports and repositories and the application's readers, and holds the
configuration. It depends on the domain and the application, never on the presentation.

## Layout: one folder per technology

```
infrastructure/
├── mongo/              # or drizzle/, prisma/...: everything about one store
│   ├── documents/      # the stored shapes (rows/tables with SQL)
│   ├── mappers/
│   ├── repositories/
│   └── readers/
├── adapters/           # the other ports: argon2.password-hasher.ts, crypto.token-generator.ts...
└── config/             # environment, logger, connections
```

Grouping by technology says where to look, and stays clear with several of them (a database, a video service, an email
provider). Names start with the technology: `MongoUserRepository`, `MongoUserMapper`, `Argon2PasswordHasher`, in
`<technology>.<port>.ts` files.

The examples below use MongoDB. The rules hold with SQL (rows instead of documents, a unique violation instead of a
duplicate key error), and the points that only concern one of them say so.

## Repositories and readers

```ts
export class MongoUserRepository implements UserRepository {
    constructor(private userCollection: Collection<MongoUserDocument>) {}

    create = async (user: User) => {
        try {
            await this.userCollection.insertOne(MongoUserMapper.toInfrastructure(user));
            return null;
        } catch (err) {
            // Duplicate key on the unique email index: the email is already in use
            if (err instanceof MongoServerError && err.code === 11000 && err['keyPattern']?.email) {
                return 'EMAIL_ALREADY_IN_USE' as const;
            }
            throw err;
        }
    };

    findById = async (id: User['id']) => {
        if (!ObjectId.isValid(id)) return null;

        const document = await this.userCollection.findOne({ _id: new ObjectId(id) });

        return document ? MongoUserMapper.toDomain(document) : null;
    };
}
```

- **A malformed id returns `null` (or `[]`)**, never a 500 (see `domain.md`). Check it before querying only when the
  store rejects it instead of finding nothing: building an `ObjectId` throws, and so does PostgreSQL comparing a `uuid`
  column with a malformed string. Against a `text` column, the query simply finds nothing.
- **A library that can only signal an expected failure with an exception** (a duplicate key meaning "email already in
  use") is adapted here: catch that precise exception, return the code, rethrow everything else. The rest of the code
  never sees it. This is the only place a business code comes from an exception.
- **Saving writes the whole aggregate**, every field but the id.
- **No business rule**: a repository only translates between the entity and the store.
- **One aggregate, one read.** With MongoDB, an aggregate is one document, children included, so loading it never
  needs a `$lookup`. With SQL, an aggregate may span several tables, joined to build it. Either way, a read spanning
  aggregates is a reader: one query (a `$lookup`, a join), then each part goes through its own mapper, and the result
  is an entity enriched with
  `Object.assign(MongoAlertMapper.toDomain(alert), { user: MongoUserMapper.toDomain(user) })`.

## Mappers

```ts
export const MongoAlertMapper = {
    toInfrastructure: (alert: Alert): MongoAlertDocument => ({
        _id: new ObjectId(alert.id),
        userId: new ObjectId(alert.userId),
        // ...every field
    }),

    toDomain: (document: MongoAlertDocument): Alert =>
        Alert.reconstitute(document._id.toHexString() as AlertId, document.userId.toHexString() as UserId /* ... */),
};
```

- **One mapper per entity or value object**, an object with `toDomain()`, which builds the domain model, and
  `toInfrastructure()`, which builds the shape the technology stores or expects: a document, a row, an external API's
  payload.
- **`toDomain()` calls `reconstitute()`**, the only caller besides tests, and casts the stored ids back to branded ids.
  A child entity goes through its parent's exposed reconstitution (`Shop.reconstituteEmployee`), a value object through
  its own mapper.
- **The stored shape is a plain type** (`MongoAlertDocument`), declared in `documents/`: no class, no `Readonly`.
  References are stored in the store's native id type (`ObjectId`), dates as dates.
- **An absent value is `null`, never a missing field**: the entity declares it `T | null`, not as an optional
  property, and the mapper passes it through as is. With SQL, the column is simply `NULL`. With MongoDB, the key is
  always written, set to `null`: clearing a value is then a plain overwrite when saving the whole document, instead of
  an `$unset` to handle on each change. Beware of unique indexes, which treat every `null` as the same value: a field
  that can be `null` needs a partial index, or a `sparse` one when it is nested under a parent that is `null`.

## Adapters

- **One class per port and technology**, implementing the domain's interface: `Argon2PasswordHasher`,
  `CryptoTokenGenerator`, `MongoIdGenerator`, `NodemailerEmailSender`.
- **Security-sensitive choices live here, with a comment**: random tokens from `node:crypto`, password hashing
  parameters, and comparing against a dummy hash when the user doesn't exist, so that the response time doesn't reveal
  whether an email is registered.

## Configuration

`config/` reads the environment and opens the connections. Each entry point calls it.

- **`env.ts` is one Zod schema in camelCase, filled by `loadEnv()` from `process.env`.** No key renaming afterwards. A
  group of settings only needed in some setups is a nested object, required only when enabled and `null` otherwise, so
  no dummy value is needed in development:

  ```ts
  logs: z.object({
      targets: z.array(z.enum(['console', 'betterstack'])).min(1),
      level: z.enum(['trace', 'debug', 'info', 'warn', 'error', 'fatal']),
      console: z.object({ format: z.enum(['json', 'pretty']) }).nullable(),
      betterStack: z.object({ url: z.url(), token: z.string().min(1) }).nullable(),
  }),
  ```

  A field only needed to validate the input (the list of targets) is dropped from the output with a `transform`, so
  the parsed config has a single source of truth.
- **Explicit settings, no `NODE_ENV`**: each variable says what it controls (`LOG_LEVEL`, `CONSOLE_LOG_FORMAT`)
  instead of one global mode changing several behaviours at once.
- **Connections check their dependency at startup** (Mongo connects, the SMTP transport runs `verify()`), so a wrong
  URL or an unreachable service stops the boot instead of failing on the first request.
- **The logger (pino) writes to every enabled target** through `pino.transport({ targets })`. The level is set on the
  logger and on each target, which otherwise filters at `info`. Its `flush()` never rejects and gives up after 3
  seconds: logs are best-effort, and the process still has to exit within Docker's 10 seconds between `SIGTERM` and
  `SIGKILL`.

## Entry points

`index.ts` and the scripts (migrations, docs generation) start the same way:

1. **Load the environment first**, outside the main `try`. If it is invalid, print the error with `console.error` and
   exit with code 1: there is no logger yet, the logging config itself may be what's wrong. A config error is
   deterministic and shows up at deployment.
2. **Create the logger** from the validated config.
3. **Run the rest in a `try`.** Every failure from there on is logged with a fixed message and the error
   (`logger.fatal({ err }, 'Failed to start server')`, `'Failed to run migrations'`), so runtime failures such as an
   unreachable database reach every log target. Then flush and exit with code 1.

`index.ts` is the composition root: it instantiates the adapters, repositories and readers, passes them to the use
cases' constructors, and the use cases to the presentation. It also handles `SIGINT` and `SIGTERM`: close the server,
then the connections, flush the logs, exit.
