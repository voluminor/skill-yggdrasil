# Load safety and overlay-friendly design

Ratatoskr v1 bounds most internal work. Applications must preserve those
bounds, configure the remaining trust-sensitive limits, and avoid generating
work that has no user value.

Exact v1.1.0 defaults are in [api-v1.1.md](api-v1.1.md).

## Start with the trust boundary

Classify every endpoint:

| Boundary | Typical example | Required decision |
| --- | --- | --- |
| Process-private | direct module call | cancellation and ownership |
| Host-local | loopback SOCKS or bridge | local-user trust and resource cap |
| Trusted mesh | controlled service peers | peer identity and workload budget |
| Untrusted mesh | public Yggdrasil listener | admission, authentication, timeouts, observability |
| Public/clearnet bridge | proxy between networks | both security domains and abuse controls |

Encryption protects the path. It does not authorize a remote node to consume
unbounded memory, goroutines, sockets, DNS work, or overlay bandwidth.

## Keep bounded defaults

Do not tune every field preemptively. Ratatoskr already bounds root shutdown,
SOCKS connections and UDP work, resolver flights and cache, NodeInfo work, peer
probes, and topology traversal.

Defaults that need an explicit deployment decision include:

- forwarding `MaxTCPConnections`: zero is unlimited;
- forwarding `MaxUDPSessions`: zero is unlimited;
- SOCKS `MaxAssociateTargetsPerPrincipal`: non-positive is unlimited.

These are not one blanket rule. Configure the resource that exists:

- a TCP-only forwarder needs a TCP admission decision, not a UDP cap;
- a UDP-only forwarder needs a UDP session decision and positive idle timeout;
- a loopback SOCKS service may rely on host-local trust;
- SOCKS shared by several users needs authentication and principal isolation.

Set a number from backend capacity, memory budget, expected concurrency, and
mesh-path measurements. Example values in references are starting points only.

## Application admission

Keep application work below library admission limits:

- use a bounded worker pool rather than one goroutine per target;
- bound request and job queues;
- deduplicate keys or names before dispatch;
- reserve capacity for interactive and maintenance work;
- reject or defer new work before creating remote traffic;
- do not construct extra resolver or probe instances to bypass a busy error.

Same-key coalescing saves remote work but does not make an unbounded caller queue
safe.

## Error policy

Classify errors before retrying:

| Class | Response |
| --- | --- |
| `ErrAskBusy`, `ErrResolveBusy`, `ErrLookupBusy`, `ErrProbeBusy` | Keep partial data, wait with bounded jitter, then defer or skip |
| Context cancellation | Stop; caller no longer wants the result |
| Deadline exceeded | Return partial data or failure; do not retry immediately |
| Invalid configuration or input | Correct the input; no retry |
| Remote unreachable | Small attempt budget with a wider delay |
| Local backend failure | Protect the overlay from rapid reconnects |
| Closed component | Stop dispatch and follow shutdown ownership |

Backoff has a maximum attempt count and responds to parent cancellation. A busy
signal says the node is protecting itself; immediate retry defeats that
protection.

## Observe local counters

Read counters at a modest local interval and alert on growth rate:

```go
snap := node.Snapshot()
socksState := snap.SOCKS

forwardState := fwd.Snapshot()
```

Useful SOCKS signals:

- active connections and UDP targets;
- pending UDP target creation;
- rejected UDP targets;
- dropped UDP packets.

Useful forwarding signals:

- active TCP and UDP sessions;
- session-queue and reverse-write UDP drops;
- terminal listener errors.

Counters are cumulative. A high stable total may describe an old event; a
growing rate describes current pressure. Raising a cap is justified only when
the backend, process, and mesh path have headroom. Otherwise reduce offered
load.

Snapshots are local operations and suit health endpoints. Publishing these
counters through NodeInfo would turn discovery into telemetry and add needless
overlay churn.

## Topology budget

Use local path, tree, peer, and snapshot state first. A remote topology tree is
the most expensive diagnostic surface because it asks other nodes for peers
across explored branches.

For every tree:

- require an explicit user or rare-event trigger;
- set context deadline, small depth, total-node cap, and worker concurrency;
- disclose `Truncated` results;
- avoid simultaneous scans from several nodes;
- reuse a probe object and bounded cached result when stale data is acceptable.

Do not build routine monitoring around whole-mesh scans.

## Requests and transfers

For ordinary requests:

- use request-scoped cancellation or a deadline where remote waiting matters;
- cap transport connections and idle connections for the workload;
- reuse connections rather than creating a chatty protocol;
- keep payloads and polling frequency proportional to user value.

Do not force chunking onto every file. Add chunking when the cost of a complete
retry is material. A resumable transfer should define:

- independently retriable ranges or chunks;
- a per-attempt deadline;
- size or checksum verification;
- persisted completion state when restart resume is required;
- bounded retry with jitter;
- cleanup of partial local data.

The application chooses chunk size and the threshold for this protocol. A
small request can remain one request.

## UDP

UDP loss under saturation is not repaired by more goroutines. Keep datagrams
within the configured payload bound, control sender rate, monitor loss and
latency, and decide whether the application can tolerate missing or reordered
messages.

Use TCP or an application reliability layer when delivery and ordering are
required. Do not implement unlimited retransmission over UDP.

## Scheduler guidance

Ratatoskr's same-host benchmark found severe sustained UDP degradation with one
Go scheduler context and a one-stream plateau at two or more contexts on that
host. The defensible operational rule is:

- use at least two scheduler contexts for an embedded node expected to carry
  sustained traffic when the platform permits it;
- do not claim that two is optimal for every concurrent workload;
- do not impose the rule on idle tools, mobile apps, or single-core targets;
- remeasure after meaningful changes to Yggdrasil, gVisor, MTU, crypto,
  queues, or the NIC bridge.

Go normally derives `GOMAXPROCS` from available CPU. Avoid overriding it to one
for a loaded node without measurement.

## Degradation policy

When capacity is exhausted:

1. preserve useful partial results;
2. reject or defer new work locally;
3. reduce polling and fan-out;
4. keep interactive work ahead of bulk diagnostics;
5. report pressure through local metrics;
6. recover gradually instead of releasing a large retry wave.

This policy protects both the process and the volunteer-operated overlay.
