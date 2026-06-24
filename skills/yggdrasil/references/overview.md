# Ratatoskr Overview

## Contents

- What Ratatoskr is
- Ratatoskr vs yggstack
- Dependency setup
- Application decision guide
- Architecture
- Module scope and stability
- Root facade vs submodules
- Source freshness checks

## What Ratatoskr Is

Ratatoskr is a Go library for embedding a Yggdrasil node inside an
application. It runs TCP and UDP in userspace through a gVisor netstack on top
of `yggdrasil-go`, so typical applications do not need:

- a TUN interface;
- root privileges;
- an external `yggdrasil` daemon;
- shelling out to a CLI for normal network operations.

The application uses standard Go interfaces:

- `DialContext(ctx, network, address)` for outbound TCP/UDP;
- `Listen(network, address)` for TCP services;
- `ListenPacket(network, address)` for UDP services;
- `net.Conn`, `net.Listener`, `net.PacketConn`, and `http.Transport`.

The core depends on the `core.Interface` contract, not on a concrete type, so
SOCKS, peer manager, forwarding, and tests can be wired against an interface.

## Ratatoskr vs yggstack

Choose Ratatoskr when the user wants a Go application, library integration, a
testable embedded network node, or direct lifecycle control.

Choose `yggstack` only when the user asks for a ready-made command-line binary
for end users, such as a standalone SOCKS proxy or CLI forwarding tool.

Do not recommend `yggstack` as the main answer for "write a Go app using
Yggdrasil". The root package `github.com/voluminor/ratatoskr` is the intended
API. (Ratatoskr even ships its own `cmd/yggstack` built on the library — proof
that yggstack-style tools are downstream of the library, not above it.)

## Dependency Setup

Use the current module version selected by the user's project:

```bash
go get github.com/voluminor/ratatoskr
go list -m github.com/voluminor/ratatoskr
```

Ratatoskr imports `yggdrasil-go` config types for node configuration:

```go
import yggconfig "github.com/yggdrasil-network/yggdrasil-go/src/config"
```

Check the active Ratatoskr `go.mod` before stating a minimum Go version. README
text can lag behind the module directive — verify, do not assume.

## Application Decision Guide

| User goal | Main API | Read next |
| --- | --- | --- |
| Embed a node and make outbound requests | `ratatoskr.New`, `node.DialContext` | `root-api.md`, `networking.md` |
| Serve HTTP inside Yggdrasil | `node.Listen("tcp", "[addr]:port")` | `networking.md`, `recipes.md` |
| Expose SOCKS5 to local apps | `node.EnableSOCKS` | `peers-socks-forwarding.md` |
| Keep the best peers connected | `ConfigObj.Peers` with `peermgr.ConfigObj` | `peers-socks-forwarding.md` |
| Forward local ports to/from Yggdrasil | `mod/forward.ManagerObj` | `peers-socks-forwarding.md` |
| Query remote NodeInfo | `node.Ask`, `node.AskAddr` | `nodeinfo-sigils.md` |
| Publish structured NodeInfo | `ConfigObj.Sigils` | `nodeinfo-sigils.md` |
| Explore topology or routes (advanced) | `mod/probe` with `UnsafeCore()` | `advanced-and-examples.md` |

## Architecture

```
Application
    │
    ▼
ratatoskr.Obj  ──────────── facade (embeds core.Interface)
    │  ├── SOCKS5 proxy      (EnableSOCKS / DisableSOCKS)
    │  ├── Peer Manager      (ConfigObj.Peers → mod/peermgr)
    │  └── ninfo             (Ask / AskAddr → mod/ninfo)
    ▼
mod/core.Obj  ───────────── core.Interface implementation
    │  └── netstack          userspace TCP/UDP (gVisor)
    ▼
yggdrasil-go/core  +  gVisor netstack
```

`ratatoskr.Obj` embeds `core.Interface`, so all network methods
(`DialContext`, `Listen`, `ListenPacket`, `Address`, `Subnet`, `PublicKey`,
`MTU`, peer operations, multicast, admin) are available directly on `node`.
SOCKS5, the peer manager, and ninfo are optional components controlled through
`Obj` methods.

