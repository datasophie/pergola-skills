---
name: pergola-mcp
description: >-
  Operate Pergola via the `mcp__pergola__*` MCP tools — list and inspect
  projects, stages, components, builds, releases, configs; manage config-data;
  exec into and forward ports to running components; trigger and poll builds
  and releases; bootstrap (pergolize) a repo that has no `pergola.yaml` via
  `pergola_init`. Prefer the pergola mcp tools and this skill over `pergola-cli` whenever the
  `mcp__pergola__*` tools are available in the session — the MCP surface is typed and structured. Trigger on any
  Pergola task that involves reading state, mutating resources, deploying, or
  inspecting from inside an agent — except when the user explicitly asks for
  the `pergola-cli` or local profile setup.
---

# Pergola via MCP

This skill is the operating manual for driving **Pergola** through its MCP
tools (`mcp__pergola__*`). Pergola is a container-based cloud that runs
server/web applications on a high-availability, auto-scaling cluster with
managed TLS. The MCP surface gives an agent typed, structured access to the
same operations the `pergola` CLI exposes, without spawning shell processes.

**When MCP tools are available in this session, use them.** Only fall back to
the CLI (`pergola-cli` skill) for the cases listed under "When to fall back to
the CLI" at the bottom of this file.

## Mental model

Same resource hierarchy as the CLI. Internalise it before calling tools:

- **Project** — top-level unit, bound to one git repository. Project root
  contains a **manifest** (`pergola.yaml`) describing the stack.
- **Build** — container image set compiled from a git commit/branch. Named
  `<branch>_b<n>` (e.g. `master_b123`). Builds take minutes.
- **Stage** — deployment environment within a project, typed `dev`, `qa`, or
  `prod`. Components run on stages.
- **Release** — a deployment of a specific build and config onto a stage.
  Activating a release is what actually runs the app.
- **Component** — an individual running service or scheduled job on a stage,
  defined by the manifest.
- **Config / config-data** — named configuration on a stage holding key/value
  entries: environment variables and files (including secrets). Default is
  conventionally `default`.

For a deeper treatment see `pergola-cli` SKILL.md §"Mental model"

## Rule 0 — Bootstrap with `pergola_init` when there's no project yet

The orient-then-mutate flow below assumes a project already exists. When it
doesn't — `pergola_list_projects` shows no matching entry, or the repo the user
wants to deploy has no `pergola.yaml` manifest — make `pergola_init` the first
step rather than guessing a manifest or jumping to `pergola_create_project`.

`pergola_init` returns a step-by-step "pergolizer" playbook for **you**, the
calling agent, to execute locally: detect the stack, run a feasibility gate,
generate a `Dockerfile` if needed, author and `pergola_validate_manifest` the
`pergola.yaml`, then hand off for an opt-in build/deploy. The MCP server cannot
read the filesystem, so it returns guidance, not analysis — you do the file
work in the repo. Pass any context the user already gave (project name, stack,
path) via the optional `context` input to frame the playbook.

Once a validated `pergola.yaml` exists and the project is created, switch to
Rule 1 below.

## Rule 1 — Orient before mutating

Before any mutating call, build a current picture of the target. Treat this
as a required preflight on first contact with a project in a session, and
again whenever you are uncertain the state has not changed.

Preferred preflight:

0. `pergola_whoami()` — cheapest possible auth check; returns the endpoint,
   active profile, and authenticated user. Call it once on first contact in a
   session: a failure here means "not logged in", caught before any real work.
1. `pergola_orient_project(project=...)` — returns project metadata, active
   stages, deployed components on each stage, and the active release per stage.
   Per-stage errors are collected on the stage entry, so one broken stage does
   not prevent orienting on the rest of the project.

Fallback sequence when `pergola_orient_project` is unavailable, or when you
need a targeted refresh:

1. `pergola_list_projects` — confirm the project name actually exists and is
   not archived. Names look real but may be wrong.
2. `pergola_get_project` — pulls metadata, repo URL, status.
3. `pergola_list_stages` — confirm the target stage exists and is not
   archived/suspended.
4. `pergola_list_components` — confirm the target component exists on that
   stage and is currently deployed.
5. `pergola_get_current_release` (target stage) — know which build and config
   is live before changing anything.

**Why:** acting on stale assumptions (wrong project name, archived stage,
component that has been renamed in the manifest) is the most common
cross-session failure.

You may skip the preflight inside a single session once you have already
oriented and the user has not changed context. If you cross sessions or the user references a different project, redo it.

## Rule 2 — Async operations: use the wait primitive, then fall back to polling

