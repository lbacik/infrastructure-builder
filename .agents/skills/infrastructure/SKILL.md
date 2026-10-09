---
name: infrastructure
description: Deploy or build an infrastructure service on a Docker context.
argument-hint: "[environment] [service]"
arguments: [ENVIRONMENT, SERVICE]
disable-model-invocation: true
---

Role: DevOps Engineer
Goal: deploy (or build) the service $SERVICE to the Docker context
$ENVIRONMENT, using compose templates from the template source and
configuration from this repository's `environments/` tree.

# Invocation

The full invocation is: $ARGUMENTS

The first two positional arguments are $ENVIRONMENT and $SERVICE. Treat any
remaining text as additional constraints for this deployment - for example,
limiting which containers of the compose project to start - and honour it
alongside the process below. When it conflicts with the process, ask the user.

All paths below are relative to the root of this repository.

# Template source

The template source is the repository holding the compose projects: one
top-level directory per service, each with `compose.yaml`, an `env.example`
listing the variables the compose files consume, and optional extra compose
files layered with `-f` (`compose.override.yaml`, `compose.local.yaml`,
`compose.public.yaml`). Its `README.md` documents per-service exceptions:
which extra compose file serves which purpose, and which services take no env
file.

`docs/resources.md` names the source in a `## Templates` section:

```markdown
## Templates

source: templates/infrastructure
```

The source is one of two modes:

- **remote** - a git URL, optionally with a ref: `<git-url>#<ref>`. This is
  the default when `docs/resources.md` or its `## Templates` section is
  absent: `https://github.com/lbacik/infrastructure.git#main`.
- **local** - a path inside this repository, always `templates/infrastructure`
  (a git submodule, see "Editing a template").

## Referencing compose files

In remote mode every compose file is a git URL that Compose fetches itself;
relative build contexts in it resolve inside that fetched copy, so images
build straight from the source. Omit `--project-directory` - it would
re-anchor those relative build contexts onto a local directory:

```bash
docker --context $ENVIRONMENT compose \
  -f '<git-url>#<ref>:$SERVICE/compose.yaml' \
  [-f '<git-url>#<ref>:$SERVICE/<extra-compose-file>'] \
  --env-file environments/$ENVIRONMENT/$SERVICE.env \
  -p $SERVICE \
  [build|up -d]
```

In local mode:

```bash
docker --context $ENVIRONMENT compose \
  --project-directory templates/infrastructure/$SERVICE \
  -f templates/infrastructure/$SERVICE/compose.yaml \
  [-f templates/infrastructure/$SERVICE/<extra-compose-file>] \
  --env-file environments/$ENVIRONMENT/$SERVICE.env \
  -p $SERVICE \
  [build|up -d]
```

Square brackets mark optional parts: include an extra compose file only when
the service has one and the deployment calls for it, pick one of `build` or
`up -d`, and drop the brackets themselves. Always pass `-p $SERVICE`
explicitly; a stale `COMPOSE_PROJECT_NAME` in an env file silently renames
containers.

## Inspecting templates in remote mode

To read `README.md`, a service's `env.example`, or the list of its extra
compose files, make a throwaway shallow clone in a temporary directory
outside this repository (`git clone --depth 1 --branch <ref> <git-url>
<tmp-dir>`). The clone is for reading only; deployment commands keep
referencing the git URL.

## Editing a template

When the user wants to change a template, follow
[`editing-templates.md`](editing-templates.md) before deploying: it switches
the template source to local mode.

# Environment

$ENVIRONMENT is the name of a Docker context (`docker context ls`, "NAME"
column). If it is missing, ask the user. Validate access with `docker
--context $ENVIRONMENT info` before continuing. Pass `--context
$ENVIRONMENT` on each command, leaving the user's current context unchanged.

# Service

$SERVICE is a top-level directory of the template source. If it is missing,
ask the user. A missing service surfaces as a `no such file or directory`
error on its `compose.yaml` during validation (step 4 of the process).

# Configuration

Configuration lives in this repository under `environments/`, one directory
per Docker context, one env file per service:

```
environments/
├── ale
│   ├── listmonk.env
│   └── portainer.env
└── local
    ├── redis.env
    └── postgresql.env
```

When `environments/$ENVIRONMENT/$SERVICE.env` is missing and the service
takes an env file, create it from the service's `env.example`, then ask the
user for every value you cannot derive. The file is complete when each
variable of `env.example` has a value.

# Process

1. Resolve the template source and its mode from `docs/resources.md`.
2. Confirm access to $ENVIRONMENT.
3. Confirm the configuration file is complete, creating it if needed.
4. Validate the composed project: run the deployment command with `config
   --quiet` in place of `build`/`up -d`. Done when it exits 0.
5. Run the deployment or build.
6. Verify with `docker --context $ENVIRONMENT compose -p $SERVICE ps` that
   every started container is running (and healthy, where a healthcheck is
   defined); for a build, that each built image is listed by `docker
   --context $ENVIRONMENT image ls`.
