---
name: pergola-cli
description: >-
  Operates the Pergola CLI (`pergola`) for the Pergola container deployment
  platform. Full CLI manual: projects, stages, builds, releases, components,
  config-data, logs, notifications, vulnerabilities, exec and port forwarding,
  lifecycle, backups and `pergola mcp serve` setup, plus the CLI-only
  `pergola login`, access keys, CLI profiles and private-repo credentials. Use
  when the user asks for the CLI, when no `mcp__pergola__*` tools are
  available, or for those CLI-only tasks. Otherwise prefer the pergola-mcp
  skill.
---

# Pergola CLI

`pergola` (CLI v2) drives Pergola, a container platform that runs web apps and
services on a high-availability, auto-scaling cluster with managed TLS.
Grammar is verb then noun: `pergola <verb> <noun> [args] [flags]`. Every
command has `--help`; run it before relying on a flag you are unsure about.

If `pergola` is missing or outdated, point the user to the official installer
at https://get.pergo.la/cli and let them run it themselves rather than running
an install script on their behalf.

## Mental model

- A **project** is bound to one git repository whose root holds the manifest
  (`pergola.yaml`, `pergola.yml` or `pergola.json`, version `v1`; see the
  pergola-manifest skill).
- A **build** is the image set built from a commit, named `<branch>_b<n>`.
- A **stage** is an environment typed `dev`, `qa` or `prod`.
- A **release** deploys a build and/or a config onto a stage. Activating it is
  what runs the app.
- A **component** is one of the manifest's services or jobs, running on a
  stage. Components link to each other only through the manifest.
- A **config** on a stage holds **config-data**: env vars and files, including
  secrets. The default config is conventionally named `default`.
- Enterprise only: **workload identities** (AWS IAM, Azure, GCP) and
  **outposts** (self-managed infrastructure for stages).

## Flags and conventions

| Flag | Meaning |
|------|---------|
| `-p, --project <name>` | Target project. Optional when the active CLI profile has a default project. |
| `-s, --stage <name>` | Target stage, required for stage-scoped commands. |
| `-o, --output <fmt>` | `table` (default), `json` or `yaml`. Use `json` when parsing output. |
| `--debug` | Verbose debug output. |
| `--access-key <key-id>:<secret>` | Authenticate non-interactively for this invocation. |

## Login, access keys and profiles

```sh
pergola login --endpoint https://api.pergola.cloud   # browser device flow; the user runs it
pergola --access-key "$MY_ACCESS_KEY" list project     # non-interactive, works on any command
pergola set cli-config --config my-cli-config --default-project my-project --endpoint <endpoint-uri>
pergola use cli-config my-cli-config
pergola list cli-config                                # the active profile is marked
```

- `--endpoint` defaults to the active profile's endpoint. After login the
  token is stored and refreshed automatically.
- `pergola create access-key` shows the secret only once. A user can hold at
  most 2 access keys, and a key carries all of its creator's permissions.
  Rotate by creating a new key, updating its users, then disabling or deleting
  the old one (`list`, `enable`, `disable`, `delete access-key`). Rotate
  immediately if a key leaks.

## Secrets

Secret values (tokens, passwords, access keys, secret config-data) must never
end up in the shell history or the context. Have the user put the value in an
environment variable or a file, then pass it by reference:

```sh
pergola create pat -p my-project --name my-token --token "$MY_PAT_SECRET"
pergola create pat -p my-project --name my-token --token "$(cat /tmp/pat_secret)"
```

The same applies to `--access-key` and `add config-data --env`. Never put the
value itself in a command or its output, and keep it out of the chat: the
user sets it, you only reference it.

## Deploy workflow

```sh
pergola create project my-project --git-url git@server:path/to/my-repo.git --display-name "My Project"
pergola push build -p my-project                 # every branch with new commits; limit with --branch or --commit
pergola list build -p my-project                 # build names, e.g. main_b12
pergola create stage dev -p my-project --type dev --display-name "Dev"
pergola add config-data default -p my-project -s dev --env SOME_KEY=some-value   # bind before the first release
pergola push release -p my-project -s dev -b main_b12 -c default --when-ready
```

