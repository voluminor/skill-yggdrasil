---
name: skill-yggdrasil
description: "Use when building, reviewing, or debugging Go programs that join the Yggdrasil end-to-end-encrypted mesh network — clients, HTTP/TCP/UDP servers, SOCKS5 proxies, port forwarders, peer management, and discoverable nodes — with the node running in userspace and no TUN device, root, or external daemon. Covers dialing and listening over Yggdrasil IPv6, integrating frameworks (net/http, fasthttp, gRPC, gnet), publishing typed NodeInfo via sigils, and designing for a low-bandwidth, fault-prone mesh."
user-invocable: true
license: LGPL-2.1
metadata:
  author: SUNsung
  version: "0.1.0"
  openclaw:
    emoji: "🌳"
    homepage: https://github.com/voluminor/skill-yggdrasil
    requires:
      bins:
        - go
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent AskUserQuestion
---

# Yggdrasil Networking In Go

## Overview

Use this skill to build Go programs that connect to and serve on the
**Yggdrasil** end-to-end-encrypted IPv6 mesh: clients, HTTP/TCP/UDP servers,
SOCKS5 proxies, port forwarders, and nodes that publish discovery metadata. The
node runs **in userspace** on a gVisor netstack — no TUN device, no root, no
external `yggdrasil` daemon, and no `yggstack` binary. You dial and listen over
Yggdrasil with standard Go interfaces (`net.Conn`, `net.Listener`,
`net.PacketConn`, `http.Transport`).

The embedding library is `github.com/voluminor/ratatoskr`: create the node with
`ratatoskr.New` and drive everything through its Go API. Scope and stability of
its packages are summarized below.

## Design For A Constrained Mesh

Yggdrasil is an end-to-end-encrypted overlay **mesh**, not a datacenter fabric.
Links are high-latency, low-bandwidth, lossy, and intermittent; a peer reachable
now may be gone a second later, and traffic may cross many hops. Build **every**
Yggdrasil solution as if for a weak, unstable connection. This is not optional
polish — it is the default shape of correct code here.

- **Never unbounded.** A deadline on every dial and request
  (`context.WithTimeout`); bounded retries; bounded queues. No infinite waits.
- **Cap concurrency.** Limit in-flight work: SOCKS `MaxConnections`, forward
  `SetMaxUDPSessions`, `http.Transport.MaxConnsPerHost`/`MaxIdleConnsPerHost`,
  your own worker pools. Both default to *unlimited* — set real limits.
- **Chunk large transfers, verify, resume.** Never move a big file in one shot.
  Split into small chunks (e.g. HTTP `Range`), verify each (size/checksum), and
  make it resumable so a dropped link costs one chunk, not the whole transfer.
- **Retry with backoff + jitter, idempotently.** Expect mid-operation
  disconnects. Make operations resumable/idempotent; exponential backoff with
  jitter; cap attempts.
- **Keep payloads small.** Avoid chatty protocols and polling; compress; use
  compact encodings. Keep NodeInfo/sigils well under the 16 KB limit.
- **Size buffers to the link.** Use `node.MTU()` for packet buffers (userspace
  MTU is dynamic, floored at 1280).
- **Degrade, don't hammer.** On failure back off and return partial results;
  never aggressively retry a peer or flood the mesh.

