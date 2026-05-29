---
name: pergola-cli
description: >-
  Operate the Pergola CLI (`pergola`), the command-line tool for the Pergola
  container deployment platform, for tasks that genuinely require the shell
  binary. Use this skill when the user explicitly asks for the `pergola-cli` or for CLI-only concerns: `pergola login` and auth flows,
  access-key management, CLI profiles (`pergola set/use/list cli-config`),
  private-repo credentials (`pergola create ssh` / `pergola create pat`). When MCP tools (`mcp__pergola__*`) are available in this session,
  defer to the `pergola-mcp` skill for read, inspect, deploy, and mutate
  tasks (project/stage/component/build/release/config-data operations,
  `exec`, logs, lifecycle, backups) — MCP is the preferred surface there.
---

# Pergola CLI

`pergola` is the command-line interface to the **Pergola** platform — a
container-based cloud that deploys and runs server/web applications on a
high-availability, auto-scaling cluster with managed TLS, without server or
cluster setup. This skill is the operating manual for the CLI.

Pergola CLI v2 is the current major version. For install or upgrade on
Linux/macOS, use the official installer:

```sh
curl -fsSL https://get.pergo.la/cli/latest/install.sh | bash
```

When acting for a user, prefer the CLI over guessing. Every command supports
`--help`; run `pergola <command> <subcommand> --help` to confirm flags before
running anything you are unsure about. Mutating actions (push, delete, stop,
suspend, restart) change live infrastructure — confirm intent before running
them unless the user has clearly authorized the action.

## Mental model

Pergola resources form a hierarchy. Understand it before running commands:

- **Project** — top-level unit, bound to one git repository. A project root must
  contain a **manifest** (`pergola.yaml`/`pergola.yml`/`pergola.json`) describing
  the application stack (comparable to Docker Compose, optimized for HA). The
  manifest is versioned; the supported version is `v1`. Validate it with
  `pergola validate manifest <path>` and use
  `https://docs.pergola.cloud/pergola_project_manifest_spec.yaml` as the schema
  reference when writing or debugging it.
- **Build** — a container image set compiled from a git commit/branch of the
  project. Builds are named like `master_b123` (`<branch>_b<n>`).
- **Stage** — a deployment environment within a project, typed `dev`, `qa`, or
  `prod`. Components run on stages.
- **Release** — a deployment of a specific **build** and/or **config** onto a
  stage. Activating a release is what actually runs the app.
- **Component** — an individual running service or scheduled job on a stage,
  defined by the manifest. **Component linking** (connecting one component to
  another) is managed strictly via the manifest using `component-ref` in the
  `env` section.
- **Config / config-data** — named configuration on a stage holding key/value
  entries: environment variables and files (including secrets). The default
  config is conventionally named `default`.
- **Workload Identity** (Enterprise only) — managed identities (AWS IAM, Azure,
  GCP) for secure cloud resource access.
- **Outpost** (Enterprise only) — self-managed infrastructure for running
  Pergola stages.
- **Access key** — non-interactive API credential (`<key-id>:<secret>`).
- **CLI config (`cli-config`)** — a local CLI profile holding an API endpoint
  and an optional default project. Multiple profiles can be stored; one is
  active.

## Setup & authentication

**Interactive login (OIDC device flow):**

```sh
pergola login --endpoint https://api.pergola.cloud
```

`--endpoint` is the Pergola API endpoint URI. If omitted, the active CLI
profile's endpoint is used. After a successful login the token is stored and
refreshed automatically.

**Non-interactive (access key)** — pass on any command via the global flag:

```sh
pergola --access-key '<key-id>:<secret>' list project
```

Manage access keys with `pergola create access-key` (the secret is shown only
once), `pergola list access-key`, `pergola enable access-key`,
`pergola disable access-key`, `pergola delete access-key`.

**Access key constraints & rotation:**
- **Limit:** Maximum of **2 access keys** per user.
- **Rotation workflow:** To rotate a key, create a new one, update your systems,
  then `disable` or `delete` the old one.
- **Security:** Access keys inherit all permissions of the user who created
  them. Always rotate keys immediately if compromised.

**CLI profiles** let you target different endpoints/projects:

```sh
# create or update a profile (creates it if the name is new)
pergola set cli-config --config my-cli-config \
  --default-project my-project \
  --endpoint https://api.mydomain.pergola.cloud/v1

pergola list cli-config          # active profile is marked
pergola use cli-config my-cli-config   # switch active profile
```

When a profile has a `--default-project`, the `-p/--project` flag becomes
optional on commands that need a project; the default is used automatically.

## Private repository access (git credentials)

A project bound to a private repo needs read credentials. Pergola stores them
per project, two mutually relevant options depending on the clone URL:

- **SSH key pair** (for `git@…` SSH clone URLs). Pergola generates and holds the
  key; you add the returned **public key** to your git provider's deploy keys.

  ```sh
  pergola create ssh -p my-project   # generate; prints public key + fingerprint
  pergola list ssh   -p my-project   # re-print the public key + fingerprint
  pergola delete ssh -p my-project
  ```