The root object is the default for applications. Direct `mod/core` usage is
for lower-level packages and advanced integrations, and even then through the
`core.Interface` contract rather than the concrete `*core.Obj`.

## Module Scope And Stability

Document and build on the finished surface; flag the rest.

- **Stable, build freely:** root `ratatoskr`, `mod/core` (via `core.Interface`),
  `mod/peermgr`, `mod/socks` (via `EnableSOCKS`), `mod/resolver`, `mod/forward`,
  `mod/sigils` (+ `info`, `services`, `public`, `inet`), `mod/ninfo`.
- **Advanced / unstable seam:** `mod/probe` — useful for topology and route
  tracing but requires `UnsafeCore()` (a documented "unstable API") and is
  itself still changing. Use only when explicitly required; see
  `advanced-and-examples.md`.
- **Work in progress:** `mod/settings` — the README marks it "WIP — schema not
  finalized". Do not build configuration tooling on it; use
  `yggdrasil-go/src/config` directly.
- **Not a library API:** everything under `cmd/*` (CLI, `yggstack`, embedded
  examples, mobile wrapper). Read it for patterns; never import it.

The public API uses `yggcore.Logger` everywhere — Ratatoskr v0.2.0 dropped the
old `gologme` logger parameter from `EnableMulticast()`, which now takes no
arguments. (`gologme/log` may still appear as an *indirect* dependency pulled in
by `yggdrasil-go`; that is upstream, not Ratatoskr's public surface.)

## Root Facade Vs Submodules

Agents should default to the root facade and only drop to submodules for a
concrete reason. Both are first-class; the choice is about surface and control,
not micro-optimization.

`ratatoskr.New` bundles core networking + optional SOCKS + optional peer
manager + ninfo (`Ask`/`AskAddr`) + sigil assembly behind **one object and one
`Close`**. It is the right default for applications.

Assemble from submodules when one of these is true:

| Situation | Build from |
| --- | --- |
| Typical app: dial/listen, maybe SOCKS, peers, or NodeInfo | root `ratatoskr.New` |
| You only need TCP/UDP `DialContext`/`Listen` and want the smallest surface | `mod/core` via `core.New` |
| You want to inject a custom or fake transport, or unit-test wiring | depend on `core.Interface`; compose submodules yourself |
| Offline NodeInfo parsing or sigil tooling, no running node | `mod/ninfo` (`Parse`) + `mod/sigils` |
| Forwarding between conns you already own | `mod/forward` with any `core.Interface` |

A `mod/core`-only app does not import the SOCKS (`go-socks5`) or peer-manager
packages at all. Choose submodules for **control and testability**; the smaller
dependency surface is a bonus, not the main reason.

Core-only example (no SOCKS, no peer manager, no ninfo):

```go
import (
    "github.com/voluminor/ratatoskr/mod/core"
    yggconfig "github.com/yggdrasil-network/yggdrasil-go/src/config"
)

cfg := yggconfig.GenerateConfig()
cfg.AdminListen = "none"
cfg.Peers = []string{"tls://peer.example.net:17117"}

node, err := core.New(core.ConfigObj{Config: cfg, Logger: logger})
if err != nil {
    return err
}
defer node.Close()

conn, err := node.DialContext(ctx, "tcp", "[200:abcd::1]:80")
```

`core.ConfigObj` fields: `Config`, `Logger`, `CoreStopTimeout`, and
`RSTQueueSize` (RST deferred-queue size; `0` → 100). `core.New` returns a
`*core.Obj` that satisfies `core.Interface`. Every other submodule
(`peermgr.New`, `socks.New`, `forward.New`, `resolver.New`, `ninfo.New`) takes a
`core.Interface`, so you can add exactly the pieces you need — or wire them
around a fake for tests.

## Source Freshness Checks

When working inside a user project, resolve the real source of the selected
version from the module cache rather than assuming any local checkout path:

```bash
dir=$(go list -m -f '{{.Dir}}' github.com/voluminor/ratatoskr)
sed -n '1,40p' "$dir/go.mod"
go doc github.com/voluminor/ratatoskr
```

Prefer the version selected in the user's `go.mod` over assumptions from older
examples or from this skill.
