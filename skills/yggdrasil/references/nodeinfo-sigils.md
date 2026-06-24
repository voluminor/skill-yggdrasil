# NodeInfo And Sigils

## Contents

- NodeInfo model
- Querying remote NodeInfo
- Parsing results
- Publishing sigils from the root package
- Built-in sigils
- Typed access and custom sigils
- Low-level ninfo usage
- Common mistakes

## NodeInfo Model

Yggdrasil NodeInfo is a JSON object associated with a node. Ratatoskr adds a
structured layer called sigils:

- each sigil owns one or more top-level NodeInfo keys;
- `sigil_core` assembles local NodeInfo;
- `ninfo` queries and parses remote NodeInfo;
- built-in parsers are registered through generated target maps.

Yggdrasil NodeInfo has a 16 KB total size limit. Treat every field as
expensive and bounded.

## Querying Remote NodeInfo

For normal applications, use the root object:

```go
ctx, cancel := context.WithTimeout(parent, 15*time.Second)
defer cancel()

res, err := node.AskAddr(ctx, "200:abcd::1")
if err != nil {
    return fmt.Errorf("ask nodeinfo: %w", err)
}

fmt.Println("rtt:", res.RTT)
fmt.Println("nodeinfo:", res.Node.String())
if res.Software != nil {
    fmt.Println("software:", res.Software.Name, res.Software.Version)
}
```

`AskAddr` accepts:

| Format | Meaning |
| --- | --- |
| `<64hex>.pk.ygg` | Public key encoded in a `.pk.ygg` name. |
| Raw 64-character hex | Public key directly. |
| `[ipv6]:port` | IPv6 extracted and resolved through Yggdrasil lookup. |
| Bare IPv6 | IPv6 resolved through Yggdrasil lookup. |

Use `Ask(ctx, key)` when the public key is already available.

Give the context a generous deadline (10–15s). `Ask`/`AskAddr` retry internally
until the context expires, which matters on a freshly started node still finding
the route — a too-short timeout can look like an unreachable peer.

## Parsing Results

`AskResultObj` contains:

```go
type AskResultObj struct {
    RTT      time.Duration
    Node     *ninfo.ParsedObj
    Software *ninfo.SoftwareObj
}
```

`Software` is nil when NodeInfo privacy hides build fields. Otherwise it is
extracted from `buildname`, `buildversion`, `buildplatform`, and `buildarch`,
and those keys are removed from `Node.Extra`.

`ParsedObj` contains:

```go
type ParsedObj struct {
    Version string
    Sigils  map[string]sigils.Interface
    Extra   map[string]any
}
```

Use `NodeInfo()` to reassemble the parsed content and `String()` for JSON.

`Sigils` holds recognized sigils as `sigils.Interface`; type-assert each to its
concrete `*Obj` for native Go types (e.g. `res.Node.Sigils["services"].(*services.Obj).Services()`).
Unrecognized keys — including custom sigils your node has no parser for — remain
in `Extra` as raw values. See [custom-sigils.md](custom-sigils.md) for typed
access and for parsing custom sigils without touching `map[string]any`.

## Publishing Sigils From The Root Package

Pass sigils at node creation:

```go
infoSigil, err := info.New(info.ConfigObj{
    Name:        "node.example",
    Type:        "service",
    Description: "public API node",
})
if err != nil {
    return err
}

svcSigil, err := services.New(map[string]uint16{
    "http": 8080,
})
if err != nil {
    return err
}

node, err := ratatoskr.New(ratatoskr.ConfigObj{
    Ctx:    ctx,
    Config: cfg,
    Sigils: []sigils.Interface{infoSigil, svcSigil},
})
```

`Config.NodeInfo` is the base; sigil data is added on top. The merge is strict:
a key collision (between two sigils, or with a base key) is **never silently
overwritten** — `SetParams`/`MergeParams` return it as an error. Assembled
through `ratatoskr.New`, that error is logged as a warning and the conflicting
sigil is **skipped** while the node still starts. So resolve collisions in code;
do not ship a node whose sigil was dropped.

