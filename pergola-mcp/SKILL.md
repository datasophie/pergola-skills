---
name: pergola-mcp
description: >-
  Operates the Pergola deployment platform through the `mcp__pergola__*` tools:
  inspect projects, stages, builds, releases and logs; deploy builds and
  configs; exec and file access in running components; component and stage
  lifecycle; config-data, ingresses, backups, members, auto-build and
  auto-release; pergolize a repo that has no `pergola.yaml`. Use for any
  Pergola task while these tools are available, unless the user asks for the
  pergola CLI or for CLI login and profile setup.
---

# Pergola via MCP

Tool names below drop the `mcp__pergola__` prefix. Each tool's description
already states its inputs, preconditions and result fields, so read it instead
of guessing. This skill adds only what spans several tools.

Hierarchy: a **project** is one git repo with a `pergola.yaml` manifest.
**Builds** (`<branch>_b<n>`) are images built from a commit. A **release** puts
a build and a **config** (config-data: env vars, files, secrets) onto a
**stage** (`dev`, `qa` or `prod`), where the manifest's **components** run.

## Workflow

1. **No project or no `pergola.yaml`:** call `pergola_init` and follow the
   playbook it returns. Don't guess a manifest.
2. **Orient** before the first mutation in a session and whenever the user
   switches project: `pergola_whoami`, then `pergola_orient_project`. Stale
   names (wrong project, archived stage, renamed component) are the most
   common failure.
3. **Mutate**, applying the confirmation rule below. If a component reports a
   status you don't recognise, tell the user instead of acting on it.
4. **Wait** for the outcome, because a successful call only means "accepted".
   Use `pergola_wait_for_build`, `pergola_wait_for_release` or
   `pergola_wait_for_component` with an explicit `timeout_seconds`. The default
   110 keeps under common MCP client call timeouts, but builds often need
   300–600. Don't hand-roll polling loops.
5. **Verify** against the table below before reporting success.

## Done means verified

| Step | Success | On failure |
|---|---|---|
| Build | `pergola_wait_for_build` returns `succeeded: true` | Read `pergola_get_build_logs`, fix, push a new build |
| Release | `pergola_wait_for_release` returns `succeeded: true` | Read `pergola_get_component_logs` of the `failed` component before retrying |
| Lifecycle | `pergola_wait_for_component` reaches the target status | On `failed: true`, read the component logs first |
| Any wait | | On `timed_out: true`, report the current state and ask whether to keep waiting |

Never report a deploy as done from the push result alone. If a check was
skipped or timed out, say so.

## Confirm before deleting, restoring, suspending, or changing access

Some calls destroy data, take a stage offline, or change who can access a
project. Before making one, tell the user what will happen and get an explicit
yes, naming the target (project, stage, component, config, member):

- **Delete or remove:** `pergola_delete_project`, `pergola_delete_stage`,
  `pergola_delete_config`, and every `pergola_remove_*` tool (config-data,
  ingresses, identities, auto-build, auto-release, project members).
- **Restore:** `pergola_restore_backup` may replace the stage's current state
  or data with the snapshot.
- **Suspend:** `pergola_suspend_stage` stops all components on the stage.
- **Member or role changes:** `pergola_add_project_member` and
  `pergola_remove_project_member`, above all when the owner role is involved.
- **Destructive exec:** a `pergola_exec_session` command that deletes files,
  drops data or kills processes.

A user request that already names the operation and its target (for example
"delete the qa stage of shop") counts as that yes. Other mutations, such as
pushing builds and releases, follow the user's request as usual.

## Where data goes

- **Config-data** (`pergola_set_config_data`) holds values that should travel
  with deploys and stay auditable: API keys, DB URLs, feature flags, config
  files. Running components see a change only after a new release.
- **The running container** (`pergola_read_file`,
  `pergola_copy_from_component`, `pergola_copy_to_component`,
  `pergola_exec_session`) is for inspection and one-off changes. Writes survive
  a restart or release only inside a `storage` mount declared in the manifest
  (check with `pergola_get_build_component`).
- If the request doesn't make the choice obvious, ask.

## Secrets

- `pergola_list_config_data` returns keys only. Set `with_values=true` only
  when the user needs the values, and don't echo them back without a reason.
- Ingress auth providers, the auto-build webhook secret and access keys are
  secrets too. An access key belongs in the MCP client configuration, never in
  the chat.

## Gotchas

- **Deferred schemas** Load a tool with ToolSearch
  `select:mcp__pergola__<name>` before calling it, and search there before
  concluding a capability is missing.
- **The manifest lives in the build** Local `pergola.yaml` edits change
  nothing until they are pushed to the git remote, built with
  `pergola_push_build`, and released.
- **An ingress host belongs to a component name** A release that moves a
  host to a differently named component fails with "is already in use", even
  on a suspended stage. Keep the owning component's name or pick another host.
- **Deleted stage names stay reserved** `pergola_delete_stage` archives the
  stage, so `pergola_create_stage` with the same name fails until the archive
  is purged.
- **`pergola_exec_session` has no timeout** Don't run streaming or
  long-running commands such as `tail -f`, and wrap slow ones in
  `timeout <seconds>`.
- **Vulnerability reports can be huge** Start with
  `pergola_get_build_vulnerability_summary`, then fetch single findings with
  `pergola_get_build_component_vulnerability`.
- **Auth errors carry their own fix** Relay the error text to the user. They
  run `pergola login` in their own terminal, then you retry the tool.

## More

- **Recipes** for build and deploy, config rollout, file fetch, backup and
  restore, stage drain and ingress edits:
  [references/recipes.md](references/recipes.md).
- **CLI-only tasks** (`pergola login`, CLI profiles, access keys, private-repo
  credentials, interactive `pergola local-connect`): use the `pergola-cli`
  skill.
- **When a tool contradicts this skill**, trust the tool and tell the user
  which statement is outdated, so the skill can be corrected.
