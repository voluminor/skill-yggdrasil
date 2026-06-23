# Advanced And Example-App Areas

This file covers parts of Ratatoskr that are **not** part of the stable,
recommended surface. Reach for them only when a task explicitly requires it,
and warn the user about the trade-off.

## Contents

- Topology probe (advanced, unstable)
- Settings module (work in progress)
- CLI and example apps (patterns, not imports)

## Topology Probe — Advanced, Unstable

`mod/probe` explores Yggdrasil topology and traces routes without a real admin
socket. It is genuinely useful for diagnostics, but treat it as advanced:

- it requires `*core.Obj` and its `UnsafeCore()` method, which the library
  documents as an **unstable API**;
- the package was recently reworked (it grew out of the former
  `mod/traceroute`) and is still changing.

Do not put `mod/probe` in ordinary networking code, and do not present it as the
default way to do anything. When the user explicitly needs route tracing or a
topology view:

```go
coreNode, ok := node.Interface.(*core.Obj)
if !ok {
    return errors.New("probe requires the default core implementation")
}
tr, err := probe.New(coreNode.UnsafeCore(), logger)
if err != nil {
    return err
}
defer tr.Close()

ctx, cancel := context.WithTimeout(parent, 10*time.Second)
defer cancel()
trace, err := tr.Trace(ctx, publicKey) // route trace toward a node key
```

Always check the live API with `go doc github.com/voluminor/ratatoskr/mod/probe`
before generating probe code, because method shapes here can move between
versions. State plainly to the user that this part may change.

## Settings Module — Work In Progress

`mod/settings` loads, parses, and saves Ratatoskr configuration. The module
README marks it **"WIP — schema not finalized"**, so do not build configuration
tooling on it and do not document its functions as stable.

For configuration that needs to be stable today, use the upstream config types
directly:

```go
import yggconfig "github.com/yggdrasil-network/yggdrasil-go/src/config"

cfg := yggconfig.GenerateConfig()
// persist/restore cfg yourself (e.g. yggconfig's own JSON), keeping cfg.PrivateKey safe
```

If a user specifically wants `mod/settings`, tell them it is experimental and
verify every signature against `go doc` first.

## CLI And Example Apps — Patterns, Not Imports

Everything under `cmd/*` is an application or example, not a library API. Never
import `cmd/...` from library code; read it for architecture patterns only.

| Path | What to learn from it |
| --- | --- |
| `cmd/embedded/tiny-http` | Minimal: plain HTTP plus a Yggdrasil HTTP listener. |
| `cmd/embedded/tiny-chat` | Peer-to-peer HTTP chat over Yggdrasil. |
| `cmd/embedded/http` | Fuller service: peer manager, plain + Yggdrasil serving. |
| `cmd/embedded/mobile` | gomobile-friendly start/stop wrapper with SOCKS, forwarding, callbacks — the cleanest reference for long-lived start/stop lifecycle. |
| `cmd/yggstack` | A yggstack-compatible binary built on the library. |
| `cmd/ratatoskr` | Operator CLI: key generation, address derivation, config import/export, `Ask`, peer info. |

The `cmd/embedded/mobile` start/stop pattern generalizes well to desktop apps
and long-lived services:

1. Store config and mapping changes while stopped; reject mutation while running.
2. `ratatoskr.New` to start.
3. Enable SOCKS / multicast only after a successful start.
4. Start forwarding with its own cancelable context.
5. On stop: cancel forwarding, `node.Close()`, then `mgr.Wait()`, then clear
   running state.

Use these as patterns to adapt, not as packages to depend on. For operator
tasks (key generation, address derivation, config conversion), point users at
the `cmd/ratatoskr` CLI rather than reimplementing them, and prefer library
APIs inside application code.
