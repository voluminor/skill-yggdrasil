# NodeInfo, sigils, and topology diagnostics

Discovery ranges from free local state reads to expensive remote topology
walks. Use the least costly source that answers the question.

## Cost order

| Need | Preferred source | Overlay cost |
| --- | --- | --- |
| Node health and direct peers | `node.Snapshot()` | none beyond normal node operation |
| Local tree and paths | `probe.SpanningTree`, `Paths`, `Path`, `Hops` | local state read |
| One remote node's metadata | `node.Ask` or `AskAddr` | one bounded, coalesced remote workflow |
| Route trace with RTT enrichment | `probe.Trace` | path lookups and possible remote calls |
| Nearby topology | bounded `probe.Tree` | one remote peer query per explored branch node |

Do not use a topology tree as a routine health endpoint.

## Local snapshots

```go
snap := node.Snapshot()
fmt.Println(snap.Address, snap.MTU, len(snap.Peers))
```

Snapshots contain node identity, peer metrics, selected managed peers, SOCKS
state, overload counters, and prior close-timeout state. They are point-in-time
copies, not atomic transactions across the entire network.

## Query remote NodeInfo

```go
ctx, cancel := context.WithTimeout(parent, 15*time.Second)
defer cancel()

res, err := node.AskAddr(ctx, target)
if res != nil {
    consume(res)
}
if err != nil {
    return err
}
```

`AskAddr` accepts a raw public-key hex string, canonical `.pk.ygg` name, bare
Yggdrasil IPv6 address, or bracketed IPv6 host-port form. Use `Ask` when the
Ed25519 public key is already available.

Concurrent work for the same node coalesces. New distinct work is bounded;
`ErrAskBusy` and `ErrResolveBusy` are load-shedding signals. Retain a non-nil
result, then defer or skip the request instead of retrying immediately.

## NodeInfo policy

The complete NodeInfo payload shares a 16 KiB budget. Keep it compact and
slow-changing. Good content includes identity, service discovery, public peer
endpoints, and stable Internet identifiers. Metrics, logs, presence heartbeats,
and fast-changing counters belong elsewhere.

Use built-in sigils first:

| Sigil | Purpose |
| --- | --- |
| `info` | node name, type, location, contacts, description |
| `services` | named Yggdrasil service ports |
| `public` | grouped public peering URIs |
| `inet` | Internet addresses associated with the node |

Example:

```go
identity, err := info.New(info.ConfigObj{
    Name: "edge.example.net",
    Type: "service",
})
if err != nil {
    return err
}

ports, err := services.New(map[string]uint16{"http": 8080})
if err != nil {
    return err
}

node, err := ratatoskr.New(ratatoskr.ConfigObj{
    Ctx:    ctx,
    Config: cfg,
    Sigils: []sigils.Interface{identity, ports},
})
```

Sigil assembly is strict. Invalid entries, duplicate names, or owned-key
conflicts make root construction fail with `ErrInvalidSigils`.

## Read parsed sigils

Known remote sigils are typed:

```go
if sg, ok := res.Node.Sigils[services.Name()].(*services.Obj); ok {
    servicePorts := sg.Services()
    _ = servicePorts
}
```

`ParsedObj.Extra` contains unclaimed fields. Build information is extracted
separately into `AskResultObj.Software` when present.

## Custom sigils

Create a custom sigil only when no built-in schema owns the data. A v1 sigil
implements:

```go
type Interface interface {
    GetName() string
    GetParams() []string
    ParseParams(map[string]any) map[string]any
    Match(map[string]any) bool
    Params() map[string]any
    Clone() Interface
}
```

Implementation requirements:

- bound every string and collection before allocation grows;
- accept JSON-decoded shapes such as `float64`, `[]any`, and
  `map[string]any`;
- use the same value rules for local construction and foreign parsing;
- never mutate caller maps or slices;
- deep-copy mutable state from `Params`, accessors, and `Clone`;
- return false or an error for malformed input, never panic;
- do not access a mutable sigil object concurrently without external
  synchronization.

Passing a custom instance through root `ConfigObj.Sigils` publishes it locally
and registers a clone as a remote parser. For parse-only use, add a prototype to
`ConfigObj.NodeInfo.Sigils`. Offline parsing can use `ninfo.Parse`.

### Compact custom sigil

This complete example owns one bounded, slow-changing `organization` string.
Strings are immutable, while every returned map and key slice is independent.