Several Pergola operations return *accepted*, not *finished*:

- **Builds** — `pergola_push_build` starts work that typically takes 2–5
  minutes; a fresh project's first build can take longer. The push itself waits
  briefly (default 30s) for the build to register and returns its assigned
  name(s) in `result.build_names`.
- **Releases** — `pergola_push_release` records a release; deployment continues
  in the background until target components reach the new version.
- **Component lifecycle** — `pergola_restart_component`,
  `pergola_start_component`, `pergola_stop_component` request a transition;
  the component spends time in `starting` / `stopping` / `pending` first.

Treat each tool's success as "the request was accepted". To know it
*finished*, wait for state.

### Builds → `pergola_wait_for_build`

`pergola_push_build` returns once the build has registered: its name(s) are in
`result.build_names`. The backend first clones the repo and checks
buildability (~10–30s), so the push waits up to `discover_timeout_seconds`
(default 30) for that. Then call `pergola_wait_for_build` with the build name to
wait for completion. The server polls on your behalf and returns once the build
reaches a terminal status (`succeeded`, `failed`, `cancelled`) or the timeout
elapses (`timed_out: true`). Do **not** hand-roll a `pergola_get_build`
polling loop — the wait primitive exists so you don't have to spend turns on
it.

```text
push = pergola_push_build(project="my-project")
# Branch on push.result.outcome:
#   "scheduled"          → build names in result.build_names; use below
#   "rejected"           → hard failure; reason in result.notifications
#                          (e.g. B0005 "no manifest on branch")
#   "skipped_no_changes" → nothing new to build (e.g. B0003 skip notification,
#                          or an untargeted push found no changed branches)
#   "pending_unknown"    → a targeted push (branch= or commit=, mutually
#                          exclusive) registered nothing in the wait window;
#                          still scheduling or skipped — poll pergola_list_builds
#                          and re-check notifications; result.message explains
build_name = push["result"]["build_names"][0]

wait = pergola_wait_for_build(
  project="my-project",
  build=build_name,
  timeout_seconds=600,        # explicit budget; the server default is only 110s
                              # (to stay under MCP client call timeouts); clamped to 1800
)
# wait.succeeded == true   → ready to release
# wait.timed_out == true   → still in progress; decide to wait more or surface
# otherwise inspect wait.status and pergola_get_build_logs
```

### Releases → `pergola_wait_for_release`

After `pergola_push_release`, call `pergola_wait_for_release` with the
returned release name. The server polls on your behalf until that release is
the active release on the stage **and** every component has settled out of
transitional rollout states (`deployed` or `failed`), or the timeout elapses.

```text
release = pergola_push_release(project="my-project", stage="dev", config="default")
wait = pergola_wait_for_release(
  project="my-project", stage="dev",
  release=release.name,
  timeout_seconds=600,
)
# wait.succeeded == true → release is active and every component is `deployed`
# wait.all_ready == true but succeeded == false → release rolled, but a component is `failed`
# wait.timed_out == true → still in progress; decide to wait more or surface
```

A `failed` component returns the wait early without satisfying `succeeded` —
read `pergola_get_component_logs` to diagnose before retrying.

### Component lifecycle → `pergola_wait_for_component`

After `pergola_restart_component` / `pergola_start_component` /
`pergola_stop_component`, call `pergola_wait_for_component`. The server polls
`pergola_get_component_status` on your behalf until the component reaches one
of the target statuses (default `["running"]`; after a stop pass
`["stopped","suspended"]`), enters `failed`/`crashloop` (`failed: true` —
returned early so you can read logs instead of burning the budget), or the
timeout elapses (`timed_out: true`). Do **not** hand-roll a status polling
loop.

If you do end up polling manually (e.g. across wait timeouts), use the host's
wakeup/scheduling primitive when one is available, with a delay of at least 60
seconds between checks. Don't tight-loop with `sleep 5`.

Budget: ~5 min for builds, ~3 min for releases / lifecycle changes. If you
blow the budget, surface the current state and ask whether to keep waiting —
don't poll silently.

On terminal failure: read logs (`pergola_get_build_logs`,
`pergola_get_component_logs`) before retrying.

## Rule 3 — Check component status before interacting

Always call `pergola_get_component_status` (or rely on a fresh
`pergola_list_components` from the orientation step) **before** invoking any
of:

- `pergola_exec_session` / `pergola_create_exec_endpoint`
- `pergola_copy_to_component`
- `pergola_copy_from_component`
- `pergola_read_file`
- `pergola_restart_component`
- `pergola_stop_component`
- `pergola_start_component`
- `pergola_create_port_forward_endpoint`

