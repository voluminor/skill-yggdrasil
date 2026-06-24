# Framework Integrations

Templates for connecting an application to the Yggdrasil network as a client and
as a server, and how popular Go networking frameworks fit (or do not fit) the
userspace stack.

## Contents

- The one rule
- Compatibility at a glance
- net/http (standard library)
- fasthttp
- gRPC
- gnet and other fd/epoll frameworks — bridge via forwarding
- Raw conn frameworks (websocket, net/rpc)
- Constrained-mesh transfer pattern
- Address checklist

## The One Rule

Ratatoskr hands you userspace `net.Conn`, `net.Listener`, `net.PacketConn`, and
`node.DialContext`. These are **gVisor netstack objects — not backed by OS file
descriptors**. That single fact decides every integration:

- A library that accepts a **`net.Listener`** (server) or a **custom dial
  function** (client) works **directly** over Yggdrasil. This covers the vast
  majority of Go networking code.
- A library that opens and polls its **own OS sockets** via epoll/kqueue on file
  descriptors (gnet, evio) **cannot** consume Ratatoskr's listeners or conns.
  Bridge it through `mod/forward` over a localhost socket instead.

`node` here is `*ratatoskr.Obj`; `node.DialContext`, `node.Listen`, and
`node.ListenPacket` come from the embedded `core.Interface`.

## Compatibility At A Glance

| Framework | Server side | Client side |
| --- | --- | --- |
| `net/http` | `srv.Serve(node.Listen(...))` | `Transport.DialContext = node.DialContext` |
| `fasthttp` | `fasthttp.Serve(node.Listen(...), h)` | `Client.Dial` → `node.DialContext` |
| `google.golang.org/grpc` | `grpcSrv.Serve(node.Listen(...))` | `grpc.WithContextDialer(node.DialContext)` |
| `coder`/`gorilla` websocket | accept on `node.Listen(...)`, upgrade | custom `NetDial`/`DialContext` |
| `net/rpc` | `rpc.Accept(node.Listen(...))` | dial via a `node.DialContext` conn |
| **gnet, evio** (fd/epoll) | no direct handoff → bridge via `mod/forward` (works) | no direct handoff → bridge via `mod/forward` (works) |

## net/http (Standard Library)

Client — route every request through Yggdrasil with one transport field:

```go
client := &http.Client{
    Transport: &http.Transport{DialContext: node.DialContext},
    Timeout:   15 * time.Second,
}
resp, err := client.Get("http://[200:abcd::1]:8080/api") // brackets required
```

Server — serve on a Yggdrasil listener instead of an OS port:

```go
ln, err := node.Listen("tcp", fmt.Sprintf("[%s]:%d", node.Address(), 8080))
if err != nil {
    return err
}
srv := &http.Server{Handler: mux, ReadHeaderTimeout: 10 * time.Second}
go func() { _ = srv.Serve(ln) }()
// on shutdown: srv.Shutdown(ctx)
```

Full lifecycle and TCP/UDP variants are in
[networking.md](networking.md).

## fasthttp

fasthttp is fully compatible: its server takes a `net.Listener` and its client
takes a `Dial` function.

Server:

```go
ln, err := node.Listen("tcp", fmt.Sprintf("[%s]:%d", node.Address(), 8080))
if err != nil {
    return err
}
h := func(ctx *fasthttp.RequestCtx) {
    ctx.SetStatusCode(fasthttp.StatusOK)
    ctx.WriteString("ok over yggdrasil")
}
go func() { _ = fasthttp.Serve(ln, h) }()
```

Client — wrap `node.DialContext` in fasthttp's `DialFunc` (it receives the
already-bracketed `host:port`):

```go
c := &fasthttp.Client{
    Dial: func(addr string) (net.Conn, error) {
        return node.DialContext(context.Background(), "tcp", addr)
    },
}
status, body, err := c.Get(nil, "http://[200:abcd::1]:8080/api")
```

For per-request cancellation use `fasthttp.HostClient` with `DoDeadline`, or
carry a context into the dialer via a closure.

## gRPC

gRPC accepts a `net.Listener` on the server and a context dialer on the client,
so it rides the userspace stack directly.

Server:

```go
ln, err := node.Listen("tcp", fmt.Sprintf("[%s]:%d", node.Address(), 9000))
if err != nil {
    return err
}
s := grpc.NewServer()
// register services …
go func() { _ = s.Serve(ln) }()
```

Client — use a context dialer and the `passthrough` scheme so gRPC does not try
DNS on a Yggdrasil literal:

```go
conn, err := grpc.NewClient(
    "passthrough:///[200:abcd::1]:9000",
    grpc.WithContextDialer(func(ctx context.Context, addr string) (net.Conn, error) {
        return node.DialContext(ctx, "tcp", addr)
    }),
    grpc.WithTransportCredentials(insecure.NewCredentials()), // or real TLS
)
```

## gnet And Other fd/epoll Frameworks — Bridge Via Forwarding

gnet (and evio) run their own event loop on raw OS file descriptors and bind
addresses themselves via syscalls. They cannot accept a gVisor `net.Listener` or
dial through `node.DialContext`, so the **direct** listener handoff is
impossible. That is the only thing that does not work — the **forward bridge
works reliably**: `mod/forward` terminates the Yggdrasil side on the gVisor
stack and shuttles bytes to/from a local OS socket that gnet owns. Wire it over
loopback.

(`AddRemoteTCP` always listens on `node.Address()` and uses only `Listen.Port`;
set `Listen.IP` to `node.Address()` for clarity.)

