# Custom Sigils

Sigils are typed blocks of Yggdrasil NodeInfo. They exist so that application
code never has to read or write the raw `map[string]any` NodeInfo by hand: you
build NodeInfo from Go structs and read it back as Go structs, with validation
and JSON-type handling done once inside the sigil.

## Contents

- Why sigils instead of raw maps
- Typed access to existing sigils
- Anatomy of a sigil
- The rules
- Template: a complete custom sigil
- Wiring a custom sigil (publish, read remote, contribute)
- Helpers and pitfalls

## Why Sigils Instead Of Raw Maps

NodeInfo is a single JSON object, max 16 KB, shared by every sigil plus the
Ratatoskr metadata key. Hand-writing `map[string]any` invites three bugs the
sigil layer removes for you:

- **No validation.** A sigil validates at construction (`New`) with compiled
  regexps and explicit limits.
- **JSON type traps.** Foreign NodeInfo comes from `encoding/json`: arrays are
  `[]any`, objects are `map[string]any`, every number is `float64`. Sigils do
  the type assertions in one place.
- **Key collisions.** Each sigil owns specific top-level keys; assembly detects
  conflicts instead of silently overwriting.

So: to **publish**, construct a sigil and let it write the map. To **read**,
parse into a sigil and use typed accessors. Touch `map[string]any` only inside
the sigil implementation.

**Assemble sigils strictly in code, not dynamically at runtime.** The merge is
deliberately strict: a duplicate key, or a sigil key that would overwrite an
existing NodeInfo key, is **never silently merged** — `SetParams`/`MergeParams`
return it as an error. Through `ratatoskr.New` that surfaces as a logged warning
and the conflicting sigil is skipped, so the collision is visible, not silent.
Decide your node's sigils when you construct it and resolve collisions in code;
do not mutate or re-register them on a running node.

## Typed Access To Existing Sigils

A `sigils.Interface` value carries data but exposes it only as
`Params() map[string]any`. To get native Go types, type-assert to the concrete
`*Obj` and use its `access.go` method:

| Sigil | Accessor | Returns |
| --- | --- | --- |
| `info` | `Info()` | `*info.ConfigObj` |
| `public` | `Peers()` | `map[string][]string` |
| `inet` | `Addrs()` | `[]string` |
| `services` | `Services()` | `map[string]uint16` |

```go
import (
    "github.com/voluminor/ratatoskr/mod/sigils"
    "github.com/voluminor/ratatoskr/mod/sigils/services"
)

var sig sigils.Interface // from ninfo, Ask result, etc.

if svc, ok := sig.(*services.Obj); ok {
    for name, port := range svc.Services() { // typed, no map[string]any
        fmt.Printf("%s -> %d\n", name, port)
    }
}
```

## Anatomy Of A Sigil

Each sigil is its own subpackage, `mod/sigils/<name>/`, with up to four files:

| File | Holds |
| --- | --- |
| `values.go` | name constant, owned key constants, regexps, numeric limits |
| `func.go` | package-level `Name()`, `Keys()`, `Match()`, `ParseParams()`, `Parse()` |
| `obj.go` | `ConfigObj`, `Obj`, `New(ConfigObj)`, and the `sigils.Interface` methods |
| `access.go` | optional typed accessors on `*Obj` (not part of `Interface`) |

The split matters: the **package-level** functions work without an instance
(used by the registry and by quick checks), while the **methods** delegate to
them and additionally store parsed data on the object.

## The Rules

These come from the library's own sigil-authoring guide. Follow all of them:

1. **Bound everything.** Max count for every collection, max length for every
   string. No unbounded data — you share 16 KB with every other sigil.
2. **Validate input in `New`** with compiled regexps, not later.
3. **Never mutate input maps.** `SetParams`/`ParseParams` copy; use
   `sigils.MergeParams` to add keys onto a copy.
4. **Handle JSON types in `Match`/`Parse`.** `[]any`, `map[string]any`,
   `float64`. Check integer-ness for ports (`f == float64(int(f))`).