What you're checking for: the component exists, is currently running, and is
not in a transitional state (just-starting / just-stopping). Skipping this
check leads to error-message ping-pong: you exec, get "no such pod", then
have to discover what changed.

Rough guide to what's safe by status:

- **running / deployed / healthy** — exec, restart, stop, port-forward all OK
- **starting / scheduled / pending** — wait; do not exec or restart yet
- **stopped / suspended** — `start` is the right action; exec and port-forward
  will fail
- **failed / crashloop** — `get_component_logs` first to understand why;
  `restart` only after you have a hypothesis
- **unknown / not listed** — re-run orientation; the component may have been
  removed in the latest manifest

If your status check fails or returns something unfamiliar, surface it to the
user rather than guessing — do not proceed to a mutation.

## Rule 4 — Choose the right surface for data: config-data vs. container filesystem

Pergola exposes two distinct surfaces for moving data into and out of a
component.

### Surface A — config-data (declarative, platform-managed)

- **Tools:** `pergola_list_config_data` (read keys/metadata by default; set
  `with_values=true` only when the user explicitly needs values),
  `pergola_set_config_data` (write), `pergola_remove_config_data` (delete).
- **What it carries:** named key/value entries — environment variables and
  configuration *files* mounted into known paths in the container. Secrets
  live here.
- **Direction:** read and write, but as a *platform* concept — you operate on
  the config object, not on a running container's filesystem.
- **Reading values:** `pergola_list_config_data` defaults to keys only. With
  `with_values=true`, text values are returned decoded as `value`; binary
  values are returned as `value_base64`. Treat returned values as secrets:
  do not echo them back unless the user explicitly asked to reveal them.
- **Lifecycle:** stored on the platform, versioned. Becomes visible to
  components only after a `pergola_push_release` that references the config.
  Editing config-data does **not** propagate to running components on its
  own; re-release to apply.
- **Use for:** anything the *deployment* owns — API keys, DB URLs, feature
  flags, app config files that should travel with releases and be auditable.

### Surface B — the live container filesystem (imperative, via exec / copy)

- **Tools:** `pergola_exec_session` for arbitrary commands (read with
   `cat` / `ls` / `find`; write via shell redirection);
  `pergola_copy_to_component` for typed file writes;
  `pergola_read_file` for text reads;
  `pergola_copy_from_component` for binary-safe file reads.
- **Direction:** both. Reads (fetching a debug log, inspecting state) and
  writes (injecting a script, seeding data) are equally valid uses — the
  file-fetch recipe under "Common recipes" below is a *read* through this
  surface.
- **Lifecycle of writes depends on the target path:**
  - Paths on the container's **ephemeral** filesystem (the container image
    layers, `/tmp`, the working directory) are wiped on the component's
    next release or restart.
  - Paths inside a **persistent storage mount declared in the manifest**
    survive restarts and releases — they are backed by the platform's
    persistent volumes.
  - If you are unsure whether a path is in a persistent mount, inspect the
    component's manifest (`pergola_get_build_manifest` /
    `pergola_get_build_component`) before writing data you expect to keep.
- **Use for:** debugging, runtime inspection, one-off artifacts, and (when
  writing to a persistent mount) seeding or repairing the data that the
  deployment manages at runtime.

### Choosing

- "Should this value travel with the next deploy and be auditable?" →
  **config-data**.
- "I just need to read or write a file inside the running container right
  now." → **read_file / copy_from_component / copy_to_component**.
- "Should the write survive a restart?" → only if the target path is in a
  persistent storage mount declared in the manifest. Otherwise expect it to
  be wiped on the next release or restart.

When the user's request doesn't make the surface obvious, ask. Don't pick by
default.

## Common recipes

### Fetch a file from inside a component

```text
1. pergola_orient_project(project)                    # confirm project/stage/component/release
2. pergola_get_component_status(...)                  # confirm running
3. If the path is unknown:
   pergola_exec_session(..., command=["sh","-c","find / -name <file> -type f 2>/dev/null | head -5"])
4. For text:
   pergola_read_file(..., path="/absolute/path")
5. For binary or exact bytes:
   pergola_copy_from_component(..., path="/absolute/path")
```

### Tail logs

```text
pergola_get_component_logs(project, stage, component, since="5m")
# or get_stage_logs / get_build_logs for stage- or build-scoped lookups
```

### Roll out a config change

```text
pergola_set_config_data(...)
pergola_push_release(... -c default)
pergola_wait_for_release(...)
```

### Trigger and watch a build

See Rule 2.

### Drain a stage