Expose a gnet server (bound to `127.0.0.1:9000`) inside Yggdrasil — forward
listens on the Yggdrasil side and dials the local gnet:

```go
// gnet listens on a real OS socket:
go gnet.Run(handler, "tcp://127.0.0.1:9000")

// bridge Yggdrasil:port -> local gnet:
mgr := forward.New(logger, 120*time.Second)
mgr.AddRemoteTCP(forward.TCPMappingObj{
    Listen: &net.TCPAddr{IP: node.Address(), Port: 9000},          // Yggdrasil side
    Mapped: &net.TCPAddr{IP: net.IPv4(127, 0, 0, 1), Port: 9000},  // local gnet
})
mgr.Start(ctx, node)
defer func() { cancel(); mgr.Wait() }()
```

Let a gnet client reach a Yggdrasil service — forward listens locally and dials
Yggdrasil; point gnet at `127.0.0.1`:

```go
mgr.AddLocalTCP(forward.TCPMappingObj{
    Listen: &net.TCPAddr{IP: net.IPv4(127, 0, 0, 1), Port: 18080},     // local
    Mapped: &net.TCPAddr{IP: net.ParseIP("200:abcd::1"), Port: 80},    // Yggdrasil
})
```

The forwarding manager API (directions, timeouts, `Wait`) is documented in
[peers-socks-forwarding.md](peers-socks-forwarding.md).

## Raw Conn Frameworks (websocket, net/rpc)

Anything built on a plain `net.Conn`/`net.Listener` works without a bridge:

- **websocket** (`coder/websocket`, `gorilla/websocket`): on the server, accept
  on `node.Listen(...)` and upgrade the request as usual; on the client, set the
  library's `NetDialContext`/`NetDial` option to `node.DialContext`.
- **net/rpc**: `rpc.Accept(node.Listen(...))` on the server; on the client, dial
  a `net.Conn` with `node.DialContext` and pass it to `rpc.NewClient`.

## Constrained-Mesh Transfer Pattern

Yggdrasil links are low-bandwidth and lossy (see the mesh-design section in
`SKILL.md`). Do not stream a large body in one shot. Fetch it in small,
independently retried `Range` chunks so a dropped link costs one chunk, and use a
bounded transport.

Bounded client (cap concurrency; per-request deadlines, not a blanket timeout):

```go
client := &http.Client{
    Transport: &http.Transport{
        DialContext:         node.DialContext,
        MaxConnsPerHost:     4,                // weak link: few connections
        MaxIdleConnsPerHost: 2,
        IdleConnTimeout:     30 * time.Second,
    },
}
```

Chunked, retried, resumable download:

```go
const chunkSize = 256 * 1024 // small: a lost link re-fetches little

// download fetches url into w (e.g. *os.File) in Range chunks. Skip offsets you
// already have to resume after a crash.
func download(ctx context.Context, client *http.Client, url string, size int64, w io.WriterAt) error {
    for off := int64(0); off < size; off += chunkSize {
        end := off + chunkSize - 1
        if end >= size {
            end = size - 1
        }
        if err := fetchRange(ctx, client, url, off, end, w); err != nil {
            return fmt.Errorf("chunk %d-%d: %w", off, end, err) // resume from off later
        }
    }
    return nil
}

func fetchRange(ctx context.Context, client *http.Client, url string, start, end int64, w io.WriterAt) error {
    var lastErr error
    for attempt := 0; attempt < 5; attempt++ {
        if attempt > 0 { // backoff + jitter, abort if caller is done
            wait := time.Duration(1<<attempt)*200*time.Millisecond + time.Duration(rand.Int63n(150))*time.Millisecond
            select {
            case <-ctx.Done():
                return ctx.Err()
            case <-time.After(wait):
            }
        }
        rctx, cancel := context.WithTimeout(ctx, 20*time.Second) // per-attempt deadline
        req, _ := http.NewRequestWithContext(rctx, http.MethodGet, url, nil)
        req.Header.Set("Range", fmt.Sprintf("bytes=%d-%d", start, end))
        resp, err := client.Do(req)
        if err != nil {
            cancel(); lastErr = err; continue
        }
        body, err := io.ReadAll(resp.Body)
        resp.Body.Close(); cancel()
        if err != nil {
            lastErr = err; continue
        }
        if resp.StatusCode != http.StatusPartialContent || int64(len(body)) != end-start+1 {
            lastErr = fmt.Errorf("bad chunk: status=%s len=%d", resp.Status, len(body)) // verify size
            continue
        }
        if _, err := w.WriteAt(body, start); err != nil {
            return err // local write failure: do not retry blindly
        }
        return nil
    }
    return lastErr
}
```

Get `size` from a `HEAD`, or from the `Content-Range` of a `bytes=0-0` probe if
the server has no `HEAD`. Add a per-chunk checksum if the server exposes one.
Persist completed offsets (or use a sparse file plus a progress record) to resume
across restarts. The same shape — small units, verify, retry, resume — applies to
uploads (chunked `PUT`/`POST`) and to any long transfer over the mesh.

## Address Checklist

- Bracket IPv6 in every URL and host-port string: `[200:abcd::1]:8080`.
- Build listen addresses from `node.Address()`:
  `fmt.Sprintf("[%s]:%d", node.Address(), port)`.
- Network strings are `tcp`/`tcp6`/`udp`/`udp6`.
- For UDP-based frameworks, use `node.ListenPacket` / `node.DialContext` with
  `udp`; size buffers from `node.MTU()`.
