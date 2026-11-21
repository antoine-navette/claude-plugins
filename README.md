# claude-plugins

Claude Code plugins holding my conventions, so that every project and every machine shares the same defaults.

| Plugin               | Covers                                                                        |
| -------------------- | ----------------------------------------------------------------------------- |
| `api-conventions`    | Clean Architecture TypeScript API: domain, use cases, routes, config and logs |
| `web-conventions`    | React frontend: structure, pages, services, entities, session and errors      |
| `docker-conventions` | Docker Compose setup: images, compose files, ports, healthchecks and nginx    |

They are defaults: a project's own `CLAUDE.md` and documentation take precedence over them.

## Add them to a project

Declare them in the project itself, like dependencies in a `package.json`: the project says which conventions apply,
and anyone who opens it, on any machine, is offered to install them.

From the project's root, register the marketplace once, then enable each plugin the project needs:

```sh
claude plugin marketplace add antoine-navette/claude-plugins --scope project
claude plugin install api-conventions@antoine-navette --scope project
claude plugin install web-conventions@antoine-navette --scope project
```

Both commands write to `.claude/settings.json`, which is committed: the marketplace entry says where to find the
plugins, and each install says which ones to enable. Once someone trusts the folder, Claude Code registers the
marketplace and offers to install those plugins.

To use a plugin in every project of a machine instead, run the same commands with `--scope user`, or without
`--scope` in a session (`/plugin marketplace add`, `/plugin install`), which asks for the scope.

## Get updates

Auto-update is off by default for this marketplace. Turn it on once per machine under **Marketplaces** in `/plugin`,
or run `/plugin marketplace update antoine-navette` to get new versions.

The plugins carry no `version`, so installs follow the latest commit.

## Develop

Add this folder as a local marketplace: plugins are then read in place, and edits apply at the next session start or
after `/reload-plugins`.

```sh
claude plugin marketplace add /path/to/claude-plugins
claude plugin validate /path/to/claude-plugins
```
