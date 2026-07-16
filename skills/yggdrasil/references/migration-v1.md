# Migrating Ratatoskr code to v1

Ratatoskr v1.1.0 is the first stable release. Its exported root and module APIs
carry the v1 compatibility guarantee. Pre-v1 code is not source-compatible and
must be migrated as one versioned change; do not mix old and new contracts.

## Establish the active version

When the user's project and Go toolchain are available, inspect the selected
module:

```text
go list -m github.com/voluminor/ratatoskr
go doc github.com/voluminor/ratatoskr
```

If inspection is unavailable, state that this guide describes v1.1.0. Do not
install Go or change module versions automatically.

## Breaking changes

| Pre-v1 pattern | v1 contract |
| --- | --- |
| Root `Obj` embeds `core.Interface` | Root exposes common networking methods; advanced methods are behind `node.Core()` |
| `UnsafeCore()` | Removed; use `node.Core()` |
| `CoreStopTimeout` | Replaced by root `CloseTimeout` |
| Zero shutdown timeout waits forever | Zero `CloseTimeout` uses 10 seconds; negative is invalid |
| Root calls `EnableMulticast`, `EnableAdmin`, or `RetryPeers` directly | Call `node.Core().EnableMulticast()`, `EnableAdmin(...)`, or `RetryPeers()` |
| Positional module constructors | Modules use their own `ConfigObj` values |
| Mutable forwarding: `Add*`, `SetTimeout`, `Start`, `Wait` | One transactional `forward.New(forward.ConfigObj{...})`; mappings are immutable |
| Forward object stopped separately from waiting | `forward.Obj.Close()` cancels and waits |
| `MaxPerProto: -1` selects passive peer mode | Negative is invalid; use `Passive: true` |
| `OnNoReachablePeers` callback | `NoReachablePeers chan<- struct{}` is a non-blocking notification channel |
| SOCKS `MaxConnections: 0` means unlimited | Zero uses the bounded default 256; negative is unlimited |
| Custom sigils remain raw in `Extra` | Root `ConfigObj.Sigils` also registers non-built-in remote parsers |
| Probe is unstable and needs `UnsafeCore` | `mod/probe` is stable; construct it with `Source: node.Core()` |
| `mod/settings` persists configuration | Package removed; use `yggdrasil-go/src/config` directly |

## Root boundary

Common methods remain on `*ratatoskr.Obj`:

- `DialContext`, `Listen`, `ListenPacket`;
- `Address`, `Subnet`, `PublicKey`, `MTU`;
- `AddPeer`, `RemovePeer`, `GetPeers`;
- SOCKS, peer-manager, NodeInfo, snapshot, and close methods.

Use `node.Core()` for multicast, admin integration, peer retry, and the local
diagnostic data required by `probe`.

## Lifecycle migration

Pass an application context when useful and still close the node explicitly:

```go
node, err := ratatoskr.New(ratatoskr.ConfigObj{
    Ctx:          ctx,
    Config:       cfg,
    CloseTimeout: 10 * time.Second,
})
if err != nil {
    return err
}
defer node.Close()
```

Root `Close` is idempotent, concurrent-safe, and dependency-aware. If its
budget expires, it returns an error matching `ErrCloseTimedOut`; unfinished
teardown continues in the background.

The root does not own `forward.Obj`, `probe.Obj`, framework servers, or modules
created directly by the application. Close those dependants before the root.

## Forwarding migration

Move all mappings and limits into one constructor:

```go
fwd, err := forward.New(forward.ConfigObj{
    Node: node,
    LocalTCP: []forward.TCPMappingObj{{
        Listen: &net.TCPAddr{IP: net.IPv4(127, 0, 0, 1), Port: 8080},
        Mapped: &net.TCPAddr{IP: net.ParseIP("200:abcd::1"), Port: 80},
    }},
    MaxTCPConnections: 64,
})
if err != nil {
    return err
}
defer fwd.Close()
```

The number is an example, not a universal production value. Set a positive cap
for each enabled protocol exposed to untrusted traffic. UDP mappings also need
a positive `UDPTimeout`.

## Peer migration

Configure static peers in `config.NodeConfig.Peers` or managed candidates in
`ratatoskr.ConfigObj.Peers`, never both. The root returns `ErrPeersConflict` for
that startup conflict.

For passive management:

```go
Peers: &peermgr.ConfigObj{
    Peers:   candidates,
    Passive: true,
}
```

Use a capacity-one notification channel for `NoReachablePeers` and do not close
it before the manager or root finishes closing.

## Sigil migration

Built-in sigils remain the first choice. A custom sigil supplied through root
`ConfigObj.Sigils` is cloned into the local NodeInfo assembly and registered as
a remote parser. Read recognized remote data from `res.Node.Sigils[name]`.
`res.Node.Extra` contains unclaimed keys.

## Removed settings package

Use `github.com/yggdrasil-network/yggdrasil-go/src/config` to generate, read,
and persist `config.NodeConfig`. The private key defines the stable Yggdrasil
identity and must be treated as a secret.

## Migration result

A migrated application targets one selected Ratatoskr version, owns every
standalone component explicitly, and handles v1 load-shedding errors without
reintroducing unbounded queues or retries.
