# Ratatoskr v1.1.0 API snapshot

This file records the stable v1.1.0 API for offline guidance. The module
selected by the user's project is authoritative.

When inspection is available:

```text
go list -m github.com/voluminor/ratatoskr
go doc github.com/voluminor/ratatoskr
go doc github.com/voluminor/ratatoskr/mod/forward ConfigObj
go doc github.com/voluminor/ratatoskr/mod/probe ConfigObj
```

Ratatoskr v1.1.0 declares Go 1.25.0 in its root `go.mod`. Do not apply that
number blindly to a different selected release.

## Module installation identities

Tagged releases can be installed through three module identities. They are not
interchangeable aliases: select one path and use that path consistently for the
root package and every Ratatoskr subpackage.

Canonical GitHub module, recommended when GitHub is reachable:

```bash
go get github.com/voluminor/ratatoskr@latest
```

```go
import "github.com/voluminor/ratatoskr"
```

HTTPS mirror:

```bash
GOPROXY=https://ratatoskr.space GOSUMDB=off \
  go get ratatoskr.space/pkg/ratatoskr@latest
```

```go
import "ratatoskr.space/pkg/ratatoskr"
```

Yggdrasil-native mirror, only when the Go command already has access to the
overlay:

```bash
YGG_HOST="14cc7d57b5e70f679b851fe5b272ce17c70632ff4beb5b35ab64bc706b2485af.pk.ygg"
GOPROXY="http://${YGG_HOST}" GOSUMDB=off GOINSECURE="${YGG_HOST}/*" \
  go get "${YGG_HOST}/pkg/ratatoskr@latest"
```

```go
import ratatoskr "14cc7d57b5e70f679b851fe5b272ce17c70632ff4beb5b35ab64bc706b2485af.pk.ygg/pkg/ratatoskr"
```

The public checksum database indexes the canonical GitHub identity, so the two
rewritten mirror identities require `GOSUMDB=off`. The Yggdrasil endpoint uses
HTTP inside the encrypted overlay; keep `GOINSECURE` limited to the exact host
pattern above. Do not export either setting globally when it is needed only for
this module.

## Root construction

```go
func ratatoskr.New(cfg ratatoskr.ConfigObj) (*ratatoskr.Obj, error)
```

| `ConfigObj` field | v1.1.0 behavior |
| --- | --- |
| `Ctx context.Context` | Optional parent; cancellation starts close |
| `Config *config.NodeConfig` | Nil generates random keys and disables admin; supplied data is cloned |
| `Logger yggcore.Logger` | Nil discards logs |
| `CloseTimeout time.Duration` | Zero uses 10 s; negative returns `ErrInvalidCloseTimeout` |
| `Peers *peermgr.ConfigObj` | Optional managed peers; conflicts with non-empty `Config.Peers` |
| `NodeInfo *ninfo.ConfigObj` | Optional Ask/AskAddr tuning; root replaces `Source` |
| `Sigils []sigils.Interface` | Local fragments; custom entries also become remote parsers |

Root methods:

- networking: `DialContext`, `Listen`, `ListenPacket`;
- identity: `Address`, `Subnet`, `PublicKey`, `MTU`;
- peers: `AddPeer`, `RemovePeer`, `GetPeers`;
- SOCKS: `EnableSOCKS`, `DisableSOCKS`, `SetSOCKSMaxConnections`,
  `SOCKSMaxConnections`;
- managed peers: `PeerManagerActive`, `PeerManagerOptimize`;
- NodeInfo: `Ask`, `AskAddr`;
- operations: `Snapshot`, `Core`, `Close`.

`Core()` exposes `core.Interface` for multicast, admin, peer retry, and local
diagnostic state. Root `Obj` does not embed that interface.

Root errors:

| Error | Meaning |
| --- | --- |
| `ErrPeersConflict` | Static and managed peers were both configured |
| `ErrPeerManagerNotEnabled` | Managed-peer method called without a manager |
| `ErrClosed` | Root is closing or closed |
| `ErrCloseTimedOut` | Close budget expired; teardown continues best-effort |
| `ErrInvalidCloseTimeout` | Negative root close timeout |
| `ErrInvalidNodeInfo` | Caller NodeInfo could not be cloned safely |
| `ErrInvalidSigils` | Sigil assembly or parser configuration failed |

Use `errors.Is`; construction rollback and shutdown can join errors.

## SOCKS configuration

`node.EnableSOCKS(ratatoskr.SOCKSConfigObj{...})` is the normal application
entry point.

