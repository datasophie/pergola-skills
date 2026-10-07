---
name: pergola-manifest
description: Authors and validates Pergola project manifests (`pergola.yaml`). Use when creating, editing, or debugging the application stack configuration, defining components, linking services, or setting up scheduled jobs on Pergola.
---

# Pergola Manifest Skill

Guidance for authoring `pergola.yaml` (or `pergola.json`), the file in the
repository root that declares an application's stack on Pergola.

## Quick Start

1.  **Version:** Always set `version: v1`, the only version.
2.  **Validate** after every edit, using the loop below.

## Validate until clean

Validation runs on the Pergola API, so it needs the MCP tools or a logged-in
CLI:

- **MCP:** `pergola_validate_manifest` with the manifest text. Done when every
  message has type `ok`.
- **CLI:** `pergola validate manifest <path>`. Done when it prints "The
  specified manifest '<file>' is fine." Validation errors print as errors but
  the exit code stays 0, so read the output.

On any `error`, fix exactly what the message names and validate again.
Warnings are allowed but worth reading. If a message is unclear, compare with
the schema at `https://docs.pergola.cloud/pergola_project_manifest_spec.yaml`.

## Core Principles

- **Manifest-based Architecture:** Every aspect of the application stack (web frontends, APIs, databases, cron jobs) must be declared in the manifest.
- **`components` is a list:** It is an array of component objects, each with a required `name` field — not a map keyed by name.
- **Internal reachability:** A component is reachable by *other components* on the `ports` it declares. **Public** web exposure is separate: declare an entry under `ingresses` (which gives a host and managed TLS).
- **Component Linking:** `component-ref` in the `env` section injects another component's runtime network name, i.e. its hostname. It carries no scheme or port. Use `config-ref` to pull a value from the stage's config-data (env vars / secrets). Never hardcode internal IPs or hostnames.
- **Resources:** Define `cpu` and `memory` limits only to prevent OOM kills and ensure stable scaling when required. No resource definition lets the application use all the resources it needs.

## Platform Constraints (feasibility gate)

Check these **before** authoring a manifest. If the app fundamentally needs any of them, say so plainly and stop — do not fake a deployable manifest:

- **Wildcard / dynamic-subdomain ingress** — Pergola ingress is **fixed-host only**. Apps that mint per-tenant subdomains at runtime cannot be exposed correctly.
- **Docker host access** — no `host.docker.internal`, no docker-in-docker, no host bind mounts.
- **Linux capabilities / privilege** — the v1 manifest has no `cap_add`, `privileged`, devices, or sysctls.
- **Multicast / raw networking / host networking.**
- Also not supported in v1 (do not emit; find another way): `healthcheck`, `depends_on`.

When in doubt, prototype the smallest viable subset and document what was dropped.

## Common Workflows

### Creating a New Manifest
When starting a new project, identify the components (e.g., a `web` frontend and a `db` backend) and draft the `components` list. See [manifest-spec.md](references/manifest-spec.md#best-practices--patterns) for templates.

### Linking Services
To link `web` to `api`, declare the port on `api` and pass its hostname and
port to `web`, which builds the URL itself:
```yaml
components:
  - name: web
    env:
      - name: API_HOST
        component-ref: api
      - name: API_PORT
        value: "8080"
  - name: api
    ports:
      - 8080
```

### Binding env vars and secrets
Values that live on the stage's config-data (set with the MCP tool
`pergola_set_config_data` or the CLI `pergola add config-data`) are pulled in by
key. A `value` next to `config-ref` acts as the default:
```yaml
env:
  - name: DATABASE_URL
    config-ref: DATABASE_URL   # key of a config-data entry on the stage
  - name: LOG_LEVEL
    config-ref: LOG_LEVEL
    value: info                # used when the key is not set
```

## Gotchas

- **An ingress host belongs to a component name** Renaming the component that
  owns an ingress host makes the next release fail with "is already in use",
  even on a suspended stage. Keep the owning component's name, or change the
  host in the same edit.
- **The manifest lives in the build** Local edits change nothing on Pergola
  until they are pushed to the git remote and a new build is released.
- **A missing config-data key doesn't fail the release** The variable falls
  back to `value`, or stays empty without one. Bind config-data before the
  first release.

## More

- **Every field and complete examples:**
  [references/manifest-spec.md](references/manifest-spec.md).
- **When the validator or the official schema contradicts this skill**, trust
  them and tell the user which statement is outdated, so the skill can be
  corrected.
