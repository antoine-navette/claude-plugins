---
name: conventions
description: Conventions for TypeScript APIs built with Clean Architecture (domain entities, use cases, repositories, Fastify routes, configuration and logs). Use when writing, reviewing or refactoring the backend code of a project that follows them, or when aligning a project on them.
---

# API conventions

These are default conventions, written as decisions with their reasons so they can be adapted to another stack (the
examples use Fastify, Zod and MongoDB).

## The project comes first

Read the project's `CLAUDE.md` and documentation before applying anything here. When the project states a different
choice, follow the project. When a convention here contradicts what the project does or says, or when it is unclear
which applies, point it out and ask before changing anything.

## How to work

- **Aligning an existing project**: first list the gaps between the project and these conventions, then propose the
  changes as separate steps, smallest and safest first. Implement one step at a time and let it be reviewed before the
  next.
- **Discuss before implementing** any choice that isn't obvious, with the trade-offs, instead of picking silently.
- **Stay within the requested scope**: mention a related improvement instead of doing it.
- **Check statically** (type checking, linting, formatting, unit tests) before handing work back.
- **Never commit or push** unless asked.

## Layers

```
src/
├── domain/          # entities, value objects, ports, repository interfaces
├── application/     # commands, queries, readers' interfaces
├── infrastructure/  # repositories, readers and ports implemented, configuration
├── presentation/    # routes, middlewares, schemas, presenters
└── index.ts         # composition root
```

- **Dependencies point inwards**: the domain imports nothing; the application imports the domain; the infrastructure
  and the presentation import the application and the domain, and never each other.
- **`index.ts` wires everything by hand**: it loads the configuration, instantiates the adapters, passes them to the use
  cases' constructors, and the use cases to the presentation. There is no dependency injection container.

## Key rules

- Entities are nominal (`declare private readonly brand`), immutable classes with a private constructor, created by a
  validating `create()` and rebuilt by `reconstitute()`. Ids are branded types.
- Expected failures are returned, never thrown: `[value, null] | [null, 'CODE']`, or `'CODE' | null`. Codes name the
  subject first (`EMAIL_INVALID`). Exceptions are for programmer errors and technical failures.
- Use cases are commands or queries named after the user's intent, the same name carrying through the route and the
  frontend service. They orchestrate, and never call each other.
- A repository only returns its own aggregate; a read spanning aggregates goes through a reader.
- Routes handle each code with an `if` and end with `return code satisfies never;`. Presenters convert entities into
  responses explicitly.
- Entry points load the environment before creating the logger.

## References

Read the file matching the task:

| File                           | Read it to                                                                                  |
| ------------------------------ | ------------------------------------------------------------------------------------------- |
| `references/domain.md`         | write an entity, a value object, an id, a port or a repository interface                    |
| `references/application.md`    | write a command, a query or a reader                                                        |
| `references/infrastructure.md` | implement a repository, a reader, a mapper or an adapter; configure env, logs, entry points |
| `references/presentation.md`   | write a route, a schema, a presenter or a middleware; handle authentication or logs         |