| Field | Zero value | Negative or non-positive behavior |
| --- | --- | --- |
| `Addr` | Required | TCP address or platform-appropriate Unix path |
| `Nameserver` | Disabled | Empty still permits IP literals and canonical `.pk.ygg` names |
| `Verbose` | False | Per-connection logging |
| `MaxConnections` | 256 | Negative is unlimited |
| `HandshakeTimeout` | 10 s | Negative disables timeout |
| `DialTimeout` | 10 s | Negative disables timeout |
| `TunnelIdleTimeout` | 5 m | Negative disables timeout |
| `MaxAssociateTargetsPerSession` | 128 | Negative is unlimited |
| `MaxAssociateTargetsPerPrincipal` | Unlimited | Non-positive is unlimited; set positive when isolation is needed |
| `MaxAssociateQueuedPacketsPerTarget` | 64 | Negative is unlimited |
| `MaxAssociateQueuedBytesPerTarget` | 64 KiB | Negative is unlimited |
| `NameserverLookupTimeout` | 10 s | Negative disables resolver deadline |
| `NameserverCacheTTL` | 30 s | Negative disables positive caching |
| `NameserverCacheMaxEntries` | 4096 | Negative disables positive caching |
| `Credentials` | None | Optional username/password validator |

Server-wide UDP ASSOCIATE targets default to 1024. New targets are resolved and
dialed through a bounded worker pool. Snapshot counters expose pending,
rejected, and dropped work.

## Root snapshot

`SnapshotObj` is a point-in-time, JSON-tagged copy:

- `Address`, `Subnet`, `PublicKey`, `MTU`;
- `Peers []PeerSnapshotObj` with direction, key, latency, cost, traffic,
  uptime, and last error;
- `ActivePeers []string` when the peer manager exists;
- `SOCKS SOCKSSnapshotObj` with listener state and load counters;
- `CloseTimedOut bool`.

SOCKS counters include `ActiveConnections`, `ActiveAssociateTargets`,
`PendingAssociateTargets`, `RejectedAssociateTargets`, and
`DroppedAssociatePackets`.

## Peer manager

`peermgr.ConfigObj`:

| Field | v1.1.0 behavior |
| --- | --- |
| `Peers` | Candidate URIs; at least one valid candidate is required |
| `MaxPerProto` | 0 or 1 selects one best per scheme; N selects top N; negative invalid |
| `Passive` | Keeps all configured peers and skips latency selection |
| `ProbeTimeout` | Zero 10 s; negative invalid |
| `RefreshInterval` | Non-positive disables scheduled refresh |
| `BatchSize` | Zero or 1 uses 64; internal cap 256; negative invalid |
| `MinPeers` | Zero disables early recovery; must be below selectable capacity |
| `HealthInterval` | Zero 10 s; negative disables health recovery |
| `MinPeersConfirmations` | Zero 3; negative invalid |
| `ReprobeInterval` | Zero 30 m; negative disables holdoff |
| `NoReachablePeers` | Best-effort non-blocking send; use a capacity-one channel |

Probe failures use exponential backoff capped at 10 minutes. `Optimize` is
blocking and serialized. `Close` removes manager-owned peers.

## Resolver and ninfo

Resolver defaults:

- lookup timeout 10 s;
- positive cache TTL 30 s;
- positive cache cap 4096 entries;
- bounded negative cache, at most 3 s;
- 256 concurrent distinct lookups, then `resolver.ErrLookupBusy`.

Ninfo defaults:

- `MaxAskTime` 30 s;
- `AskRetryPause` 500 ms;
- `LookupInterval` 100 ms;
- `MaxLookupTime` 30 s;
- 64 concurrent distinct asks, then `ninfo.ErrAskBusy`;
- 64 concurrent distinct address resolutions, then `ninfo.ErrResolveBusy`;
- remote NodeInfo response cap 16 KiB.

Concurrent requests for the same key or address share one flight. Caller
cancellation stops that caller's wait without invalidating shared work.
`Ask` and `AskAddr` can return a useful result with an error; inspect a non-nil
result before discarding it.

## Built-in sigil bounds

Every sigil name, including a custom one, must match
`^[a-z0-9._-]{3,32}$`. The v1.1.0 built-ins apply these exact bounds:

| Sigil | Exact v1.1.0 constraints |
| --- | --- |
| `info` | Required name `^[a-z0-9._-]{4,64}$`; required type `^[a-z0-9.-]{2,32}$`; optional location and description are 2–514 printable runes with no leading or trailing space; 0–8 contact groups with 1–8 values each; group `^[a-z0-9.-]{2,32}$`; value 3–258 printable runes with no leading or trailing space |
| `services` | 1–256 entries; name `^[a-z0-9_-]{2,32}$`; port 1–65535 |
| `public` | 1–8 groups; group `^[a-z0-9]{2,16}$`; 1–16 URIs per group; URI `^[a-zA-Z0-9+._/:@\[\]-]{8,256}$` |
| `inet` | 1–32 unique addresses; address `^[a-zA-Z0-9._:/-]{4,256}$` |

The ASCII regular expressions bound bytes. The `info` free-text limits count
printable runes. These are schema checks, not DNS, IP, URI-reachability, or
service-protocol validation.

## Forwarding

```go
func forward.New(cfg forward.ConfigObj) (*forward.Obj, error)
```

Mappings are immutable and bound transactionally:

- `LocalTCP`: local listener to Yggdrasil destination;
- `RemoteTCP`: Yggdrasil listener to local destination;
- `LocalUDP`: local packet listener to Yggdrasil destination;
- `RemoteUDP`: Yggdrasil packet listener to local destination.

| Field | v1.1.0 behavior |
| --- | --- |
| `UDPTimeout` | Required positive when any UDP mapping exists |
| `DialTimeout` | Zero 10 s; negative disables |
| `TCPIdleTimeout` | Zero 5 m; negative disables |
| `MaxTCPConnections` | Zero unlimited; negative invalid; object-wide |
| `MaxUDPSessions` | Zero unlimited; negative invalid; object-wide |
| `UDPMaxPacketSize` | Zero uses node MTU; negative permits max datagram |
| `UDPWriteTimeout` | Zero 5 s; negative disables |

`forward.Obj.Close()` is idempotent and waits for owned work.
`forward.Obj.Snapshot()` returns `ActiveTCP`, `ActiveUDP`, `SessionUDPDrops`,
`ReverseUDPDrops`, and `TerminalErrors`.

## Probe

Construct one reusable object:

```go
pr, err := probe.New(probe.ConfigObj{Source: node.Core()})
```

| Field | v1.1.0 behavior |
| --- | --- |
| `MaxTotalNodes` | Zero 4096; negative invalid |
| `PollInterval` | Zero 200 ms; negative invalid |
| `LookupRetryEvery` | Zero 1 s; negative invalid |
| `MaxDuration` | Zero 5 m; negative removes probe-imposed cap |
| `RemoteTimeout` | Zero 30 s; negative removes probe-imposed timeout |

Fixed bounds include 1024 peers accepted from one remote response, 256 remote
flights, and a 1 MiB remote-response cap. Tree worker concurrency defaults to
16 and is clamped to 256.

`NodeObj` fields in v1.1.0:

```go
type NodeObj struct {
    Key         ed25519.PublicKey
    Parent      ed25519.PublicKey
    Sequence    uint64
    Depth       int
    RTT         time.Duration
    Unreachable bool
    Children    []*NodeObj
}
```

`TreeResultObj` contains `Root`, `Total`, `Truncated`, and `Limit`.
`TraceResultObj` contains `TreePath` and `Hops`; either can be partial.
`probe.ErrProbeBusy` reports admission pressure and can accompany partial data.

Local methods such as `Peers`, `Sessions`, `SpanningTree`, `Paths`, `Path`, and
`Hops` read core state. `Tree` performs bounded remote discovery. `Trace`
issues path lookups and can make remote calls to enrich RTT.

## Sigils

Built-in constructors and primary limits:

| Package | Data | Primary bound |
| --- | --- | --- |
| `info` | name, type, location, contacts, description | 8 contact groups and 8 entries per group |
| `services` | named ports | 256 entries |
| `public` | grouped public peer URIs | 8 groups and 16 URIs per group |
| `inet` | Internet addresses | 32 unique values |

All NodeInfo data shares the 16 KiB protocol budget. Custom implementations of
`sigils.Interface` must bound foreign input, clone owned data, avoid mutating
caller maps or slices, and never panic on malformed JSON-shaped values.

## Core-only mode

`core.New(core.ConfigObj{Config: cfg, Logger: logger})` returns a standalone
`*core.Obj`. It is useful when raw dial/listen is the whole requirement. Root
`CloseTimeout` does not apply to a core-only composition; the application owns
the lifecycle of every component it creates.