## Built-In Sigils

| Package | Key(s) | Constructor | Validation (current source) |
| --- | --- | --- | --- |
| `mod/sigils/info` | `name`,`type`,`location`,`contact`,`description` | `info.New(info.ConfigObj{...})` | `Name` req. `[a-z0-9._-]{4,64}`; `Type` req. `[a-z0-9.-]{2,32}`; `Description` opt. 2–514 chars; `Location` opt. (free text, **not** length-validated in current source); `Contacts` ≤ 8 groups × ≤ 8, group `[a-z0-9.-]{2,32}`, value 3–258 chars. |
| `mod/sigils/services` | `services` | `services.New(map[string]uint16{...})` | name `[a-z0-9_-]{2,32}`; port 1–65535; ≤ 256 entries. |
| `mod/sigils/public` | `public` | `public.New(map[string][]string{...})` | group `[a-z0-9]{2,16}`; URI `[a-zA-Z0-9+._/:@[\]-]{8,256}`; ≤ 8 groups × ≤ 16 URIs. |
| `mod/sigils/inet` | `inet` | `inet.New([]string{...})` | address `[a-zA-Z0-9._:/-]{4,256}`; ≤ 32; no duplicates. |

`info.ConfigObj` fields:

```go
type ConfigObj struct {
    Name        string              // required, [a-z0-9._-]{4,64}
    Type        string              // required, [a-z0-9.-]{2,32}
    Location    string              // optional free text — NOT validated in current source; keep it small
    Contacts    map[string][]string // optional, ≤8 groups × ≤8; group [a-z0-9.-]{2,32}, value 3–258
    Description string              // optional, validated 2–514 chars
}
```

Constructors return an error on violation — handle it **before** `ratatoskr.New`.
These limits are read from the current source; re-check with `go doc` if the
module version changes.

Read built-in sigils back with typed accessors after a type assertion:
`info.Obj.Info()`, `public.Obj.Peers()`, `inet.Obj.Addrs()`,
`services.Obj.Services()`.

## Typed Access And Custom Sigils

To read sigils as native Go types and to define/wire your own sigils — the
`sigils.Interface` contract, the four-file package layout, a complete
copy-paste template, and the publish / read-remote / contribute wiring paths —
see [custom-sigils.md](custom-sigils.md). The goal of that file is to keep
application code off raw `map[string]any` entirely: build from typed structs,
read back through typed accessors.

## Low-Level ninfo Usage

Direct `ninfo.New` is for tools that already have a `*yggcore.Core`:

```go
coreNode := node.Interface.(*core.Obj)
ni, err := ninfo.New(coreNode.UnsafeCore(), logger)
```

It captures the `getNodeInfo` admin handler through `core.SetAdmin`. Most
applications should not do this because `ratatoskr.New` already creates ninfo
and exposes `node.Ask` and `node.AskAddr`.

`ninfo.Parse(nodeInfo)` (no extra sigils) parses an arbitrary NodeInfo map
without a network request, recognizing the built-in sigils. To decode an
**app-defined** sigil, read the leftover keys from the result's `Extra` with
your sigil's package-level `Parse`. Do not pass your sigil to `ninfo.Parse` or
`ninfo.AddSigil` for dynamic parsing: sigils are assembled strictly in code, and
that path re-applies the strict merge and conflicts by design (see
[custom-sigils.md](custom-sigils.md)).

## Common Mistakes

| Mistake | Correction |
| --- | --- |
| Publishing unbounded arrays or maps | Define strict limits for every collection. |
| Parsing JSON as native Go types | Remember JSON numbers become `float64`; arrays become `[]any`. |
| Mutating caller maps in `SetParams` | Use `sigils.MergeParams` or copy explicitly. |
| Creating `ninfo.New` in normal app code | Use `node.Ask` and `node.AskAddr`. |
| Expecting an app-defined sigil to auto-parse via `ninfo.AddSigil`/`Parse(sg)` | Sigils are assembled in code; built-ins auto-parse. Decode your own keys from `res.Node.Extra` with your `Parse`. |
