---
name: pergola-manifest
description: Author and validate Pergola project manifests (`pergola.yaml`). Use this skill when creating, editing, or debugging the application stack configuration, defining components, linking services, or setting up scheduled jobs on Pergola.
---

# Pergola Manifest Skill

This skill provides expert guidance for authoring `pergola.yaml` (or `pergola.json`) files, the core configuration for deploying applications on the Pergola platform.

## Quick Start

1.  **Version:** Always use `version: v1`.
2.  **Validation:** Run `pergola validate manifest <path>` frequently.
3.  **Reference:** For detailed schema properties and common patterns, see [manifest-spec.md](references/manifest-spec.md).

## Core Principles

- **Manifest-based Architecture:** Every aspect of the application stack (web frontends, APIs, databases, cron jobs) must be declared in the manifest.
- **`components` is a list:** It is an array of component objects, each with a required `name` field — not a map keyed by name.
- **Internal reachability:** A component is reachable by *other components*. **Public** web exposure is separate: declare an entry under `ingresses` (which gives a host and managed TLS).
- **Component Linking:** Use `component-ref` in the `env` section to inject another component's runtime network name. Use `config-ref` to pull a value from the stage's config-data (env vars / secrets). Never hardcode internal IPs or hostnames.
- **Resources:** Define `cpu` and `memory` limits only to prevent OOM kills and ensure stable scaling when required. No resource definition lets the application use all the resources it needs.

## Common Workflows

### Creating a New Manifest
When starting a new project, identify the components (e.g., a `web` frontend and a `db` backend) and draft the `components` list. See [manifest-spec.md](references/manifest-spec.md#best-practices--patterns) for templates.

### Linking Services
To link `web` to `api`:
In the `web` component, add:
   ```yaml
   env:
     - name: API_ENDPOINT
       component-ref: api
   ```

### Binding env vars and secrets
Values that live on the stage (set via `pergola add config-data`, see the `pergola-cli` skill) are pulled in by key:
```yaml
env:
  - name: DATABASE_URL
    config-ref: DATABASE_URL   # key of a config-data entry on the stage
```

### Debugging Validation Errors
If `pergola validate manifest` fails:
1. Check indentation (standard YAML).
2. Ensure all `component-ref` values match the `name` of an existing component in the `components` list.
3. Verify that any referenced ports are integers.
4. Compare against the schema at `https://docs.pergola.cloud/pergola_project_manifest_spec.yaml`.

## Reference Documentation
For a complete guide to all supported fields and advanced configurations, read:
- [manifest-spec.md](references/manifest-spec.md)
