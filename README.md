# claude-plugins

Claude Code plugins holding my conventions, so that every project and every machine shares the same defaults.

| Plugin               | Covers                                                                         |
| -------------------- | ------------------------------------------------------------------------------ |
| `api-conventions`    | Clean Architecture TypeScript API: domain, use cases, routes, config and logs  |
| `web-conventions`    | React frontend: entities, mappers, services and HTTP client                    |
| `docker-conventions` | Docker Compose setup: images, compose files, ports, healthchecks and nginx     |

They are defaults: a project's own `CLAUDE.md` and documentation take precedence over them.

## Install

Once per machine, in a Claude Code session:

```
/plugin marketplace add antoine-navette/claude-plugins
/plugin install api-conventions@antoine-navette
```

Install only the plugins a project needs. Auto-update is off by default: enable it under **Marketplaces** in `/plugin`,
or run `/plugin marketplace update antoine-navette` to get new versions.

To share them with everyone working on a repository, run this once there and commit the `.claude/settings.json` it
writes:

```
claude plugin marketplace add antoine-navette/claude-plugins --scope project
```

## Develop

Add this folder as a local marketplace: plugins are then read in place, and edits apply at the next session start or
after `/reload-plugins`.

```
claude plugin marketplace add ./claude-plugins
claude plugin validate ./claude-plugins
```

The plugins carry no `version`, so that installs follow the latest commit.