- Private repositories need read credentials first, see
  [references/operations.md](references/operations.md). SSH clone URLs
  (`git@server:path`) are recommended.
- `--file <path>` on `add config-data` stores a file; its basename becomes the
  key.
- `push release` needs at least one of `-b` and `-c`. Without `-b` the stage's
  last deployed build is reused. Without `-c` the active config is used, if
  there is one.
- `--when-ready` needs `-b`. It waits up to 30 minutes for that build to
  succeed and pushes nothing if the build fails.

## Done means verified

| Step | Success | On failure |
|---|---|---|
| Build | `--when-ready` prints "Build '<build>' succeeded", or `pergola list build` shows `succeeded` | Output ends in "release not pushed". The exit code is still 0 in CLI v2.3, so read the output. Run `pergola logs build <build> -p <project>`, fix, push a new build |
| Release | `pergola list release -p <project> -s <stage>` shows the new release `deployed` and active, and `pergola list component` shows every component `deployed` | Read `pergola logs component <component> -p <project> -s <stage>` for each `failed` component before retrying |
| Still rolling out | | Deployments take a few minutes, a first release may wait for TLS. Re-check at least 60 seconds apart, report the state, and ask whether to keep waiting |

Add `-o json` to read the `status` fields. Never report a deploy as done from
the push message alone, because it only means "accepted".

## Confirm before deleting, restoring, suspending, or changing access

Before any of these, tell the user what will happen and get an explicit yes,
naming the target:

- **Delete or remove:** every `delete` and `remove` command, including access
  keys and git credentials.
- **Replace credentials:** `create ssh` and `create pat` replace the project's
  existing credential of that type. `disable access-key` cuts off its users.
- **Restore:** `restore backup` replaces the stage's current data.
- **Suspend or kill:** `suspend stage` stops all components on the stage, and
  `stop component --kill` stops one immediately.
- **Member or role changes:** `add member` and `remove member`, above all for
  the owner role.

A user request that already names the operation and its target counts as that
yes. Builds and releases follow the user's request as usual.

## Gotchas

- **The manifest lives in the build** Local `pergola.yaml` edits change
  nothing until they are pushed to the git remote and built with
  `push build`.
- **`push build` may create several builds or none** Scope it with
  `--branch` or `--commit` (mutually exclusive), above all with `--force`.
- **Config changes need a release** Config-data edits don't reach running
  components until a new release. A config-only release may keep unchanged
  components running, so apps may need a reload or `restart component`.
- **The container filesystem is ephemeral** Durable state belongs in a
  manifest `storage` mount or in config-data. System packages belong in the
  Dockerfile, not in a running container.
- **An ingress host belongs to a component name** A release that moves a host
  to a differently named component fails with "is already in use". Keep the
  owning component's name or pick another host.
- **`delete` archives projects and stages** `restore` brings them back, and an
  archived stage keeps its name, so `create stage` with that name fails with
  "already exists or is archived". `delete config` is permanent, with no
  backup. `remove` strips entries such as config-data keys, members or rules.
- **Suspended stages defer releases** A release pushed to a suspended stage
  becomes active after `resume stage`. `@release` jobs may then need
  `start component --now`.
- **`pergola login` needs the user** It is an interactive browser flow, and
  run from an agent's shell it just waits. Ask the user to run it.
- **`--with-values` prints secrets** from `list config-data`. Use it only when
  the user needs the values, and don't echo them back.
- **Build failures** usually reproduce locally with `docker build .` in the
  repository root.
- **`exec` and `local-connect` need a running component**, and
  `local-connect` also needs an exposed port.

## More

- **Operations** (private-repo credentials, exec, port forwarding, logs,
  lifecycle, backups, notifications, vulnerabilities, cost, MCP server setup):
  [references/operations.md](references/operations.md).
- **Every command and flag:**
  [references/command-reference.md](references/command-reference.md), and the
  [CLI online documentation](https://docs.pergola.cloud/docs/cli.md).
- **When `--help` or the CLI's behavior contradicts this skill**, trust the
  CLI and tell the user which statement is outdated, so the skill can be
  corrected.