A naive "serve a big file over `net/http`" or "one giant request/response" is the
wrong shape. The right shape is bounded, chunked, resumable, fault-tolerant by
default. See the constrained-mesh transfer pattern in
[references/integrations.md](references/integrations.md#constrained-mesh-transfer-pattern).

## Source Of Truth

The active module is authoritative — not memory, not stale README text, not
this skill. Before writing version-specific guidance, inspect what the project
actually depends on:

```bash
go list -m github.com/voluminor/ratatoskr   # selected version
go doc github.com/voluminor/ratatoskr       # root API as compiled
go doc github.com/voluminor/ratatoskr/mod/peermgr ConfigObj
```

To read the real source of the selected version, resolve it from the module
cache instead of guessing a path:

```bash
go list -m -f '{{.Dir}}' github.com/voluminor/ratatoskr
```

Do not freeze the minimum Go version from README text — it can lag behind the
`go.mod` directive. Read the active `go.mod` of the selected Ratatoskr version.

## Default Workflow

1. Identify the application shape: embedded node, HTTP client/server, raw
   TCP/UDP, SOCKS5, port forwarding, peer management, or NodeInfo/sigils.
2. Use the root package `github.com/voluminor/ratatoskr` for applications.
   Reach into `mod/core` only through the `core.Interface` contract; avoid
   `*core.Obj`/`UnsafeCore()` unless a low-level package genuinely requires it.
3. Own a `context.Context`, pass it as `ratatoskr.ConfigObj.Ctx`, **and** still
   `defer node.Close()` after a successful start. Both are needed.
4. Configure peers in exactly one place:
   - Static Yggdrasil peers: `cfg.Config.Peers`
   - Ratatoskr peer manager: `ratatoskr.ConfigObj.Peers` (then leave
     `cfg.Config.Peers` empty, or `New` returns `ErrPeersConflict`).
5. Use `node.DialContext` for outbound TCP/UDP and `node.Listen` /
   `node.ListenPacket` for services inside Yggdrasil.
6. Bracket IPv6 addresses in URLs and host-port strings:
   `http://[200:abcd::1]:8080`, never `http://200:abcd::1:8080`.
7. Treat the link as weak: put a deadline on every call, cap concurrency, and
   chunk/verify/resume large transfers (see Design For A Constrained Mesh).
8. When you generate code into a real project, add a compile check or focused
   test rather than assuming it builds.

## Module Scope And Stability

Yggdrasil addresses are derived from the node key, so the network surface is
small and stable. Prefer the parts the library treats as finished; flag the
rest explicitly instead of building on it.

| Package | Status | Use it through |
| --- | --- | --- |
| root `ratatoskr` | Stable, primary API | `New`, `EnableSOCKS`, `Ask`/`AskAddr`, `Snapshot`, `Close` |
| `mod/core` | Stable contract | the `core.Interface` methods embedded on the node |
| `mod/peermgr` | Stable | `ratatoskr.ConfigObj.Peers` |
| `mod/socks` | Stable | `node.EnableSOCKS` (do not instantiate directly) |
| `mod/resolver` | Stable | created automatically by `EnableSOCKS`; direct use is advanced |
| `mod/forward` | Stable | `forward.New` → `Add*` → `Start` |
| `mod/sigils` (+ `info`, `services`, `public`, `inet`) | Stable | `ratatoskr.ConfigObj.Sigils` |
| `mod/ninfo` | Stable | `node.Ask` / `node.AskAddr` |
| `mod/probe` | **Advanced / unstable** — requires `UnsafeCore()`, API may change | only when topology/route tracing is explicitly required |
| `mod/settings` | **Work in progress — schema not finalized** | avoid; use `yggdrasil-go/src/config` directly |
| `cmd/*` | Example apps, not importable library API | read as patterns, never import |

**Root vs submodules:** default to `ratatoskr.New` (core + SOCKS + peer manager
+ ninfo + sigils behind one `Close`). Drop to `mod/core` via `core.New` only for
the smallest surface (raw dial/listen), or compose submodules around
`core.Interface` for custom transports and tests. Every submodule
(`peermgr`/`socks`/`forward`/`resolver`/`ninfo`) takes a `core.Interface`. See
[references/overview.md](references/overview.md#root-facade-vs-submodules).

## System Sigils — Publish Here First

NodeInfo is how a node describes itself to the network. Four **built-in
("system") sigils** are parsed automatically by every node, so data you put in
them is recognized by other participants. Reach for these before inventing a
custom sigil; pass them via `ratatoskr.ConfigObj.Sigils`. Keep each tiny — they
share one 16 KB NodeInfo budget.

| Sigil | What belongs here | Why use it |
| --- | --- | --- |
| `info` | Node identity: name, type, location, contacts, description | Lets others know who/what your node is. |
| `services` | Ports you expose inside Yggdrasil, as `name → port` | Service discovery without port scanning. |
| `public` | Your public peering URIs grouped by network | Lets others peer with you directly. |
| `inet` | Your real-world addresses (domains/IPs) | Maps your Yggdrasil node to clearnet identity. |

Put discoverable, standard data in these so the **whole network** can read it.
Only define a custom sigil for data none of the four cover — and remember only
peers running your code will parse it. Constructors validate and return errors;
handle them before `ratatoskr.New`. Details, exact limits, typed access, and
custom-sigil authoring: [references/nodeinfo-sigils.md](references/nodeinfo-sigils.md)
and [references/custom-sigils.md](references/custom-sigils.md).

## Reference Map

Read only the files needed for the current task:

- Embedded-node basics, Ratatoskr vs yggstack, architecture, scope:
  [references/overview.md](references/overview.md)
- `ratatoskr.New`, `ConfigObj`, lifecycle, thread safety, root methods,
  errors, snapshot: [references/root-api.md](references/root-api.md)
- TCP, UDP, `DialContext`, `Listen`, `ListenPacket`, HTTP client/server,
  multicast/admin, testing seams: [references/networking.md](references/networking.md)
- Static peers, peer manager, SOCKS5, resolver, port forwarding:
  [references/peers-socks-forwarding.md](references/peers-socks-forwarding.md)
- NodeInfo, `Ask`/`AskAddr`, built-in sigils with exact limits:
  [references/nodeinfo-sigils.md](references/nodeinfo-sigils.md)
- Typed access and authoring/wiring your own sigils (no raw `map[string]any`):
  [references/custom-sigils.md](references/custom-sigils.md)
- Client/server templates and framework integrations (net/http, fasthttp,
  gRPC, gnet bridge): [references/integrations.md](references/integrations.md)
- Advanced/WIP areas (probe, settings) and CLI/example-app patterns:
  [references/advanced-and-examples.md](references/advanced-and-examples.md)
- Ready-to-adapt application recipes:
  [references/recipes.md](references/recipes.md)

## Common Mistakes

| Mistake | Correction |
| --- | --- |
| Starting an external Yggdrasil daemon for an embedded app | Use `ratatoskr.New`; it runs a userspace gVisor netstack. |
| Recommending `yggstack` for library integration | `yggstack` is an end-user binary. Use Ratatoskr for Go API control. |
| Setting both `cfg.Config.Peers` and `ConfigObj.Peers` | Choose static peers or the peer manager; never both → `ErrPeersConflict`. |
| Passing `Ctx` but skipping `Close` (or vice versa) | Do both: cancellation triggers shutdown, `Close` releases resources. |
| Building URLs without IPv6 brackets | Use `fmt.Sprintf("http://[%s]:%d", ip, port)`. |
| Creating `ninfo.New` for basic queries | Use `node.Ask` / `node.AskAddr`; the root object already builds ninfo. |
| Assembling or registering sigils dynamically at runtime | Sigils are assembled **strictly in code**; duplicate or overwriting keys are hard errors by design, not silent merges. Publish via `ConfigObj.Sigils`; read your own keys from `res.Node.Extra`. |
| Reaching for `UnsafeCore()` / `mod/probe` / `mod/settings` by default | Stay on the root API; these are advanced or work-in-progress. |
| Hardcoding a Go version from README | Read the active `go.mod` of the selected module version. |

## Validation

Keep `SKILL.md` as the navigation layer and put heavy API detail in
`references/`. Forward-test changes against the scenarios in
[evals/evals.json](evals/evals.json) before publishing.
