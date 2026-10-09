# Infrastructure

> **Work in progress** - the layout and the `/infrastructure` skill are still
> changing.

Per-project infrastructure: the configuration of which services run on which
Docker hosts, deployed by an agent skill from shared Docker Compose templates.

## How it works

Three pieces come together on every deployment:

| Piece | Where | What |
| --- | --- | --- |
| Templates | [lbacik/infrastructure](https://github.com/lbacik/infrastructure) (default) or `templates/infrastructure` | One directory per service: `compose.yaml`, `env.example`, optional extra compose files |
| Configuration | `environments/<docker-context>/<service>.env` | Populated env file per host and service |
| Target | a Docker context | Local or remote Docker engine (`docker context ls`) |

The `/infrastructure` skill joins them into one `docker compose` call, e.g.
for the default (remote) templates:

```bash
docker --context local compose \
  -f 'https://github.com/lbacik/infrastructure.git#main:redis/compose.yaml' \
  --env-file environments/local/redis.env \
  -p redis \
  up -d
```

Compose fetches the compose files from git itself, and images build from
that remote copy - templates are not copied into this repository.

## Layout

```
.
├── .agents/skills/infrastructure/   # the skill (.claude -> .agents)
├── docs/resources.md                # optional: template source override
├── environments/                    # configuration, one dir per Docker context
│   └── local/
│       └── redis.env
└── templates/infrastructure/        # only when templates are edited locally
```

## Configuration

### Docker contexts

Every target host is a Docker context; its name is the `environments/`
directory name.

```bash
docker context create rpi5 --docker host=ssh://user@rpi5
```

### Environment files

`environments/<docker-context>/<service>.env` holds the values for that
service's `env.example`. The skill creates a missing file from `env.example`
and asks for the values it cannot fill in. `environments/` is gitignored -
the files carry secrets.

### Template source

Without `docs/resources.md` the skill uses
`https://github.com/lbacik/infrastructure.git#main`. To use another
repository or ref, add:

```markdown
## Templates

source: https://github.com/<user>/<repo>.git#<ref>
```

To edit templates, ask the skill to change one: it adds the template
repository as a git submodule in `templates/infrastructure`, switches
`source:` to that path, and commits template changes in the submodule, apart
from this repository's history. Remove `templates/` from `.gitignore` first,
or let the skill do it.

## Using the skill

The skill is user-invoked only: Claude Code runs it as `/infrastructure`,
Codex as `$infrastructure`. Neither agent starts it on its own.

```
/infrastructure <docker-context> <service> [extra instructions]
```

Missing arguments are asked for. Examples:

```
# deploy redis to the local Docker engine
/infrastructure local redis

# deploy postgresql to a remote host, with host port bindings
/infrastructure ale postgresql include compose.override.yaml

# build the image only, without starting containers
/infrastructure rpi5 litellm build only

# start just one container of a multi-container project
/infrastructure local elastic start only elasticsearch

# change a template, then deploy the changed version
/infrastructure local redis set maxmemory to 256mb in the template

# no arguments - the skill asks for the context and the service
/infrastructure
```

Each run validates the composed project (`docker compose ... config`)
before deploying, and finishes by checking the containers with
`docker compose -p <service> ps`.
