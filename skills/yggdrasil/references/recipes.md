# Ratatoskr Recipes

## Contents

- Embedded HTTP client
- HTTP service inside Yggdrasil
- Static peers
- Peer manager startup
- SOCKS5 proxy
- Local TCP forwarding
- NodeInfo query
- Snapshot endpoint
- Start/stop wrapper pattern

## Embedded HTTP Client

Use this when an application needs to call services reachable only through
Yggdrasil:

```go
func newYggHTTPClient(node *ratatoskr.Obj) *http.Client {
    return &http.Client{
        Transport: &http.Transport{
            DialContext: node.DialContext,
        },
        Timeout: 15 * time.Second,
    }
}
```

Call:

```go
resp, err := client.Get("http://[200:abcd::1]:8080/api")
```

## HTTP Service Inside Yggdrasil

```go
func serveYggHTTP(ctx context.Context, node *ratatoskr.Obj, port int, h http.Handler) error {
    addr := fmt.Sprintf("[%s]:%d", node.Address(), port)
    ln, err := node.Listen("tcp", addr)
    if err != nil {
        return fmt.Errorf("listen yggdrasil http: %w", err)
    }

    srv := &http.Server{
        Handler:           h,
        ReadHeaderTimeout: 10 * time.Second,
        IdleTimeout:       60 * time.Second,
    }

    go func() {
        <-ctx.Done()
        shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()
        _ = srv.Shutdown(shutdownCtx)
    }()

    go func() {
        if err := srv.Serve(ln); err != nil && !errors.Is(err, http.ErrServerClosed) {
            log.Printf("yggdrasil http: %v", err)
        }
    }()

    return nil
}
```

## Static Peers

```go
func startNodeWithStaticPeers(ctx context.Context, peers []string) (*ratatoskr.Obj, error) {
    cfg := yggconfig.GenerateConfig()
    cfg.AdminListen = "none"
    cfg.Peers = peers

    node, err := ratatoskr.New(ratatoskr.ConfigObj{
        Ctx:             ctx,
        Config:          cfg,
        CoreStopTimeout: 5 * time.Second,
    })
    if err != nil {
        return nil, fmt.Errorf("start ratatoskr: %w", err)
    }
    return node, nil
}
```

The caller must call `Close`.

## Peer Manager Startup

```go
func startNodeWithPeerManager(ctx context.Context, peers []string, logger yggcore.Logger) (*ratatoskr.Obj, error) {
    cfg := yggconfig.GenerateConfig()
    cfg.AdminListen = "none"

    node, err := ratatoskr.New(ratatoskr.ConfigObj{
        Ctx:    ctx,
        Config: cfg,
        Logger: logger,
        Peers: &peermgr.ConfigObj{
            Peers:           peers,
            MaxPerProto:     1, // one best peer per protocol
            BatchSize:       4,
            RefreshInterval: 5 * time.Minute,
        },
    })
    if err != nil {
        return nil, fmt.Errorf("start ratatoskr with peer manager: %w", err)
    }
    return node, nil
}
```

Do not also set `cfg.Peers`.

## SOCKS5 Proxy

```go
func enableSOCKS(node *ratatoskr.Obj) error {
    return node.EnableSOCKS(ratatoskr.SOCKSConfigObj{
        Addr:           "127.0.0.1:1080",
        Nameserver:     "",
        MaxConnections: 100,
    })
}
```

Use `Nameserver` only when the deployment has a DNS server reachable through
Yggdrasil. `.pk.ygg` and IP literals do not need DNS.

## Local TCP Forwarding

Forward local `127.0.0.1:8080` to a Yggdrasil HTTP service:

```go
func startForwarding(ctx context.Context, node *ratatoskr.Obj, logger yggcore.Logger) *forward.ManagerObj {
    mgr := forward.New(logger, 120*time.Second)
    mgr.AddLocalTCP(forward.TCPMappingObj{
        Listen: &net.TCPAddr{IP: net.IPv4(127, 0, 0, 1), Port: 8080},
        Mapped: &net.TCPAddr{IP: net.ParseIP("200:abcd::1"), Port: 80},
    })
    mgr.Start(ctx, node)
    return mgr
}
```

On shutdown:

```go
cancel()
mgr.Wait()
```

## NodeInfo Query

```go
func askNode(ctx context.Context, node *ratatoskr.Obj, target string) (*ninfo.AskResultObj, error) {
    ctx, cancel := context.WithTimeout(ctx, 15*time.Second)
    defer cancel()

    res, err := node.AskAddr(ctx, target)
    if err != nil {
        return nil, fmt.Errorf("ask %s: %w", target, err)
    }
    return res, nil
}
```

`target` can be a `.pk.ygg` name, raw public key hex, bare IPv6, or
`[ipv6]:port`.

## Snapshot Endpoint

```go
func snapshotHandler(node *ratatoskr.Obj) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        _ = json.NewEncoder(w).Encode(node.Snapshot())
    })
}
```

Use this for local admin endpoints, not public unauthenticated internet
endpoints, unless exposing peer and key metadata is acceptable.

## Start/Stop Wrapper Pattern

For long-lived services and desktop/mobile apps with explicit start/stop
controls, follow the wrapper lifecycle documented in
[advanced-and-examples.md](advanced-and-examples.md#cli-and-example-apps--patterns-not-imports):
store config while stopped, `ratatoskr.New` to start, enable SOCKS/multicast and
forwarding only after start, then on stop cancel forwarding, `node.Close()`, and
`mgr.Wait()` before clearing running state.