- **Personal access token** (for HTTPS clone URLs). You supply the token; it is
  write-only from the CLI's perspective — `list pat` shows only its name.

  ```sh
  pergola create pat -p my-project --name my-token --token <secret>
  pergola list   pat -p my-project   # shows the configured name only, never the value
  pergola delete pat -p my-project
  ```

`create pat`/`create ssh` replace any existing credential of that type for the
project. After adding a key/token, push a build to confirm Pergola can clone.

## Global flags & conventions

Available on (almost) every command:

| Flag | Meaning |
|------|---------|
| `-p, --project <name>` | Target project. Optional if a default project is set in the active CLI profile; otherwise required. |
| `-s, --stage <name>` | Target stage. Required for stage-scoped commands. |
| `-o, --output <fmt>` | Output format: `table` (default), `json`, or `yaml`. Use `json`/`yaml` when scripting or parsing. |
| `--debug` | Verbose debug output. |
| `--access-key <key-id>:<secret>` | Authenticate non-interactively for this invocation. |
| `-v, --version` | (root only) Print CLI version. |

Grammar is **verb → noun**: `pergola <verb> <resource> [args] [flags]`, e.g.
`pergola create stage`, `pergola list component`, `pergola push release`.

## Core deployment lifecycle (worked example)

This is the canonical path from empty to running. Replace placeholder names.

```sh
# 1. Create the project from its git repository
#    Note: For private repositories, Pergola needs read access.
#    SSH clone URLs (git@server:path) are recommended.
pergola create project my-project \
  --git-url git@server:path/to/my-repo.git \
  --display-name "My Project"

# 2. Trigger builds (runs in the background, takes a few minutes)
pergola push build -p my-project
#    optional: build a specific branch, a specific commit on it, or force despite no new commits
#    pergola push build -p my-project --branch my-branch --commit 9f3a1c2 --force
#    --commit must be a 7-40 char lowercase hex SHA on the selected --branch

# 3. Wait until the build is ready — check its status
pergola list build -p my-project

# 4. Create a stage (type: dev | qa | prod)
pergola create stage dev -p my-project --type dev --display-name "Dev"

# 5. Bind configuration (env vars / files / secrets) BEFORE first release
pergola add config-data default -p my-project -s dev \
  --env SOME_KEY=some-value \
  --env ANOTHER_KEY=another-value
#    files: the file's basename becomes the key, its content the value
#    pergola add config-data default -p my-project -s dev --file /path/to/file

# 6. Push the release: deploy a build and a config onto the stage
pergola push release -p my-project -s dev -b master_b123 -c default

# 7. Verify what is running
pergola list component -p my-project -s dev
```

`push release` requires **at least one** of `-b/--build` or `-c/--config`. If
`-b` is omitted, the last deployed build on the stage is reused; if `-c` is
omitted, the current active config is used if one exists, otherwise a build-only
release is created. Deployment happens in the background and may take a few
minutes. A first release may also wait on infrastructure such as ingress and TLS
certificate provisioning.

## Generic operating patterns

These patterns recur across applications deployed on Pergola:

**Bind secrets before the first release.** Any value an app needs (API keys,
DB URLs, tokens) is a config-data entry. Generate secrets locally and bind
them, then release. Inspect entries (optionally revealing values — may expose
sensitive data):

```sh
pergola list config-data default -p my-project -s dev               # keys only
pergola list config-data default -p my-project -s dev --with-values # keys + values
```

**Re-release after changing config.** Editing config-data does not
automatically propagate to running components. Push a new release (same build
is fine — just supply `-c`) to apply config changes. A config-only release may
keep unchanged components running; applications should reload changed config
files themselves, or you may need to restart affected components after the
release.

**The container filesystem is ephemeral.** Every release or restart wipes
anything written outside the app's persistent storage. Durable state belongs in
persistent storage (declared in the manifest) or in config-data. For durable
system dependencies or image changes, change the repository's Dockerfile and
rebuild — do not rely on packages installed manually inside a running
container.

**Run commands inside a running component** with `pergola exec`. Everything
after `--` runs inside the component:

```sh
# one-off command
pergola exec my-component -p my-project -s dev -- ls -la

# interactive shell (TTY is detected automatically)
pergola exec my-component -p my-project -s dev -- bash
```

Tip: alias a frequently-used component's CLI, e.g.
`alias myapp="pergola exec my-component -p my-project -s dev -- myapp"`, so its
subcommands run seamlessly inside the stage.

**Reach a component's port locally** with `pergola local-connect` (alias
`port-forward`). The component must expose at least one port. Forms:

```sh
# local 28080 -> component 8080
pergola local-connect my-component 28080:8080 -p my-project -s dev

# random local port -> component 8080
pergola local-connect my-component 8080 -p my-project -s dev

# random local port -> the component's single exposed port
pergola local-connect my-component -p my-project -s dev

# bind a specific local interface: <host>:<localPort>:<remotePort>
pergola local-connect my-component 192.168.128.42:28080:8080 -p my-project -s dev
```

