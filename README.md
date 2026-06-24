# yggdrasil

Agent skill for building, reviewing, and debugging Go programs that embed a
Yggdrasil end-to-end-encrypted IPv6 mesh node through
`github.com/voluminor/ratatoskr`.

The skill focuses on userspace Yggdrasil applications: no TUN device, no root
privileges, and no external `yggdrasil` daemon. It covers clients, HTTP/TCP/UDP
servers, SOCKS5 proxies, port forwarding, peer management, NodeInfo sigils, and
mesh-friendly transfer patterns.

## Install

Install the published skill:

```bash
npx skills add https://github.com/voluminor/skill-yggdrasil --skill yggdrasil
```

Or install from the short GitHub source:

```bash
npx skills add voluminor/skill-yggdrasil --skill yggdrasil
```

The skills.sh page is:

```text
https://www.skills.sh/voluminor/skill-yggdrasil/yggdrasil
```

## Repository Layout

```text
skills/yggdrasil/SKILL.md        Main skill entrypoint
skills/yggdrasil/references/     Detailed Ratatoskr and Yggdrasil references
skills/yggdrasil/evals/          Forward-test scenarios for the skill
```

The skill name is `yggdrasil`, and the skill directory is also
`skills/yggdrasil` so discovery tools and indexes can identify it by the same
slug.

## Validate

From the repository root:

```bash
npx skills-ref validate ./skills/yggdrasil
npx skills add . --list
```

Both commands should report a single valid skill named `yggdrasil`.
