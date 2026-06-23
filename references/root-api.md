# Root Package API

## Contents

- Import path
- Creating a node
- Configuration fields
- Stable addresses
- Lifecycle
- Thread safety
- Root object methods
- Peer manager hooks
- NodeInfo hooks
- Snapshot
- Errors and error handling
- Testing without a live network

## Import Path

Use the root package for applications:

```go
import "github.com/voluminor/ratatoskr"
```

Use `yggdrasil-go/src/config` only when a custom `*config.NodeConfig` is
needed:

```go
import yggconfig "github.com/yggdrasil-network/yggdrasil-go/src/config"
```

## Creating A Node

Minimal static-peer startup:

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

cfg := yggconfig.GenerateConfig()
cfg.AdminListen = "none"
cfg.Peers = []string{"tls://peer.example.net:17117"}

node, err := ratatoskr.New(ratatoskr.ConfigObj{
    Ctx:    ctx,
    Config: cfg,
})
if err != nil {
    return err
}
defer node.Close()
```

If `Config` is nil, Ratatoskr generates a random config and disables
`AdminListen`.

## Stable Addresses

A Yggdrasil IPv6 address is derived from the node's Ed25519 public key. A random
config means a new key and a new address on every start. For a service that must
keep the same address across restarts, persist the private key and reuse it.

`config.NodeConfig.PrivateKey` is a `KeyBytes` (raw 64-byte Ed25519 private key,
hex-encoded in JSON). Two reuse options:

```go
// Option A — keep the bytes yourself.
cfg := yggconfig.GenerateConfig()        // first run only; then store cfg.PrivateKey
cfg.AdminListen = "none"
cfg.PrivateKey = yggconfig.KeyBytes(stored) // later runs: stored is the 64-byte key

// Option B — point at a PEM file managed by yggdrasil-go.
cfg.PrivateKeyPath = "/var/lib/myapp/key.pem" // loaded via UnmarshalPEMPrivateKey
```

Treat the key as a secret: anyone holding it can impersonate the node's address.
There is no separate "set address" — the address always follows the key.

## Configuration Fields

`ratatoskr.ConfigObj`:

| Field | Use |
| --- | --- |
| `Ctx context.Context` | Parent context; cancellation calls `Close`. Still call `Close` yourself. |
| `Config *config.NodeConfig` | Yggdrasil config. Nil generates random keys. |
| `Logger yggcore.Logger` | Nil discards logs. Use a real logger in services. |
| `CoreStopTimeout time.Duration` | Limit wait for `core.Stop`; zero waits indefinitely. |
| `Peers *peermgr.ConfigObj` | Enables Ratatoskr peer manager. Do not also set `Config.Peers`. |
| `Sigils []sigils.Interface` | Publishes structured NodeInfo. |

`ratatoskr.SOCKSConfigObj`:

| Field | Use |
| --- | --- |
| `Addr` | TCP address like `127.0.0.1:1080` or Unix socket like `/tmp/ygg.sock`. |
| `Nameserver` | Yggdrasil DNS server for `.ygg` names. Empty enables only literals and `.pk.ygg`. |
| `Verbose` | Log each SOCKS connection. |
| `MaxConnections` | Maximum simultaneous SOCKS connections. Zero means unlimited. |

## Lifecycle

Use both context cancellation and explicit close:

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

node, err := ratatoskr.New(ratatoskr.ConfigObj{
    Ctx:             ctx,
    Config:          cfg,
    CoreStopTimeout: 5 * time.Second,
})
if err != nil {
    return err
}
defer node.Close()

<-ctx.Done()
```

`Close` is safe to call repeatedly. It stops the peer manager, SOCKS server,
ninfo module, listeners, and the underlying core.

## Thread Safety

All public methods of `*ratatoskr.Obj` are safe for concurrent use. Useful
guarantees when designing services:

