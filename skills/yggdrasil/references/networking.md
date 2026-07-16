# Networking and framework integration

Ratatoskr exposes userspace implementations of `net.Conn`, `net.Listener`, and
`net.PacketConn`. They behave through standard Go interfaces but are not OS
file descriptors.

## Integration rule

| Consumer | Integration |
| --- | --- |
| Accepts `net.Listener` | serve directly on `node.Listen` |
| Accepts a context dial function | use `node.DialContext` |
| Opens and polls its own OS file descriptors | bridge through `mod/forward` and a host-local socket |

This distinction covers `net/http`, gRPC, fasthttp, websocket libraries,
`net/rpc`, and fd-oriented loops such as gnet or evio.

## Addresses

Yggdrasil addresses are IPv6. Bracket them in host-port strings and URLs:

```go
listenAddr := fmt.Sprintf("[%s]:%d", node.Address(), 8080)
serviceURL := fmt.Sprintf("http://[%s]:%d/api", peerIP, 8080)
```

Use bare IPv6 only when the API expects an IP without a port. Supported network
strings are `tcp`, `tcp6`, `udp`, and `udp6` where applicable.

## HTTP client

```go
transport := &http.Transport{
    DialContext:         node.DialContext,
    MaxConnsPerHost:     4,
    MaxIdleConnsPerHost: 2,
    IdleConnTimeout:     30 * time.Second,
}

client := &http.Client{
    Transport: transport,
    Timeout:   15 * time.Second,
}
```

These connection values suit a small example, not every deployment. Choose
them from workload and backend capacity. A request-specific context is
preferable when different operations need different budgets.

```go
req, err := http.NewRequestWithContext(ctx, http.MethodGet,
    "http://[200:abcd::1]:8080/api", nil)
if err != nil {
    return err
}
resp, err := client.Do(req)
```

Close response bodies and reuse the transport. For costly large transfers, see
[load-safety.md](load-safety.md); small responses do not need a chunk protocol.

## HTTP server

```go
ln, err := node.Listen("tcp", fmt.Sprintf("[%s]:%d", node.Address(), 8080))
if err != nil {
    return err
}

srv := &http.Server{
    Handler:           mux,
    ReadHeaderTimeout: 10 * time.Second,
    IdleTimeout:       60 * time.Second,
}

serveErr := make(chan error, 1)
go func() {
    serveErr <- srv.Serve(ln)
}()
```

The application owns `http.Server`. On shutdown, stop accepting requests and
call `srv.Shutdown` before closing the Ratatoskr node. Root close also closes
tracked listeners, but it cannot coordinate application handlers as well as the
server's own shutdown method.

Apply authentication and handler concurrency limits according to the listener's
trust boundary.

## Raw TCP

Outbound:

```go
conn, err := node.DialContext(ctx, "tcp", "[200:abcd::1]:9000")
if err != nil {
    return err
}
defer conn.Close()
```

Inbound:

```go
ln, err := node.Listen("tcp", fmt.Sprintf("[%s]:%d", node.Address(), 9000))
if err != nil {
    return err
}
defer ln.Close()

for {
    conn, err := ln.Accept()
    if err != nil {
        return err
    }
    if !admit() {
        _ = conn.Close()
        continue
    }
    go handleBounded(conn)
}
```

`admit` and `handleBounded` represent an application-specific admission limit.
Do not spawn unlimited handlers on an untrusted listener.

## UDP

```go
pc, err := node.ListenPacket("udp", fmt.Sprintf("[%s]:%d", node.Address(), 5353))
if err != nil {
    return err
}
defer pc.Close()

buf := make([]byte, int(node.MTU()))
n, addr, err := pc.ReadFrom(buf)
```

Use packet deadlines or a closing context according to the server lifecycle.
Control offered rate and define how the protocol handles loss, duplication, and
reordering. More parallel senders can increase overlay loss under saturation.

## Frameworks

| Framework | Server | Client |
| --- | --- | --- |
| `net/http` | `srv.Serve(listener)` | `Transport.DialContext` |
| fasthttp | `fasthttp.Serve(listener, handler)` | `Client.Dial` wrapper |
| gRPC | `grpcServer.Serve(listener)` | `grpc.WithContextDialer` |
| websocket | upgrade on a Ratatoskr-backed HTTP server | library-specific `NetDialContext` |
| `net/rpc` | serve accepted connections | `rpc.NewClient(conn)` |
| gnet or evio | host-local forwarding bridge | host-local forwarding bridge |

### gRPC client

Use a passthrough target so gRPC does not resolve a Yggdrasil literal through
the host DNS stack:

```go
conn, err := grpc.NewClient(
    "passthrough:///[200:abcd::1]:9000",
    grpc.WithContextDialer(func(ctx context.Context, addr string) (net.Conn, error) {
        return node.DialContext(ctx, "tcp", addr)
    }),
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

Yggdrasil encrypts the path, but application TLS can still provide service
identity and end-to-end policy at the application protocol layer.

### fd-based framework bridge

gnet and evio own OS sockets and cannot consume a gVisor listener directly.
Expose a host-local backend through forwarding:

```go
fwd, err := forward.New(forward.ConfigObj{
    Node: node,
    RemoteTCP: []forward.TCPMappingObj{{
        Listen: &net.TCPAddr{IP: node.Address(), Port: 9000},
        Mapped: &net.TCPAddr{IP: net.IPv4(127, 0, 0, 1), Port: 9000},
    }},
    MaxTCPConnections: 64,
})
```

The application owns the forwarder and closes it before the node. Protect the
host-local backend so other local users cannot bypass application controls.

## Multicast and admin

LAN discovery is behind the root core boundary:

```go
if err := node.Core().EnableMulticast(); err != nil {
    return err
}
defer node.Core().DisableMulticast()
```

Interfaces come from `config.NodeConfig.MulticastInterfaces`. Multicast is
platform and network-policy dependent; failure should not silently switch the
application to an unbounded public peer search.

The admin endpoint is an unhardened upstream control interface. Prefer
`Snapshot`, peer methods, NodeInfo, and `mod/probe`. If an operator explicitly
needs admin access, bind it to a protected platform-appropriate local endpoint.

## Platform adapters

Loopback TCP is the portable local bridge. Unix sockets, signals, services, and
filesystem paths require a confirmed target. See
[platforms.md](platforms.md).

The portable application lifecycle is context cancellation followed by
dependant-first explicit close. Do not make POSIX signals part of reusable
networking packages.

## Testing seams

Business logic should depend on the narrow contract it consumes:

```go
type ContextDialer interface {
    DialContext(context.Context, string, string) (net.Conn, error)
}

type Network interface {
    ContextDialer
    Listen(string, string) (net.Listener, error)
    ListenPacket(string, string) (net.PacketConn, error)
}
```

This shape permits fakes without a live mesh. The skill does not create or run
tests automatically; the user chooses checks for the target platform.