```go
package discovery

import (
    "fmt"
    "regexp"

    "github.com/voluminor/ratatoskr/mod/sigils"
)

const organizationKey = "organization"

var organizationPattern = regexp.MustCompile(
    `^[a-z0-9][a-z0-9 ._-]{1,63}$`,
)

type OrganizationSigil struct {
    value string
}

func validOrganization(value string) bool {
    return len(value) >= 2 && len(value) <= 64 &&
        organizationPattern.MatchString(value)
}

func NewOrganizationSigil(value string) (*OrganizationSigil, error) {
    if !validOrganization(value) {
        return nil, fmt.Errorf("invalid organization")
    }
    return &OrganizationSigil{value: value}, nil
}

func organizationFrom(nodeInfo map[string]any) (string, bool) {
    value, ok := nodeInfo[organizationKey].(string)
    return value, ok && validOrganization(value)
}

func (o *OrganizationSigil) GetName() string {
    return organizationKey
}

func (o *OrganizationSigil) GetParams() []string {
    return []string{organizationKey}
}

func (o *OrganizationSigil) ParseParams(nodeInfo map[string]any) map[string]any {
    parsed := make(map[string]any, 1)
    if value, exists := nodeInfo[organizationKey]; exists {
        parsed[organizationKey] = value
    }
    if value, ok := organizationFrom(parsed); ok {
        o.value = value
    }
    return parsed
}

func (o *OrganizationSigil) Match(nodeInfo map[string]any) bool {
    _, ok := organizationFrom(nodeInfo)
    return ok
}

func (o *OrganizationSigil) Params() map[string]any {
    return map[string]any{organizationKey: o.value}
}

func (o *OrganizationSigil) Clone() sigils.Interface {
    return &OrganizationSigil{value: o.value}
}

func (o *OrganizationSigil) Organization() string {
    return o.value
}
```

Publish it with the built-ins:

```go
organization, err := NewOrganizationSigil("example foundation")
if err != nil {
    return err
}

node, err := ratatoskr.New(ratatoskr.ConfigObj{
    Sigils: []sigils.Interface{identity, ports, organization},
})
```

The same registered type is available after remote parsing:

```go
if organization, ok := res.Node.Sigils[organizationKey].(*OrganizationSigil); ok {
    fmt.Println(organization.Organization())
}
```

Do not add live CPU, queue depth, or similar telemetry to this sigil. Publish
fast-changing measurements through an application metrics endpoint so NodeInfo
stays compact and does not churn across the overlay.

## Topology probe

Create one probe per node and reuse it:

```go
pr, err := probe.New(probe.ConfigObj{
    Source:        node.Core(),
    MaxTotalNodes: 256,
})
if err != nil {
    return err
}
defer pr.Close()
```

The sample node limit suits a bounded diagnostic, not every deployment. The
application owns `probe.Obj`; root `Close` does not close it.

### Local diagnostics

Use these first:

```go
self := pr.Self()
peers := pr.Peers()
tree := pr.SpanningTree()
paths := pr.Paths()
```

`Path(key)` derives a root-to-target path from local spanning-tree state.
`Hops(key)` reads a pathfinder route after a lookup has populated it.

### Route trace

```go
ctx, cancel := context.WithTimeout(parent, 30*time.Second)
defer cancel()

res, err := pr.Trace(ctx, key)
if res != nil {
    useRoute(res.TreePath, res.Hops)
}
```

Tree path and hops are independent partial results. RTT enrichment can hit the
remote-flight admission limit; preserve the route when the error matches
`probe.ErrProbeBusy`.

### Bounded topology tree

```go
ctx, cancel := context.WithTimeout(parent, time.Minute)
defer cancel()

res, err := pr.Tree(ctx, 2, 8)
if res != nil {
    fmt.Println(res.Total, res.Truncated)
}
```

`maxDepth` must be positive. Set `MaxTotalNodes` on the probe for the question
being asked. Treat `Truncated` as part of the result contract: a truncated tree
is a bounded sample, not the whole network.

In v1.1.0, `probe.NodeObj.Parent` is an `ed25519.PublicKey`, not a pointer to
the parent node. `Children` holds the recursive tree.

## Mesh-friendly rules

- Run trees on demand, not on a short timer.
- Prefer depth 1 or 2 for neighborhood questions.
- Do not fan out tree scans from several nodes.
- Cache a result only with an explicit size and freshness policy.
- Reuse one probe so same-key remote calls can coalesce.
- On `ErrProbeBusy`, keep partial data and retry later only if the user still
  needs fresher data.
- On remote timeout, do not pile another call onto work that may still own its
  admission slot.

## Admin endpoint

`Core().EnableAdmin` exposes upstream administrative handlers without a strong
security boundary. It is not needed for snapshots, NodeInfo, or probe. If an
operator explicitly needs it, bind it to a protected platform-appropriate
local endpoint and treat access as full process control.
