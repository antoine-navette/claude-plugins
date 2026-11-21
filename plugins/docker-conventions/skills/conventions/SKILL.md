---
name: conventions
description: Conventions for running a web project (an API and a frontend) entirely through Docker Compose, in development and production (layered compose files, ports, healthchecks, environment files, Dockerfiles, stop signals, nginx as the single origin, backups). Use when writing, reviewing or refactoring a project's Docker setup, or when aligning a project on it.
---

# Docker conventions

These are default conventions, written as decisions with their reasons so they can be adapted to another stack (the
examples use Node or Bun, MongoDB and nginx).

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
- **Check what can be checked** before handing work back (`docker compose config`, `nginx -t`), and list what changes on
  the server for a deployment (files to edit, steps to run, their order).
- **Never commit or push** unless asked.

## Key rules

- The project only runs through Docker Compose: internal values (ports, service addresses) are hardcoded, only the host
  side is configurable.
- Compose files are layered: `compose.yaml` shared, `compose.override.yaml` for development, `compose.prod.yaml` for
  production, selected by `COMPOSE_FILE` in the root `.env`. Development runs with no configuration.
- Every published port is bound to `127.0.0.1`. The API is never published: the web container relays `/api` to it.
- Every dependency has a fast-starting healthcheck, and services wait for theirs to be healthy.
- Production services read their settings from an `env_file`. Development scripts read `.env.example` then `.env`
  themselves, through the mounted code. Neither file ever reaches an image.
- Production images are multi-stage, run as a non-root user, and start the server so it receives `SIGTERM`.

## References

Read the file matching the task:

| File                    | Read it to                                                                              |
| ----------------------- | --------------------------------------------------------------------------------------- |
| `references/compose.md` | lay out compose files, ports, healthchecks, environment files, volumes, images, backups |
| `references/images.md`  | write a Dockerfile, pick a base image, handle stop signals, write the `.dockerignore`   |
| `references/nginx.md`   | set up the single origin: Vite's proxy in development, `nginx.conf` in production       |
