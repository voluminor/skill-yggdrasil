# yggdrasil

Portable Agent Skill for building, reviewing, and debugging Go applications
that embed a Yggdrasil node through
[`github.com/voluminor/ratatoskr`](https://github.com/voluminor/ratatoskr).

The skill targets Ratatoskr's stable v1 API and covers TCP, UDP, HTTP, SOCKS5,
port forwarding, peer management, NodeInfo, sigils, topology diagnostics, and
load-aware design. Ratatoskr runs its network stack in userspace: no TUN
device, root privileges, external Yggdrasil daemon, or `yggstack` process is
required.

## Install the skill

From GitHub:

```text
npx skills add https://github.com/voluminor/skill-yggdrasil --skill yggdrasil
```

Short source form:

```text
npx skills add voluminor/skill-yggdrasil --skill yggdrasil
```

The Ratatoskr mirror is available at:

```text
https://www.ratatoskr.space/pkg/skill-yggdrasil
```

Mirrors can lag behind GitHub. Compare the version shown by the mirror with
`metadata.version` in `skills/yggdrasil/SKILL.md` before relying on it.

## Install the Ratatoskr module

Ratatoskr releases are available under three distinct Go module identities.
Use exactly one identity, including its subpackage imports, throughout a Go
module.

Canonical GitHub module:

```text
go get github.com/voluminor/ratatoskr@latest
```

HTTPS mirror:

```text
GOPROXY=https://ratatoskr.space GOSUMDB=off \
  go get ratatoskr.space/pkg/ratatoskr@latest
```

Yggdrasil-native mirror, when the Go command already has overlay access:

```text
YGG_HOST="14cc7d57b5e70f679b851fe5b272ce17c70632ff4beb5b35ab64bc706b2485af.pk.ygg"
GOPROXY="http://${YGG_HOST}" GOSUMDB=off GOINSECURE="${YGG_HOST}/*" \
  go get "${YGG_HOST}/pkg/ratatoskr@latest"
```

The mirror identities require different import paths. The `.pk.ygg` endpoint
uses HTTP inside the encrypted overlay, so keep `GOINSECURE` scoped to that
exact host. Full v1.1.0 commands and imports are recorded in
[`api-v1.1.md`](skills/yggdrasil/references/api-v1.1.md#module-installation-identities).

## Portability model

The skill uses standard Agent Skills metadata and does not name tools from a
specific agent client. It separates two concerns:

- the skill can be read by Agent Skills-compatible clients;
- generated application guidance targets the OS and runtime named by the user.

An application may intentionally be Windows-only, mobile-only, or Unix-only.
The skill does not impose unrelated platform checks.

Ratatoskr platform claims are labeled by evidence: runtime-tested,
compile-tested, experimental, or unknown. See
[`references/platforms.md`](skills/yggdrasil/references/platforms.md).

## Layout

```text
skills/yggdrasil/SKILL.md
skills/yggdrasil/references/migration-v1.md
skills/yggdrasil/references/platforms.md
skills/yggdrasil/references/networking.md
skills/yggdrasil/references/services.md
skills/yggdrasil/references/discovery.md
skills/yggdrasil/references/load-safety.md
skills/yggdrasil/references/api-v1.1.md
skills/yggdrasil/evals/evals.json
```

`SKILL.md` is the decision and routing layer. Exact v1.1.0 API details live in
`api-v1.1.md`; operational choices live in the focused references.

## Optional maintainer checks

Maintainers may run checks appropriate to their environment, for example:

```text
npx skills-ref validate ./skills/yggdrasil
npx skills add . --list
```

The skill does not run these commands, install their dependencies, or add a
platform build matrix on behalf of users.
