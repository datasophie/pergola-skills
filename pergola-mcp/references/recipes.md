# Pergola MCP recipes

Step sequences that span several tools. Each tool's description has its full
parameters and result fields.

## Contents
- Build and deploy
- Roll out a config change
- Fetch a file from a component
- Back up and restore a stage
- Drain a stage
- Change ingresses

## Build and deploy

```text
push = pergola_push_build(project, branch="main")   # omit branch to build every changed branch
# push.result.outcome:
#   scheduled          → build names in push.result.build_names
#   rejected           → reason in push.result.notifications (e.g. B0005 "no manifest on branch")
#   skipped_no_changes → nothing new to build; pass force=true only if a rebuild is really wanted
#   pending_unknown    → still scheduling or skipped: check pergola_list_builds and
#                        pergola_get_project_notifications, see push.result.message
pergola_wait_for_build(project, build=<build name>, timeout_seconds=600)
# succeeded: true → release it; otherwise read pergola_get_build_logs
release = pergola_push_release(project, stage, build=<build name>, config="default")
pergola_wait_for_release(project, stage, release=<Name from push_release>, timeout_seconds=600)
```

## Roll out a config change

```text
pergola_set_config_data(project, stage, config="default", data=[{key, value}])
release = pergola_push_release(project, stage, config="default")
pergola_wait_for_release(project, stage, release=<Name from push_release>, timeout_seconds=600)
```

A config-only release may keep unchanged components running. Apps that read
config files only at start may need `pergola_restart_component` afterwards.

## Fetch a file from a component

```text
pergola_get_component_status(project, stage, component)        # must be running
# only if the path is unknown:
pergola_exec_session(..., command=["sh","-c","find / -name <file> -type f 2>/dev/null | head -5"])
pergola_read_file(..., path="/absolute/path")                  # text
pergola_copy_from_component(..., path="/absolute/path")        # binary or exact bytes
```

## Back up and restore a stage

A restore changes live infrastructure and may replace the stage's current
state or data with the snapshot. Confirm with the user before restoring,
naming the stage and the backup.

```text
created = pergola_create_backup(project, stage, display_name="pre-change")
# backup id: created.result.backup.name
pergola_get_backup(project, stage, backup=<id>)   # until every storages[].ready is true
pergola_suspend_stage(project, stage)             # restore fails with "Stage is not suspended" otherwise
pergola_restore_backup(project, stage, backup=<id>)
pergola_resume_stage(project, stage)
```

When polling `pergola_get_backup`, leave at least 60 seconds between checks
and use the host's wakeup or scheduling primitive if it has one.

## Drain a stage

```text
pergola_suspend_stage(project, stage)   # stops components and scheduled jobs; confirm first
# work / wait / inspect
pergola_resume_stage(project, stage)
```

A release pushed to a suspended stage becomes active only after the resume.
Scheduled `@release` components may then need
`pergola_start_component(..., now=true)`.

## Change ingresses

`pergola_set_config_ingresses` replaces the whole ingress configuration of a
config, so read it with `pergola_get_config_ingresses`, merge your change, and
send the complete set. To drop one component's ingress, use
`pergola_remove_config_ingress`, which keeps the rest. Push a release
afterwards for the change to take effect.