| Method / group | Guarantee |
| --- | --- |
| `DialContext`, `Listen`, `ListenPacket` | Thread-safe; netstack swapped via `atomic.Pointer`. |
| `EnableSOCKS` / `DisableSOCKS` | Mutex-protected. |
| `EnableMulticast` / `EnableAdmin` | Protected by `sync.RWMutex`. |
| `AddPeer` / `RemovePeer` | Delegate to `yggdrasil-go/core` (thread-safe). |
| `PeerManagerActive` | Returns a copy; mutex-protected. |
| `PeerManagerOptimize` | Blocks; serialized via an internal mutex. |
| `Ask` / `AskAddr` | Thread-safe; network call runs in a goroutine, cancellable via `ctx`. |
| `Close` | Idempotent (`sync.Once`). |
| `Snapshot` | Thread-safe; aggregates the above. |

You can call these from multiple goroutines without external locking.

## Root Object Methods

`*ratatoskr.Obj` embeds `core.Interface`. These methods are available directly:

| Method | Use |
| --- | --- |
| `DialContext(ctx, network, address)` | Outgoing TCP/UDP over Yggdrasil. |
| `Listen(network, address)` | TCP service inside Yggdrasil. |
| `ListenPacket(network, address)` | UDP service inside Yggdrasil. |
| `Address()` | Node IPv6 address. |
| `Subnet()` | Node routable subnet. |
| `PublicKey()` | Ed25519 public key. |
| `MTU()` | Stack MTU. |
| `GetPeers()` | Yggdrasil peer list and metrics. |
| `AddPeer(uri)` / `RemovePeer(uri)` | Runtime peer changes. |
| `RetryPeers()` | Immediately retry disconnected peers (no-op if core type differs). |
| `EnableMulticast()` / `DisableMulticast()` | LAN discovery. No arguments; interfaces come from config. |
| `EnableAdmin(addr)` / `DisableAdmin()` | Admin socket. Off by default (`AdminListen = "none"`). |

Supported network strings are `tcp`, `tcp6`, `udp`, and `udp6`.

`EnableMulticast()` takes no arguments; it reads interfaces from
`cfg.Config.MulticastInterfaces` (`GenerateConfig()` sets a working default).

## Peer Manager Hooks

If `ConfigObj.Peers` is nil, `PeerManagerActive` returns nil and
`PeerManagerOptimize` returns `ErrPeerManagerNotEnabled`.

```go
active := node.PeerManagerActive()
if err := node.PeerManagerOptimize(); err != nil {
    // handle ErrPeerManagerNotEnabled or optimize failure
}
```

Never set both:

```go
cfg.Peers = []string{"tls://peer:17117"}       // static Yggdrasil peers
ratatoskr.ConfigObj{Peers: &peermgr.ConfigObj{}} // peer manager
```

That combination returns `ErrPeersConflict`.

## NodeInfo Hooks

Use the root object for ordinary NodeInfo queries:

```go
ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
defer cancel()

res, err := node.AskAddr(ctx, "200:abcd::1")
if err != nil {
    return err
}
fmt.Println(res.RTT, res.Node.String())
```

Do not create `ninfo.New` from application code unless you are bypassing the
root facade or building a lower-level tool. `ratatoskr.New` creates ninfo for
`Ask` and `AskAddr`.

## Snapshot

Use `Snapshot` for operational status:

```go
snap := node.Snapshot()
fmt.Println(snap.Address, snap.PublicKey, snap.SOCKS.Enabled)
```

Snapshot contains:

- address, subnet, public key, MTU;
- dropped RST count when the underlying core is `*core.Obj`;
- peers with URI, state, latency, traffic, uptime, and last error;
- active peer manager peers when enabled;
- SOCKS state.

## Errors And Error Handling

Root package errors:

| Error | Meaning |
| --- | --- |
| `ErrPeersConflict` | `Config.Peers` and `ConfigObj.Peers` were both set. |
| `ErrPeerManagerNotEnabled` | Peer-manager method called without peer manager. |

Both fire from `ratatoskr.New`/`PeerManagerOptimize` synchronously, so they are
checkable at startup.

## Testing Without A Live Network

Ratatoskr reaches the network through `core.Interface`, so code can depend on
that contract (or the narrower `DialContext`/`Listen` it actually uses) and run
against a fake instead of a live node. The mockable seams are listed in
[networking.md](networking.md#test-seams).
