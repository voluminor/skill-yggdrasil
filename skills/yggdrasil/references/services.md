# Peers, SOCKS, resolver, and forwarding

This reference covers optional network services around a Ratatoskr node. Read
[load-safety.md](load-safety.md) before exposing any listener beyond a trusted
local boundary. Exact v1.1.0 defaults are in [api-v1.1.md](api-v1.1.md).

## Choose peer ownership

Use one owner for a peer set:

| Requirement | Model |
| --- | --- |
| Small fixed deployment | static `config.NodeConfig.Peers` |
| Select the best candidates or recover peer health | root `ConfigObj.Peers` with `peermgr.ConfigObj` |
| Keep every configured candidate under manager ownership | peer manager with `Passive: true` |
| External control plane chooses peers | root `AddPeer` and `RemovePeer` |

Do not set both static peers and a root peer-manager config. `ratatoskr.New`
returns `ErrPeersConflict`.

### Static peers

```go
cfg := yggconfig.GenerateConfig()
cfg.AdminListen = "none"
cfg.Peers = []string{"tls://peer.example.net:17117"}

node, err := ratatoskr.New(ratatoskr.ConfigObj{
    Ctx:    ctx,
    Config: cfg,
})
```

Static peers are the simplest choice when the application already trusts and
operates a short list.

### Managed peers

```go
node, err := ratatoskr.New(ratatoskr.ConfigObj{
    Ctx:    ctx,
    Config: cfg, // cfg.Peers must be empty
    Peers: &peermgr.ConfigObj{
        Peers:       candidates,
        MaxPerProto: 1,
    },
})
```

The manager probes candidates in bounded batches, groups selection by transport
scheme, tracks health, and backs off failed candidates. Probing is network work;
do not enable it when static peers solve the problem.

Passive mode keeps all configured peers without latency selection:

```go
Peers: &peermgr.ConfigObj{
    Peers:   candidates,
    Passive: true,
}
```

For `NoReachablePeers`, use a capacity-one channel. Sends are non-blocking and
the caller must not close the channel before manager shutdown completes.

### Runtime operations

```go
if err := node.AddPeer(uri); err != nil {
    return err
}
peers := node.GetPeers()
if err := node.RemovePeer(uri); err != nil {
    return err
}
```

Use `node.Core().RetryPeers()` for an explicit reconnect request. Do not mix an
external controller with peer-manager ownership for the same URI set.

## Root SOCKS5 service

For ordinary applications, let the root wire SOCKS to the node and resolver:

```go
err := node.EnableSOCKS(ratatoskr.SOCKSConfigObj{
    Addr:       "127.0.0.1:1080",
    Nameserver: "",
})
if err != nil {
    return fmt.Errorf("enable SOCKS: %w", err)
}
defer node.DisableSOCKS()
```

An empty nameserver still permits Yggdrasil IP literals and canonical
`<public-key>.pk.ygg` names. Configure a nameserver only for DNS names that
require remote resolution.

Loopback TCP is the portable local endpoint. Unix sockets need a confirmed
platform and a private directory; see [platforms.md](platforms.md).

The root service has bounded defaults for accepted connections, handshakes,
dials, idle tunnels, UDP targets, queues, and resolver work. One exception
within SOCKS policy is the per-principal UDP target limit, which is unlimited
unless set. For a listener beyond loopback, decide explicitly:

- who can connect;
- whether username/password authentication is required;
- how many connections the process and backend can serve;
- how many UDP targets one principal may create;
- whether DNS is allowed and which Yggdrasil nameserver is trusted.

`SetSOCKSMaxConnections` changes the active connection cap at runtime. Do not
raise it automatically in response to rejected work.

Use `node.Snapshot().SOCKS` to observe active connections, UDP targets,
pending target creation, rejections, and packet drops.

## Direct resolver use

Root SOCKS creates and owns its resolver. Instantiate `mod/resolver` only for a
custom stack or direct resolution workflow:

```go
r, err := resolver.New(resolver.ConfigObj{
    Dialer:     node,
    Nameserver: "[200:abcd::53]:53",
})
if err != nil {
    return err
}
defer r.Close()
```

Resolution order:

1. canonical `.pk.ygg` public-key domain without network I/O;
2. IP literal, restricted to the Yggdrasil range;
3. DNS over the configured Yggdrasil nameserver.

Distinct concurrent DNS work is bounded. `resolver.ErrLookupBusy` means the
resolver is shedding a new distinct lookup. Existing same-name work remains
joinable. Back off or defer the new request; do not create another resolver to
bypass the bound.

## Forwarding

`mod/forward` bridges host sockets and the userspace Yggdrasil stack. One
constructor validates, clones, binds, and starts an immutable mapping set.

Directions:

| Field | Listener | Destination |
| --- | --- | --- |
| `LocalTCP` | host TCP | Yggdrasil TCP |
| `RemoteTCP` | Yggdrasil TCP | host TCP |
| `LocalUDP` | host UDP | Yggdrasil UDP |
| `RemoteUDP` | Yggdrasil UDP | host UDP |

Example: expose a local backend to the mesh.

```go
fwd, err := forward.New(forward.ConfigObj{
    Node: node,
    RemoteTCP: []forward.TCPMappingObj{{
        Listen: &net.TCPAddr{IP: node.Address(), Port: 9000},
        Mapped: &net.TCPAddr{IP: net.IPv4(127, 0, 0, 1), Port: 9000},
    }},
    MaxTCPConnections: 64,
})
if err != nil {
    return err
}
defer fwd.Close()
```

The sample limit is a starting point, not a production recommendation. Since
the Yggdrasil listener accepts untrusted traffic, it needs a positive TCP cap.
No UDP cap is needed because this object has no UDP mapping.

For UDP mappings:

- set a positive `UDPTimeout`;
- set `MaxUDPSessions` when untrusted clients can reach the listener;
- keep packet size bounded, normally from node MTU;
- monitor session and reverse packet-drop counters;
- control offered rate instead of increasing concurrency after loss.

`MaxTCPConnections` and `MaxUDPSessions` are object-wide and zero means
unlimited. One hot mapping can consume the corresponding shared budget.

## Lifecycle

The root owns root SOCKS and its resolver. It does not own `forward.Obj` or a
resolver created directly by the application.

Shutdown order:

1. stop new application work;
2. close forwarding and directly created services;
3. close the root node.

Creating a new forwarding object is the v1 way to replace mappings. Bind the
replacement before retiring the old object only when the local addresses do not
conflict and the application has budget for overlap.

## Failure policy

- Invalid configuration: correct it; no retry.
- Busy resolver or admission signal: defer with bounded jittered backoff.
- Backend dial failure: avoid rapid reconnect loops across the mesh.
- Growing drops: reduce offered load or investigate backend capacity before
  raising limits.
- Partial shutdown timeout: report it and preserve the root close error with
  `errors.Is` checks.