```text
pergola_suspend_stage(project, stage)   # stops components, disables scheduling
# work / wait / inspect
pergola_resume_stage(project, stage)
```

### Back up and restore a stage

```text
created = pergola_create_backup(project, stage, display_name="pre-change")
# created.result.backup.name → the backup id (server-assigned; discovered for you)
# poll until the snapshot is ready:
pergola_get_backup(project, stage, backup=created["result"]["backup"]["name"])
#   → storages[].ready == true before restoring

# Restore REQUIRES a suspended stage:
pergola_suspend_stage(project, stage)
pergola_restore_backup(project, stage, backup=...)   # fails with "Stage is not suspended" otherwise
pergola_resume_stage(project, stage)
```

## Mutating config sub-resources and project automation

Beyond config-data, these stage-config sub-resources and project-level
automation have MCP tools. None of them has a per-item PATCH — they are
**full-replacement or read-modify-write**, so to change one entry you send the
complete desired set (or let the remove tool read-modify-write for you).

- **Ingresses** — `pergola_get_config_ingresses` (read),
  `pergola_set_config_ingresses` (write), `pergola_remove_config_ingress`
  (drop one component).
  - `set_config_ingresses` is a **full replacement**: the `components` and
    `local_auth_providers` you pass become the entire ingress config. To add to
    existing ingresses, read first and send the merged set.
  - `remove_config_ingress` drops one component by name (read-modify-write),
    leaving other components and local auth providers intact.
  - `local_auth_providers` carry secrets (`client_secret`, basic-auth
    passwords) — treat them like config-data secrets.
  - Ingress changes need a new release to take effect.
- **Project members** — `pergola_list_project_members` (read),
  `pergola_add_project_member`, `pergola_remove_project_member`. Members are
  identified by **user id (UUID)**, not name or email; read existing ids from
  `list_project_members`. `add` grants member access by default; set `owner`
  for the owner role.
- **Auto-build** (project webhook) — `pergola_get_auto_build`,
  `pergola_add_auto_build`, `pergola_remove_auto_build`. The webhook **secret is
  sensitive**; re-creating rotates it, so `add` fails unless `renew_secret` is
  set. `get` returns null when none is configured.
- **Auto-release** (continuous delivery, per stage) —
  `pergola_list_auto_releases`, `pergola_set_auto_release`,
  `pergola_remove_auto_release`. `set` maps a branch (+ optional `config`, else
  the stage's current config) to active/disabled, updating in place or
  appending; `remove` drops the matching entry.

## When to fall back to `pergola-cli`

Use the CLI skill (and the `pergola` shell command) for:

- `pergola login` and any auth-token interaction (`pergola login` triggers a user device flow requiring to visit a url in a browser and click a button)
- CLI profile management (`pergola set cli-config`, `pergola use cli-config`)
- Access-key management (`pergola create/list/delete access-key`)
- Private repo credential setup (`pergola create ssh` / `pergola create pat`)
- `pergola local-connect` — interactive port-forward from the **user's** own
  terminal. The MCP `create_port_forward_endpoint` is for endpoint-managed
  forwards; an interactive local session is a CLI job.
- Anything the MCP server does not expose. Check `ToolSearch` for
  `mcp__pergola__*` before assuming a capability is missing.

Everything else — read, mutate, deploy, inspect — should go through MCP tools
in this session.

## Gotchas

- **Tool schemas are deferred.** Most `mcp__pergola__*` tools must be loaded
  via `ToolSearch` with `select:mcp__pergola__<name>[,...]` before they can be
  called. Calling without loading the schema returns `InputValidationError`.
- **Async means async.** Successful tool result = "accepted", not "finished".
  See Rule 2 for the wait primitive and polling fallback.
- **`set_config_ingresses` is a full replacement, not a merge.** It overwrites
  all ingress components and local auth providers on the config. To preserve
  existing entries, read with `get_config_ingresses` first and send the merged
  set; use `remove_config_ingress` to drop a single component safely.
- **`exec_session` may mutate component state.** Treat any non-read command
  as a mutation: get user confirmation before destructive commands.
- **Config-data values may contain secrets.** `pergola_list_config_data`
  returns keys only by default. If you set `with_values=true`, never echo the
  returned secret values back to the user without an explicit reason. Likewise,
  `pergola_set_config_data` inputs can contain secrets.
- **Not logged in / session expired.** If a tool fails with `not logged in or
  session expired`, the MCP server's credentials are unusable. The user must run
  `pergola login` (CLI device flow); after that, simply **retry the tool** — the
  server reloads the fresh credentials from disk automatically. A server restart
  is only needed if the endpoint or active profile changed since it started.
