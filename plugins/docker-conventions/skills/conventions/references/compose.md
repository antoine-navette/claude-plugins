# Docker Compose

The project only runs through Docker Compose, in development as in production. This removes every "it works on my
machine" difference, and lets internal values (ports, service addresses) be hardcoded instead of configured.

## Layered files

```
compose.yaml           # what both environments share: the database, its volume and healthcheck, startup order
compose.override.yaml  # development: Dockerfile.dev, code mounted as volumes, dev-only services, defaults
compose.prod.yaml      # production: production images, env_file, restart policy, backups
compose.local.yaml     # optional personal tweaks, never committed
```

- **The root `.env` picks the combination** with `COMPOSE_FILE`, and names the project with `COMPOSE_PROJECT_NAME`,
  which prefixes containers, networks and volumes (so a staging and a production copy on one server don't collide).
  Without it, Compose takes `compose.yaml` and `compose.override.yaml`, that is, development.
- **The root `.env.example` is the development setup**, copied to `.env` on a server and edited there:
  `COMPOSE_FILE="compose.yaml:compose.prod.yaml"`, real passwords, the host ports to use.
- **The root `.env` only holds what Compose itself reads**: the files, the project name, the host ports, the
  database's credentials. Each service's own settings live in its own `.env` (see below).
- **Development needs no configuration**: `compose.override.yaml` gives every variable a default (`${WEB_PORT:-5173}`),
  so a fresh clone runs with `docker compose up`. Production gives none, so a missing variable fails instead of
  silently taking a development value.

## Ports

```yaml
web:
  # Not reachable from the local network: localhost is also the only secure context over plain HTTP
  ports:
    - "127.0.0.1:${WEB_PORT:-5173}:5173"
```

- **Internal ports are hardcoded** in the code and the images (the API on 3000, Vite on 5173, nginx on 80): they never
  change inside the network Compose creates. Only the host side of a published port is configurable.
- **Every published port is bound to `127.0.0.1`**, in development as in production. Docker publishes on every
  interface by default, and its iptables rules bypass the host firewall: a port published without an address is
  reachable from the network even if the firewall blocks it. Nothing needs to reach these services from the network:
  in production, a tunnel or a reverse proxy on the host reaches the web container, and the database is reached
  through an SSH tunnel.
- **Only what must be reached from the host is published**: the web container, the database for a GUI client, a
  development mail catcher's interface. The API is never published, the web container relays to it (see `nginx.md`).

## Healthchecks and startup order

```yaml
mongo:
  # Probed every second while starting, so the API starts as soon as Mongo is ready, then every 30 seconds once it
  # is (start_interval needs Docker Engine 25+). Failures during start_period don't count toward the retries
  healthcheck:
    test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping')"]
    start_period: 30s
    start_interval: 1s
    interval: 30s
    timeout: 5s
    retries: 3
```

- **Every dependency has a healthcheck**, probed every second while it starts (`start_interval`) and rarely once it
  runs: a fixed `interval: 5s` makes every startup wait up to 5 seconds for nothing.
- **Services wait for their dependencies to be healthy** (`depends_on` with `condition: service_healthy`), so the API
  never has to retry its first connection.
- **The `test` uses the exec form** (`["CMD", ...]`), which runs the command without a shell.

## Environment files

- **In production, each service gets its settings from its own `env_file`** (`api/.env`, `required: true`), written
  on the server and never committed.
- **In development, no `env_file`**: the code is mounted in the container, and the development scripts read
  `.env.example` then `.env` themselves (Node's `--env-file-if-exists`). Injecting the same files through Docker too
  would load them twice, by two different mechanisms.

## Volumes and restarts

- **Data lives in named volumes** (`mongo:/data/db`), which survive the containers being recreated.
- **Code is a bind mount in development only** (`./api:/app`), so changes reload without rebuilding. Dependencies are
  installed inside the container but land in the mounted folder, so the editor and the tools run on the host find
  them.
- **Production containers restart on their own** after a crash or a reboot (`restart: unless-stopped`).

## Production images

Two valid ways, the choice depending on the CI/CD:

- **Built on the server** (`build:` in `compose.prod.yaml`): no registry, the server needs to be able to build.
- **Pulled from a registry** (`image: registry/project-api:${IMAGE_TAG}`): built once in the CI, the same image going
  to staging then production. The tag is immutable (one per commit), never `latest`, so it always says what runs and a
  rollback is a redeploy of an older tag.

## Backups

A small `backup` service in `compose.prod.yaml` dumps the database:

- every night through a cron made for containers (`supercronic`, which passes the container's environment to the job
  and logs its output), and before every deployment's migrations, which stop if the dump fails;
- into a host folder next to the project (`./dumps`), outside the database's volume, so a `docker compose down -v`
  doesn't remove them;
- under a temporary name renamed once complete, so a failed dump never looks valid, keeping the most recent ones per
  label (`daily`, `deploy`).

Those backups stay on the server: they protect from a mistake or a failed migration, not from losing the server.