Keep the command running; it prints the local `ip:port` to connect to. If a
local port is already in use, choose a different local port (e.g.
`19119:9119`). Use this for dashboards/APIs with no public ingress, or to reach
a stage database from local tooling.

## Day-2 operations

**Logs** — for builds, components, or stages. Filter and stream:

```sh
pergola logs build my-build -p my-project --query my-keyword --since 5m
pergola logs component my-component -p my-project -s dev --follow
pergola logs stage dev -p my-project --since-time "2022-03-29T14:35Z"
```

- `--query` filters by a case-insensitive search term.
- `--since` takes a relative duration (`5s`, `2m`, `3h`, `5d3h7m10s`).
- `--since-time` takes a timestamp such as `2023-01-09`, `2023-01-09 17:18`,
  or a more precise timestamp with seconds, fractional seconds, or timezone.
  `--since` and `--since-time` are mutually exclusive.
- `--follow` streams new log lines until interrupted.

**Component lifecycle:**

```sh
pergola stop component my-component -p my-project -s dev          # graceful
pergola stop component my-component -p my-project -s dev --kill   # immediate
pergola start component my-component -p my-project -s dev
pergola start component cron-job -p my-project -s dev --now       # run scheduled job now
pergola restart component my-component -p my-project -s dev
```

For a scheduled (cron) component, `stop` suspends scheduling; `start` resumes
it; `--now` triggers an immediate run.

**Stage lifecycle:**

```sh
pergola suspend stage my-stage -p my-project   # stop all components, disable scheduling
pergola resume stage my-stage -p my-project    # bring it back
```

**Backups** of all persistent storage on a stage:

```sh
pergola create backup -p my-project -s dev --display-name "Restore Point"
pergola list backup -p my-project -s dev
pergola restore backup <backup-id> -p my-project -s dev
```

The backup argument is the technical ID returned by `list backup`. Suspend the
stage before restoring a backup.

**Notifications & security:**

```sh
pergola notifications project my-project --since 1h
pergola list vulnerabilities master_b123 -p my-project
```

**Cost Control:**
Pergola costs are currently monitored and analyzed exclusively via the
**Pergola Web UI** (look for the cost badge on the Project start page). There
are no CLI commands for cost tracking.

## Full command reference

For the exhaustive, grouped list of every command and its flags, see
[references/command-reference.md](references/command-reference.md). Always run
`pergola <verb> <noun> --help` to confirm exact, current flags before relying on
anything you are unsure about.

When in doubt, also see [CLI online documentation](https://docs.pergola.cloud/docs/cli.md).

## MCP server

`pergola mcp serve` exposes the CLI as a Model Context Protocol server over
stdio (JSON-RPC). The active CLI profile must already be logged in; the server
authenticates non-interactively and refreshes its token. Configure an MCP
client to launch it with `command: pergola`, `args: [mcp, serve]`. It provides
read-only tools (list projects/stages/components, component status, bounded log
snapshots, validate manifest) and mutating tools flagged destructive
(push release, restart/start/stop component, suspend/resume stage).

## Gotchas

- **Build before release.** A release needs a ready build. Check `list build`
  before `push release`; freshly pushed builds take a few minutes.
- **`push build` may create multiple builds or none.** By default it checks all
  branches with valid manifests for new commits. Use `--branch` or `--commit` to limit scope,
  especially with `--force`, to avoid unnecessary builds.
- **`push release` needs `-b` or `-c`** (or both). Neither given → error.
- **Re-release to apply config changes** — config-data edits do not propagate
  on their own. A config-only release may still require an app-level reload or
  component restart.
- **Max 2 access keys per user.** You cannot create more than two access keys
  simultaneously. Use the manual rotation workflow to replace old keys.
- **Docker build failures:** If a build fails in Pergola, try running
  `docker build .` locally in your project root to reproduce and debug the
  environment.
- **Private Git repos:** Ensure Pergola has read access. For SSH, generate the
  key with `pergola create ssh` (or retrieve it via `pergola list ssh`/the UI)
  and add the public key to your Git provider's deploy keys; for HTTPS, store a
  token with `pergola create pat`. See "Private repository access" above.
- **Suspended-stage releases are deferred.** A release pushed to a suspended
  stage becomes active after `resume stage`; scheduled `@release` components
  may need a manual `start component --now`.
- **`delete` archives, it does not hard-delete.** Projects/stages are marked for
  deletion/archival and can be brought back with `restore`. **`remove`** strips
  entries (config-data keys, members, rules) and is not the same as `delete`.
- **Deleting configs is permanent.** `delete config` removes the configuration
  and underlying data; docs warn there are no backups for deleted configs.
- **`--with-values` can expose secrets** in plaintext; use deliberately.
- **`exec`/`local-connect` need a *running* component** with (for local-connect)
  at least one exposed port.
- **Deployments and builds run in the background** — success messages mean
  "accepted/started", not "finished". Verify with `list build` /
  `list component` or `logs`.
