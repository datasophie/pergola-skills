# Pergola CLI operations

Tasks beyond the deploy workflow in SKILL.md. Run
`pergola <verb> <noun> --help` for exact, current flags.

## Contents
- Private repository access
- Run commands in a component
- Reach a component's port locally
- Logs
- Component and stage lifecycle
- Backups
- Notifications, vulnerabilities and cost
- MCP server setup

## Private repository access

A project bound to a private repo needs read credentials. Pergola stores them
per project. Pick the type by clone URL:

- **SSH key pair** for `git@…` SSH clone URLs. Pergola generates and holds the
  key, and you add the returned **public key** to your git provider's deploy
  keys.

  ```sh
  pergola create ssh -p my-project   # generate; prints public key + fingerprint
  pergola list ssh   -p my-project   # re-print the public key + fingerprint
  pergola delete ssh -p my-project
  ```

- **Token (PAT)** for HTTPS clone URLs. You supply the token. It is write-only
  from the CLI's side, so `list pat` shows only its name.

  ```sh
  pergola create pat -p my-project --name my-token --token "$MY_PAT_SECRET"   # never the literal value
  pergola list   pat -p my-project   # shows the configured name only, never the value
  pergola delete pat -p my-project
  ```

`create pat` and `create ssh` replace any existing credential of that type for
the project, and `delete pat` and `delete ssh` remove it. Confirm with the user
first, naming the project. After adding a key or token, push a build to
confirm Pergola can clone.

## Run commands in a component

Everything after `--` runs inside the component:

```sh
pergola exec my-component -p my-project -s dev -- ls -la   # one-off command
pergola exec my-component -p my-project -s dev -- bash     # interactive shell, TTY detected automatically
```

Commands that write or delete inside the container are mutations. Tip: alias a
frequently used tool, e.g.
`alias myapp="pergola exec my-component -p my-project -s dev -- myapp"`.

## Reach a component's port locally

`pergola local-connect` (alias `port-forward`) needs a running component that
exposes at least one port:

```sh
pergola local-connect my-component 28080:8080 -p my-project -s dev   # local 28080 -> component 8080
pergola local-connect my-component 8080 -p my-project -s dev         # random local port -> 8080
pergola local-connect my-component -p my-project -s dev              # random local port -> the single exposed port
pergola local-connect my-component 192.168.128.42:28080:8080 -p my-project -s dev   # bind a local interface
```

Keep the command running. It prints the local `ip:port` to connect to. If a
local port is taken, choose another one (e.g. `19119:9119`). Use it for
dashboards or APIs without a public ingress, or to reach a stage database from
local tooling.

## Logs

```sh
pergola logs build my-build -p my-project --query my-keyword --since 5m
pergola logs component my-component -p my-project -s dev --follow
pergola logs stage dev -p my-project --since-time "2022-03-29T14:35Z"
```

- `--query` filters by a case-insensitive search term.
- `--since` takes a relative duration (`5s`, `2m`, `3h`, `5d3h7m10s`).
- `--since-time` takes a timestamp such as `2023-01-09` or `2023-01-09 17:18`,
  optionally with seconds, fractions or a timezone. It excludes `--since`.
- `--follow` streams until interrupted, so don't use it from an agent's shell.

## Component and stage lifecycle

```sh
pergola stop component my-component -p my-project -s dev          # graceful
pergola stop component my-component -p my-project -s dev --kill   # immediate
pergola start component my-component -p my-project -s dev
pergola start component cron-job -p my-project -s dev --now       # run a scheduled job now
pergola restart component my-component -p my-project -s dev
pergola suspend stage dev -p my-project   # stop all components, disable scheduling
pergola resume stage dev -p my-project
```

For a scheduled (cron) component, `stop` suspends scheduling, `start` resumes
it, and `--now` triggers an immediate run.

`stop component --kill` and `suspend stage` interrupt live traffic. Confirm
with the user before running them, naming the project and the component or
stage.

## Backups

A backup covers all persistent storage on a stage. A restore replaces the
stage's current data with the snapshot, so confirm with the user first, naming
the stage and the backup.

```sh
pergola create backup -p my-project -s dev --display-name "Restore Point"
pergola list backup -p my-project -s dev                 # the restore argument is the technical ID listed here
pergola suspend stage dev -p my-project                  # restore needs a suspended stage
pergola restore backup <backup-id> -p my-project -s dev
pergola resume stage dev -p my-project
```

## Notifications, vulnerabilities and cost

```sh
pergola notifications project my-project --since 1h
pergola list vulnerabilities master_b123 -p my-project                              # per-component summary
pergola list vulnerabilities master_b123 -p my-project -c my-component              # full component report
pergola list vulnerabilities master_b123 -p my-project -c my-component --id <vulnerability-id>
```

Start with the summary. A full component report can be very large.

Costs are visible only in the Pergola web UI (the cost badge on the project
start page). There are no CLI commands for cost tracking.

## MCP server setup

`pergola mcp serve` exposes the CLI as a Model Context Protocol server over
stdio (JSON-RPC). By default the active CLI profile must already be logged in.
The server then authenticates non-interactively and refreshes its token.
Configure an MCP client to launch it with `command: pergola`,
`args: [mcp, serve]`. It exposes most of the platform as typed tools (~80):
reads, mutations flagged destructive, exec/file/port-forward access to running
components, server-side `wait_for_build`/`wait_for_release` primitives, and
the `pergola_init` pergolizer playbook. The `pergola-mcp` skill is its
operating manual.

**Pre-configured access key** On a machine where nobody can log in, set
`PERGOLA_ACCESS_KEY=<key-id>:<secret>` in the client's `env` for the server.
That is preferred over `--access-key` in `args`, which other local users may
see. The server then sends the key only to the active profile's endpoint
(https unless localhost) and never falls back to the login session: a rejected
key needs a fixed client config plus a server restart, not `pergola login`. A
key belongs in the client config only, never in the chat.
