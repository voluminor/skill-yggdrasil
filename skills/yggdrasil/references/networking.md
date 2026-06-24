# Networking Patterns

## Contents

- Address formatting rules
- Outbound HTTP
- HTTP server inside Yggdrasil
- TCP client and server
- UDP client and server
- LAN discovery and admin socket
- Contexts, deadlines, and shutdown
- Test seams

## Address Formatting Rules

Yggdrasil addresses are IPv6. Always bracket IPv6 in host-port strings and
URLs:

```go
addr := fmt.Sprintf("[%s]:%d", node.Address(), 8080)
url := fmt.Sprintf("http://[%s]:%d/api", peerIP, 8080)
```

Use bare IPv6 only when an API expects just an IP string, not host-port:

```go
fmt.Println(node.Address().String()) // "200:..."
```

## Outbound HTTP

Use `node.DialContext` as an `http.Transport` dialer:

```go
client := &http.Client{
    Transport: &http.Transport{
        DialContext: node.DialContext,
    },
    Timeout: 15 * time.Second,
}

resp, err := client.Get("http://[200:abcd::1]:8080/api")
if err != nil {
    return fmt.Errorf("request over yggdrasil: %w", err)
}
defer resp.Body.Close()
```

Use `DisableKeepAlives: true` for simple peer-to-peer demos where stale
connections are more confusing than useful:

```go
client := &http.Client{
    Transport: &http.Transport{
        DialContext:       node.DialContext,
        DisableKeepAlives: true,
    },
    Timeout: 10 * time.Second,
}
```

For real use over the mesh, bound the transport — `MaxConnsPerHost`,
`MaxIdleConnsPerHost`, per-request deadlines — and chunk large transfers. See
[integrations.md](integrations.md#constrained-mesh-transfer-pattern).

## HTTP Server Inside Yggdrasil

Listen on the node address and a service port:

```go
addr := fmt.Sprintf("[%s]:%d", node.Address(), 8080)
ln, err := node.Listen("tcp", addr)
if err != nil {
    return fmt.Errorf("listen on yggdrasil: %w", err)
}

srv := &http.Server{
    Handler:           mux,
    ReadHeaderTimeout: 10 * time.Second,
    IdleTimeout:       60 * time.Second,
}

go func() {
    if err := srv.Serve(ln); err != nil && !errors.Is(err, http.ErrServerClosed) {
        log.Printf("yggdrasil http server: %v", err)
    }
}()

go func() {
    <-ctx.Done()
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    _ = srv.Shutdown(shutdownCtx)
}()
```

`node.Close()` closes listeners created through the core, but still shut down
HTTP servers cleanly when the application owns request handling.

## TCP Client And Server

Outbound TCP:

```go
conn, err := node.DialContext(ctx, "tcp", "[200:abcd::1]:9000")
if err != nil {
    return err
}
defer conn.Close()
```

Inbound TCP:

```go
ln, err := node.Listen("tcp", fmt.Sprintf("[%s]:%d", node.Address(), 9000))
if err != nil {
    return err
}
defer ln.Close()

for {
    conn, err := ln.Accept()
    if err != nil {
        if ctx.Err() != nil {
            return nil
        }
        return err
    }
    go handleConn(conn)
}
```

Use the same `net.Conn` patterns as normal Go networking.

## UDP Client And Server

Outbound UDP uses `DialContext` with `udp`:

```go
conn, err := node.DialContext(ctx, "udp", "[200:abcd::1]:5353")
if err != nil {
    return err
}
defer conn.Close()
```

Inbound UDP uses `ListenPacket`:

```go
pc, err := node.ListenPacket("udp", fmt.Sprintf("[%s]:%d", node.Address(), 5353))
if err != nil {
    return err
}
defer pc.Close()

buf := make([]byte, node.MTU())
n, addr, err := pc.ReadFrom(buf)
```

Use `node.MTU()` for packet buffers when the size should match the userspace
stack.

## LAN Discovery And Admin Socket

Multicast lets nodes on the same LAN find each other without configured peers:

```go
if err := node.EnableMulticast(); err != nil {
    return fmt.Errorf("enable multicast: %w", err)
}
defer node.DisableMulticast()
```

`EnableMulticast()` takes no arguments. Which interfaces it uses comes from
`cfg.Config.MulticastInterfaces` (each entry has `Regex`, `Beacon`, `Listen`,
`Port`, `Priority`, `Password`); `yggconfig.GenerateConfig()` fills a working
default. It logs through the node's own logger, so there is no separate logging
dependency to wire up.

The admin socket is off by default (`AdminListen = "none"`). Most apps never
need it because Ratatoskr exposes `Snapshot`, `Ask`/`AskAddr`, and peer methods
directly. Enable it only for compatibility with external `yggdrasilctl`-style
tooling:

```go
if err := node.EnableAdmin("unix:///run/myapp/ygg.sock"); err != nil {
    return err
}
defer node.DisableAdmin()
```

## Contexts, Deadlines, And Shutdown

Use per-operation deadlines for remote calls. A running Yggdrasil node can
outlive a request, but application requests should not block forever:

```go
ctx, cancel := context.WithTimeout(parent, 10*time.Second)
defer cancel()

conn, err := node.DialContext(ctx, "tcp", target)
```

Use `signal.NotifyContext` for command-line services:

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()
```

Pass that context into `ratatoskr.ConfigObj.Ctx` and into your own servers.

## Test Seams

For outbound-only code:

```go
type ContextDialer interface {
    DialContext(context.Context, string, string) (net.Conn, error)
}
```

For services:

```go
type YggNetwork interface {
    DialContext(context.Context, string, string) (net.Conn, error)
    Listen(string, string) (net.Listener, error)
    ListenPacket(string, string) (net.PacketConn, error)
}
```

Use fakes or in-memory pipes for business logic tests. Keep live Ratatoskr
integration tests behind an explicit build tag or environment variable.
