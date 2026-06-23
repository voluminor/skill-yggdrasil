# Peers, SOCKS, Resolver, And Forwarding

## Contents

- Static peers vs peer manager
- Peer manager
- Runtime peer operations
- SOCKS5 proxy
- Resolver
- Port forwarding
- Operational notes

## Static Peers Vs Peer Manager

Ratatoskr supports two peer paths. Pick exactly one at startup.

Static Yggdrasil peers:

```go
cfg := yggconfig.GenerateConfig()
cfg.AdminListen = "none"
cfg.Peers = []string{"tls://peer.example.net:17117"}

node, err := ratatoskr.New(ratatoskr.ConfigObj{
    Ctx:    ctx,
    Config: cfg,
})
```

Ratatoskr peer manager:

```go
node, err := ratatoskr.New(ratatoskr.ConfigObj{
    Ctx:    ctx,
    Config: cfg, // cfg.Peers must be empty
    Peers: &peermgr.ConfigObj{
        Peers:       []string{"tls://peer1:443", "quic://peer2:443"},
        MaxPerProto: 1,
        BatchSize:   4,
    },
})
```

If both `cfg.Peers` and `ConfigObj.Peers` are set, `ratatoskr.New` returns
`ErrPeersConflict`.

## Peer Manager

Import:

```go
import "github.com/voluminor/ratatoskr/mod/peermgr"
```

Important fields:

| Field | Meaning |
| --- | --- |
| `Peers` | Candidate peer URIs. Required. |
| `Logger` | Optional at root; Ratatoskr injects root logger if nil. Required for direct `peermgr.New`. |
| `MaxPerProto` | Best peers per protocol. `0` or `1` means one best per protocol. `-1` means passive mode. |
| `ProbeTimeout` | Wait timeout per probing batch. Default 10s. |
| `RefreshInterval` | Re-evaluate periodically. Zero only evaluates at start. |
| `BatchSize` | Sliding probing window. `0` or `1` means all candidates in one batch. |
| `MinPeers` | `uint8` threshold for unscheduled re-evaluation. Ignored in passive mode. Otherwise must be `< MaxPerProto` and `< len(Peers)`, or `New` returns `ErrMinPeersTooHigh`/`ErrMinPeersTooMany`. `0` disables it. |
| `OnNoReachablePeers` | Non-blocking callback when no peer responds. |

Allowed URI schemes are `tcp`, `tls`, `quic`, `ws`, and `wss`.

Use `PeerManagerActive` for status:

```go
active := node.PeerManagerActive()
```

Use `PeerManagerOptimize` for manual re-evaluation:

```go
if err := node.PeerManagerOptimize(); err != nil {
    return fmt.Errorf("optimize peers: %w", err)
}
```

## Runtime Peer Operations

For simple runtime peer changes, use the embedded core methods:

```go
if err := node.AddPeer("tls://peer.example.net:17117"); err != nil {
    return err
}
if err := node.RemovePeer("tls://peer.example.net:17117"); err != nil {
    return err
}
node.RetryPeers()
```

Use these for imperative control. Use the peer manager when Ratatoskr should
probe, select, and maintain peers from a candidate list.

## SOCKS5 Proxy

Use the root package for normal applications:

```go
err := node.EnableSOCKS(ratatoskr.SOCKSConfigObj{
    Addr:           "127.0.0.1:1080",
    Nameserver:     "", // only literals and .pk.ygg
    MaxConnections: 100,
})
if err != nil {
    return fmt.Errorf("enable socks: %w", err)
}
defer node.DisableSOCKS()
```

Address rules:

| Address | Listener |
| --- | --- |
| `127.0.0.1:1080` | TCP |
| `[::1]:1080` | TCP |
| `/tmp/ygg.sock` | Unix socket |
| `./local.sock` | Unix socket |

Unix sockets get stale-file handling. Live sockets return an error; symlinks
are not removed.

## Resolver

`EnableSOCKS` creates a resolver automatically. Direct resolver use is for
advanced integrations:

```go
r := resolver.New(node, "[200:abcd::53]:53")
_, ip, err := r.Resolve(ctx, "service.example.ygg")
```

Resolution order:

1. `<public-key>.pk.ygg` maps to an IPv6 address via `address.AddrForKey`.
2. IP literals return as-is.
3. DNS over Yggdrasil is used only when a nameserver is configured.

Empty nameserver means DNS is disabled; `.pk.ygg` and literals still work.

## Port Forwarding

Import:

```go
import "github.com/voluminor/ratatoskr/mod/forward"
```

Create mappings before `Start`:

```go
mgr := forward.New(logger, 120*time.Second)

mgr.AddLocalTCP(forward.TCPMappingObj{
    Listen: &net.TCPAddr{IP: net.IPv4(127, 0, 0, 1), Port: 8080},
    Mapped: &net.TCPAddr{IP: net.ParseIP("200:abcd::1"), Port: 80},
})

mgr.Start(ctx, node)
defer func() {
    cancel()
    mgr.Wait()
}()
```

Mapping directions:

| Method | Listens on | Connects to |
| --- | --- | --- |
| `AddLocalTCP` | Local TCP | Yggdrasil TCP |
| `AddRemoteTCP` | Yggdrasil TCP | Local TCP |
| `AddLocalUDP` | Local UDP | Yggdrasil UDP |
| `AddRemoteUDP` | Yggdrasil UDP | Local UDP |

Settings before `Start`:

| Method | Meaning |
| --- | --- |
| `SetTimeout` | UDP session inactivity timeout. |
| `SetTCPCloseTimeout` | Time to wait for peer after one side closes. |
| `SetMaxUDPSessions` | Per-mapping UDP session limit. Zero is unlimited. |
| `ClearLocal` / `ClearRemote` | Remove configured mappings before start. |

`forward.New` panics if the UDP session timeout is not positive.

## Operational Notes

- `OnNoReachablePeers` and other peer-manager callbacks run on the manager
  goroutine and must not block.
- Configure forwarding rules before `Start`; on shutdown cancel the forwarding
  context, then call `Wait`.
