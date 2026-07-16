---
name: yggdrasil
description: "Use when building, reviewing, or debugging Go applications that embed Ratatoskr for Yggdrasil TCP, UDP, HTTP, SOCKS5, forwarding, peers, NodeInfo, sigils, topology diagnostics, or load handling."
license: LGPL-2.1
compatibility: "Compilation requires a Go toolchain compatible with the selected Ratatoskr version. Review and design tasks can run without Go or network access."
metadata:
  author: SUNsung
  version: "1.0.0"
---

# Yggdrasil networking with Ratatoskr

## Purpose

Use `github.com/voluminor/ratatoskr` to embed a Yggdrasil node in a Go
application. Ratatoskr provides a userspace gVisor TCP/IP stack, so the
application needs no TUN device, root privileges, external Yggdrasil daemon,
or `yggstack` process.

This skill targets the stable Ratatoskr v1 API. The module selected by the
user's project is authoritative; bundled API details are a v1.1.0 snapshot.

## Required workflow

1. Identify the selected Ratatoskr version when the project exposes it.
2. Identify the target OS, architecture, runtime, and deployment constraints.
3. Choose the root facade or a justified standalone module.
4. Classify each listener or bridge by trust boundary.
5. Read only the references needed for the task.
6. Produce code or guidance for the user's target, not an unrelated platform
   matrix.
7. State which target-specific checks the user may run and what remains
   unverified. Do not install tools or run checks automatically.

If the selected module cannot be inspected, use
[references/api-v1.1.md](references/api-v1.1.md) as a labeled fallback and say
that the active API was not verified.

## Choose the smallest justified API

| Need | API |
| --- | --- |
| Typical embedded node, NodeInfo, SOCKS, or managed peers | `ratatoskr.New` |
| Only raw TCP/UDP dial and listen | `mod/core` via `core.New` |
| TCP/UDP port forwarding | application-owned `mod/forward` object |
| Route or topology diagnostics | one reused, application-owned `mod/probe` object |
| Custom transport or isolated module test | the module's narrow consumer interface |

Default to the root facade. Do not choose `core.New` for speculative
micro-optimization.

The root owns core, ninfo, its optional peer manager, and root SOCKS service.
It does not own separately created forwarding, probe, resolver, HTTP, or gRPC
objects. Shut down dependants before the node.

## Reference routing

| Task | Read |
| --- | --- |
| Installing Ratatoskr or selecting a GitHub, HTTPS-mirror, or `.pk.ygg` module identity | [api-v1.1.md](references/api-v1.1.md#module-installation-identities) |
| Migrating pre-v1 code or resolving old examples | [migration-v1.md](references/migration-v1.md) |
| Agent-client, OS, architecture, shell, or mobile concerns | [platforms.md](references/platforms.md) |
| TCP, UDP, HTTP, gRPC, fasthttp, websocket, or fd-based frameworks | [networking.md](references/networking.md) |
| Static/managed peers, SOCKS, resolver, or forwarding | [services.md](references/services.md) |
| NodeInfo, sigils, snapshots, paths, traces, or topology | [discovery.md](references/discovery.md) |
| Untrusted traffic, limits, overload, retries, counters, or large transfers | [load-safety.md](references/load-safety.md) |
| Exact v1.1.0 fields, defaults, errors, and result types | [api-v1.1.md](references/api-v1.1.md) |

## Trust and load rules

Classify exposure as process-private, host-local, trusted mesh, untrusted mesh,
or public/clearnet bridge. Yggdrasil encrypts traffic but does not make every
remote node trusted.

- Keep Ratatoskr's bounded defaults unless measurements justify a change.
- For forwarding exposed to untrusted traffic, set a positive limit for each
  protocol actually enabled. TCP-only forwarding does not need a UDP limit.
- SOCKS beyond loopback needs an explicit authentication and per-principal
  limit decision.
- Treat `Err*Busy` as load shedding: retain partial data, back off with jitter,
  and defer or skip work. Never retry it in a tight loop.
- Bound application queues and worker pools below library admission limits so
  other node consumers retain capacity.
- Deduplicate requests before dispatch even when Ratatoskr can coalesce them.

Read [load-safety.md](references/load-safety.md) before generating a public
listener, bulk operation, periodic job, or high-load service.

## Protect the overlay

Prefer local snapshots, peer state, paths, and counters. A topology tree scan
is remote work across other people's links: run it only on demand with explicit
depth, node, concurrency, and time bounds. Do not recommend periodic whole-mesh
scans.

Keep NodeInfo small and slow-changing. Use it for discovery, not telemetry.
Use ordinary bounded requests for small payloads. Add chunking, verification,
and resume only when a full retry would be expensive.

For sustained traffic, Ratatoskr's benchmark supports using at least two Go
scheduler contexts. Treat this as workload guidance, not a universal rule for
idle tools, mobile apps, or constrained devices.

## Platform claims

Use the evidence labels from [platforms.md](references/platforms.md):

- **runtime-tested:** Linux, macOS, Windows;
- **compile-tested:** the GOOS/GOARCH targets listed by Ratatoskr CI;
- **experimental:** Android and iOS through the gomobile example;
- **unknown:** unlisted targets such as WASM and Plan 9.

Do not turn compile evidence into a runtime guarantee. The skill itself is
portable; an application may intentionally target one platform.

## Stable v1 boundaries

- Use `node.Core()` for multicast, admin, peer retry, and probe wiring. The root
  no longer embeds `core.Interface`.
- Configure a peer set through static peers, the peer manager, or an external
  runtime controller. Do not let several mechanisms own the same peers.
- Prefer built-in sigils. A custom sigil in root `ConfigObj.Sigils` is both
  published locally and registered as a remote parser.
- Prefer `Snapshot` and `probe` to the unhardened admin endpoint.
- Compare wrapped and joined errors with `errors.Is`.

## Validation responsibility

Do not add CI, fixtures, cross-builds, or validation dependencies unless the
user explicitly requests them. Offer checks only for the actual target and
leave execution to the user. For review-only work, report conclusions without
requiring a Go installation.

## Common mistakes

| Mistake | Correction |
| --- | --- |
| Starting `yggstack` or an external daemon | Embed the node with Ratatoskr |
| Mixing v0 and v1 names | Read `migration-v1.md` and target one selected version |
| Assuming `node.Close()` owns `forward` or `probe` | Close application-owned dependants first |
| Treating all encrypted mesh traffic as trusted | Classify the listener's trust boundary |
| Setting both forwarding limits for a single-protocol service | Limit only enabled protocols |
| Treating a BSD cross-build as runtime validation | Label it compile-tested |
| Using Unix sockets or POSIX signals unconditionally | Select a platform adapter |
| Polling topology for routine health | Use local state and on-demand bounded probes |
