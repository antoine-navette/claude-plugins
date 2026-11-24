---
name: conventions
description: Conventions for React frontends (Vite) talking to a typed API through generated OpenAPI types (folder structure, pages and their state, services, entities and mappers, session, guards and error handling). Use when writing, reviewing or refactoring the frontend code of a project that follows them, or when aligning a project on them.
---

# Web conventions

These are default conventions, written as decisions with their reasons so they can be adapted to another stack (the
examples use React, Vite, react-router and openapi-fetch).

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
- **Check statically** (type checking, linting, formatting) before handing work back.
- **Never commit or push** unless asked.

## Key rules

- The code favours reading over anticipating: a page holds its own state and reads top to bottom, a component is
  extracted only when reused, hooks are for reusable technical logic, and some repetition is accepted. Pages and
  modals are screens: each has its own file, in `pages/` or `modals/`.
- Data flows page → service → HTTP client → mapper. Pages never see `fetch`, the client nor the generated types.
- Services are plain functions, one per API route and named like it, returning `[value, null] | [null, 'CODE']` (or
  `'CODE' | null`), with `NETWORK_ERROR` when the request doesn't complete.
- Entities are `Readonly` types, not classes, with `Date`s where the API sends strings. Mappers convert the API's
  schemas into them.
- The session is loaded before anything renders, so `useAuth().user` is a user or `null`. Any `UNAUTHORIZED` response
  signs the user out.
- Errors are stored as codes and shown through one message table. The frontend validates and cleans the input before
  sending, since the API refuses anything else.

## References

Read the file matching the task:

| File                               | Read it to                                                                           |
| ---------------------------------- | ------------------------------------------------------------------------------------ |
| `references/structure.md`          | place a file, write a page or a modal, extract a component or a hook, name things    |
| `references/api-access.md`         | call the API: generated types, HTTP client, services, entities, mappers              |
| `references/session-and-errors.md` | handle the session, guards, error codes and messages, input validation               |