5. **Name must pass `sigils.ValidateName`** — pattern `^[a-z0-9._-]{3,32}$`.
6. **Owned keys must be unique** across all sigils; assembly errors on collision.
7. **`Match` never panics and never errors** — return `false` on any nil, wrong
   type, or empty-where-meaningless. It checks structure and types only, not
   content validity (that is `New`'s job).
8. **`Clone` deep-copies** every mutable field and must be safe to call on a
   freshly created instance (see the pitfall at the end).

## Template: A Complete Custom Sigil

A minimal but correct sigil named `org`, owning keys `org` (string) and `tags`
(`[]string`). Copy and adapt.

`values.go`
```go
package org

import "regexp"

const sigName = "org"

const (
    keyOrg  = "org"
    keyTags = "tags"
)

var sigKeys = []string{keyOrg, keyTags}

const maxTags = 16

var (
    reOrg = regexp.MustCompile(`^[a-z0-9._-]{2,64}$`)
    reTag = regexp.MustCompile(`^[a-z0-9._-]{2,32}$`)
)
```

`func.go`
```go
package org

import "errors"

func Name() string   { return sigName }
func Keys() []string { return sigKeys }

// Match validates structure and JSON types only — never content, never panics.
func Match(nodeInfo map[string]any) bool {
    org, ok := nodeInfo[keyOrg].(string)
    if !ok || org == "" {
        return false
    }
    if raw, present := nodeInfo[keyTags]; present { // tags optional
        arr, ok := raw.([]any)
        if !ok || len(arr) == 0 {
            return false
        }
        for _, item := range arr {
            if _, ok := item.(string); !ok {
                return false
            }
        }
    }
    return true
}

// ParseParams extracts only this sigil's keys, no storage.
func ParseParams(nodeInfo map[string]any) map[string]any {
    out := make(map[string]any)
    if v, ok := nodeInfo[keyOrg].(string); ok {
        out[keyOrg] = v
    }
    if raw, ok := nodeInfo[keyTags].([]any); ok {
        out[keyTags] = raw
    }
    return out
}

// Parse builds a typed *Obj from foreign NodeInfo.
func Parse(nodeInfo map[string]any) (*Obj, error) {
    if !Match(nodeInfo) {
        return nil, errors.New("org: no match")
    }
    o := &Obj{conf: &ConfigObj{}}
    o.ParseParams(nodeInfo)
    return o, nil
}
```

`obj.go`
```go
package org

import (
    "errors"
    "fmt"

    "github.com/voluminor/ratatoskr/mod/sigils"
)

// Compile-time guarantee that *Obj satisfies the contract.
var _ sigils.Interface = (*Obj)(nil)

type ConfigObj struct {
    Org  string   // required, 2–64 chars, [a-z0-9._-]
    Tags []string // optional, max 16, each 2–32 chars
}

func validateConfig(c *ConfigObj) error {
    if !reOrg.MatchString(c.Org) {
        return errors.New("org: invalid or missing org")
    }
    if len(c.Tags) > maxTags {
        return fmt.Errorf("org: too many tags: %d (max %d)", len(c.Tags), maxTags)
    }
    for i, t := range c.Tags {
        if !reTag.MatchString(t) {
            return fmt.Errorf("org: invalid tag[%d]", i)
        }
    }
    return nil
}

type Obj struct{ conf *ConfigObj }

// New constructs and validates the sigil.
func New(conf ConfigObj) (*Obj, error) {
    if err := validateConfig(&conf); err != nil {
        return nil, err
    }
    return &Obj{conf: &conf}, nil
}

func (o *Obj) GetName() string     { return Name() }
func (o *Obj) GetParams() []string { return Keys() }

func (o *Obj) SetParams(nodeInfo map[string]any) (map[string]any, error) {
    return sigils.MergeParams(nodeInfo, o.Params()) // copies, errors on collision
}

func (o *Obj) Params() map[string]any {
    out := make(map[string]any)
    if o.conf == nil {
        return out
    }
    if o.conf.Org != "" {
        out[keyOrg] = o.conf.Org
    }
    if len(o.conf.Tags) > 0 {
        out[keyTags] = o.conf.Tags
    }
    return out
}

func (o *Obj) ParseParams(nodeInfo map[string]any) map[string]any {
    parsed := ParseParams(nodeInfo)
    conf := ConfigObj{}
    if v, ok := parsed[keyOrg].(string); ok {
        conf.Org = v
    }
    if arr, ok := parsed[keyTags].([]any); ok {
        conf.Tags = make([]string, 0, len(arr))
        for _, item := range arr {
            if s, ok := item.(string); ok {
                conf.Tags = append(conf.Tags, s)
            }
        }
    }
    o.conf = &conf
    return parsed
}

func (o *Obj) Match(nodeInfo map[string]any) bool { return Match(nodeInfo) }

func (o *Obj) Clone() sigils.Interface {
    if o.conf == nil { // nil-safe: parsers are cloned from a prototype
        return &Obj{conf: &ConfigObj{}}
    }
    conf := *o.conf
    conf.Tags = append([]string(nil), o.conf.Tags...)
    return &Obj{conf: &conf}
}
```

`access.go`
```go
package org

// Org returns the sigil's typed data without going through map[string]any.
func (o *Obj) Org() *ConfigObj { return o.conf }
```

## Wiring A Custom Sigil

Three things people mean by "connecting" a sigil: publish it, read it back, or
contribute it upstream. Pick by intent.

### 1. Publish it on your own node (most common)

No registration needed. Construct it and pass it through `ConfigObj.Sigils`:

```go
orgSig, err := org.New(org.ConfigObj{Org: "example", Tags: []string{"relay", "eu"}})
if err != nil {
    return err // handle BEFORE starting the node
}

node, err := ratatoskr.New(ratatoskr.ConfigObj{
    Ctx:    ctx,
    Config: cfg,
    Sigils: []sigils.Interface{orgSig},
})
```

Your sigil name is now listed in the node's NodeInfo metadata, so other nodes
running your code can recognize and parse it.

### 2. Read it from a remote node (typed, root API)

`node.AskAddr` / `node.Ask` auto-parse only the **built-in** sigils. An unknown
custom sigil's keys are left untouched in `res.Node.Extra`. Decode them with
your package-level `Parse` — the caller still never sees `map[string]any`:

```go
res, err := node.AskAddr(ctx, addr)
if err != nil {
    return err
}
if orgSig, err := org.Parse(res.Node.Extra); err == nil {
    cfg := orgSig.Org() // *org.ConfigObj
    fmt.Println(cfg.Org, cfg.Tags)
}
```

> **Read your own sigils the "non-system" way — do not register them for
> dynamic parsing.** The built-in ("system") sigils are always parsed
> automatically; an app-defined sigil is yours to own in code and read
> explicitly, as in step 2. Registering an instance for dynamic parsing
> (`ninfo.AddSigil` / `ninfo.Parse(raw, yourSigil)`) is **not** the intended
> path: that route re-applies the strict assembly merge to keys it already
> owns, so it conflicts by design — the same strictness that protects you at
> build time. Decode from `res.Node.Extra` with your package-level `Parse`
> instead. To make a sigil part of the always-parsed system set, contribute it
> into the module (step 3).

### 3. Contribute it to Ratatoskr itself (for everyone, automatic)

Only when adding a sigil to the Ratatoskr module: place the package under
`mod/sigils/<name>/`, export `Name`, `Keys`, `Match`, `ParseParams`, `Parse`,
and run the `_generate/sigils` generator. It discovers the package and writes
`target/sigils.go` so every node parses it automatically. Do **not** hand-edit
`target/sigils.go`. This path does not apply to sigils defined in your own app.

## Helpers And Pitfalls

- `sigils.MergeParams(nodeInfo, params)` — copy `nodeInfo`, add `params`, error
  on key conflict. Use it in `SetParams`; never write to the input directly.
- `sigils.ValidateName(name)` — the `^[a-z0-9._-]{3,32}$` check `GetName` must
  satisfy.
- **Keep `Clone` and `Params` nil-safe** and deep-copying, as in the template.
  The contribute path builds a fresh `*Obj` before populating it, and defensive
  nil handling keeps these methods from dereferencing a nil config.
- **Test like the built-ins:** constructor boundaries (min/max length and count,
  every invalid case), `Match` against wrong JSON types and nils, a parse
  round-trip, `SetParams` conflict + no-mutation, and `Clone` independence.
