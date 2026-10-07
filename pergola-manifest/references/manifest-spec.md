# Pergola Project Manifest (pergola.yaml)

The Project Manifest is the central configuration for a Pergola project, describing the application stack, networking, and scaling. It is conceptually similar to Docker Compose but optimized for high-availability cloud deployments.

## Contents
- Core Schema (v1): top-level properties
- Component Configuration: key component fields
- Linking Components
- Best Practices & Patterns: web + database, scheduled jobs, resource constraints
- Validation

## Core Schema (v1)

The only supported version is `v1`. The manifest file name can be `pergola.yaml`, `pergola.yml`, or `pergola.json`.

The full OpenAPI schema is available at: `https://docs.pergola.cloud/pergola_project_manifest_spec.yaml`

### Top-level Properties

| Property | Type | Description |
| :--- | :--- | :--- |
| `version` | string | `v1`, the only version and the default. Set it explicitly. |
| `components` | array | **Required.** List of component objects; each has a required `name`. |

## Component Configuration

A component represents a service, worker, or job. `components` is a **list**; each entry has a required `name`.

```yaml
components:
  - name: webapp
    docker:
      image: my-image:latest   # use an existing image
      # --- or build from the repo instead of `image`: ---
      # file: Dockerfile        # Dockerfile path
      # build-context: .        # build context (default ".")
    ports:                      # ports exposed
      - 8080
    env:
      - name: DB_HOST
        component-ref: database
    resources:
      cpu: 100m
      memory: 256Mi
```

### Key Component Fields

| Field | Type | Notes |
| :--- | :--- | :--- |
| `name` | string | **Required.** Identifier used by `component-ref`. |
| `docker` | object | Container source. `image` (existing image) **or** `file` (Dockerfile path) + `build-context` (default `.`) + `build-args` (list of `{name, value}`). |
| `ports` | array\<int\> | Ports exposed for linking and for `ingresses` to target. |
| `ingresses` | array | Public web exposure. Entry fields: `host` (**required**), `port` (required if the component has multiple ports), `path` (default `/`), `sticky` (bool, default `false`). |
| `env` | array | Each entry: `name` (**required**) plus `value`, `config-ref` (a stage config-data key), or `component-ref` (another component's name). `component-ref` resolves to that component's hostname, without scheme or port. A `value` next to `config-ref` is the fallback when the key is not set on the stage. |
| `files` | array | Each entry: `path` (**required**) plus `content` (inline) or `config-ref` (a stage config-data key). |
| `resources` | object | `cpu` (e.g. `"500m"`) and `memory` (e.g. `"2Gi"`). Optional — omit to use what the app needs; set to cap and prevent OOM kills. |
| `scaling` | object | `min` and `max` instance counts (integers). |
| `storage` | array | Persistent volumes. Each entry: `name`, `path`, `size` (Kubernetes format, e.g. `"2Gi"`) — all **required** — plus `type` (`standard` \| `premium` \| `temporary` \| `memory`, default `standard`). |
| `scheduled` | string | Cron expression or `@release` (run once at deployment) for jobs. |
| `scheduled-timezone` | string | Timezone for `scheduled` (default `Etc/UTC`). |
| `max-retries` | int | Retry count for scheduled jobs. |
| `command` / `args` | array\<string\> | Override the container entrypoint / its arguments. |

Fields not listed here, such as `identity`, `user` and `docker.platform`, are described in the full schema.

## Linking Components

Linking is strictly manifest-based. To connect `component-A` to `component-B`:

1.  **Component-A** references Component-B in its `env` section:

```yaml
# pergola.yaml
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

## Best Practices & Patterns

### 1. Web + Database Pattern
```yaml
version: v1
components:
  - name: web
    docker:
      file: Dockerfile
      build-context: ./frontend
    ports:
      - 3000
    ingresses:
      - host: app          # public hostname prefix; Pergola manages TLS
        port: 3000
    env:
      - name: BACKEND_HOST
        component-ref: api
  - name: api
    docker:
      file: Dockerfile
      build-context: ./backend
    ports:
      - 8080
    env:
      - name: DB_HOST
        component-ref: db
  - name: db
    docker:
      image: postgres:15
    ports:
      - 5432
    storage:
      - name: pgdata
        path: /var/lib/postgresql/data
        size: "10Gi"
```

### 2. Scheduled Jobs (Cron)
```yaml
components:
  - name: cleanup
    docker:
      file: Dockerfile
      build-context: ./scripts
    scheduled: "0 0 * * *"        # every night at midnight
    scheduled-timezone: Etc/UTC   # optional, this is the default
```

### 3. Resource Constraints
`resources` are optional: omit them to let the component use what it needs. Set them to cap usage and prevent OOM kills, or to make scaling predictable.
```yaml
resources:
  cpu: "200m"
  memory: "512Mi"
```

## Validation

Always validate the manifest before committing, with `pergola_validate_manifest` (MCP) or
`pergola validate manifest <path>` (CLI). See "Validate until clean" in SKILL.md for what success looks like.
