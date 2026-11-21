# Images

Each application (`api/`, `web/`) has two Dockerfiles: `Dockerfile` for production, `Dockerfile.dev` for development.
The two have almost nothing in common (no code copied in development, a different command), so two short files read
better than one with several targets.

## Base images

- **Official images, in a slim variant, pinned to a major version** (`node:24-slim`, `oven/bun:1.4`, `nginx:alpine`,
  `mongo:8.0`), the same major as the one the CI uses, so that development, CI and production run the same runtime.
- **Available for the server's architecture**: check that every base image exists for it (ARM64 on a Raspberry Pi).

## Development

```dockerfile
FROM node:24-slim

WORKDIR /app

USER node
EXPOSE 3000
CMD ["sh", "-c", "npm install && npm run migrations:run:dev && exec node_modules/.bin/tsx watch --env-file-if-exists=.env.example --env-file-if-exists=.env src/index.ts"]
```

- **No code is copied**: it is mounted by Compose (see `compose.md`), so a change reloads without rebuilding.
- **At startup, the container installs the dependencies, applies the pending migrations, then starts the development
  server**: a fresh clone or a changed lockfile needs nothing more than `docker compose up`.
- **The server is started with `exec`**, which replaces the shell instead of running as its child, so it receives the
  stop signals (see below).
- **It runs as the image's non-root user** (`node`, `bun`), so the files it writes in the mounted folder (dependencies,
  generated files) belong to a regular user on the host.

## Production

```dockerfile
FROM node:24-slim AS build

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build


FROM node:24-slim

WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist

USER node
EXPOSE 3000
CMD ["node", "dist/src/index.js"]
```

- **Multi-stage**: the build stage installs everything and compiles, the final stage only installs the production
  dependencies and copies the build. Dev tools, sources and tests never reach the final image.
- **The lockfile is copied and installed before the code**, so Docker reuses the dependencies layer as long as the
  lockfile doesn't change.
- **It runs as the image's non-root user.**
- **The web image** builds the frontend in a first stage, then copies only the built files and `nginx.conf` into an
  nginx image (see `nginx.md`).
- **Images carry a label naming the project** (`app=<project>`, set by the Dockerfile's `LABEL` or the CI's build), so
  the server can remove its unused images without touching other projects': `docker image prune -af --filter
  "label=app=<project>"`.

## The server must receive the stop signals

`docker stop` sends `SIGTERM` to the container's main process, waits 10 seconds, then kills it. The server only shuts
down gracefully (finish the requests, close the connections, flush the logs) if that signal reaches it:

- **Use the exec form** (`CMD ["node", "dist/src/index.js"]`), never the shell form (`CMD node dist/src/index.js`),
  which runs the server as a child of a shell that doesn't forward the signal.
- **With Node, start `node` directly**, not `npm start`: npm runs the script through a shell, and doesn't reliably
  forward the signal. One-shot commands (`npm run migrations:run`) can go through npm, since nothing stops them.
- **With Bun, `bun run start` forwards the signals**, so the script can be used.
- **To check it**: `docker compose stop api` must log the graceful shutdown and return within a second. Taking 10
  seconds means the signal didn't reach the server, which was killed.

## `.dockerignore`

```
node_modules/
dist/
coverage/
.env
# Only the development scripts read these files: in production, every value comes from the env_file of Docker Compose,
# and a missing variable must stop the API instead of silently taking its example value
.env.example
```

Nothing built on the host, and no environment file, reaches the image.
