# Platform guidance

Portability has two layers: the Agent Skill must not depend on one client, and
the generated application must match the target named by the user. A program
does not need to support unrelated operating systems.

## Agent-client baseline

The portable skill format uses standard Markdown, relative references, and
standard frontmatter fields. Instructions name actions rather than tools from a
particular client.

- Do not require tool names such as `Read`, `Bash`, `Agent`, or
  `AskUserQuestion`.
- Do not require Go or network access for design and review tasks.
- Do not install missing tools automatically.
- Do not assume a POSIX shell, PowerShell, browser, or GUI.
- Keep reference paths relative to the skill root.
- Treat validation commands as suggestions for the user's target.

## Evidence labels

Use one of these labels whenever platform support matters:

| Label | Evidence in Ratatoskr v1.1.0 | Meaning |
| --- | --- | --- |
| Runtime-tested | Release-source CI on Linux, macOS, and Windows | Tests run on that OS; deployment-specific behavior still belongs to the application |
| Compile-tested | Ratatoskr's release cross-build targets | Source compiles for the named GOOS/GOARCH; runtime behavior is not proven |
| Experimental | `cmd/embedded/mobile` for Android and iOS | The Go module compiles and its unit tests pass; target artifacts and SDK runtime still need validation |
| Unknown | Unlisted targets such as WASM and Plan 9 | Inspect dependencies and the user's environment before making a support claim |

Ratatoskr cross-builds these families in its v1.1.0 release workflow:

- Linux on amd64, arm64, armv6, armv7, 386, riscv64, mips, mipsle,
  mips64, mips64le, ppc64, ppc64le, and s390x;
- Windows on amd64, arm64, and 386;
- macOS on amd64 and arm64;
- FreeBSD on amd64, arm64, and 386;
- OpenBSD on amd64 and arm64;
- NetBSD on amd64 and arm64.

This list records v1.1.0 evidence. Check the selected release before repeating
it for a later version.

## Platform decision

Before generating code, identify:

1. target GOOS and GOARCH;
2. executable, service, library, or mobile binding;
3. filesystem and credential-storage constraints;
4. whether local TCP or a platform-specific local socket is appropriate;
5. lifecycle owner and shutdown signal source;
6. background, battery, and memory restrictions;
7. which checks the user considers relevant.

If the target is missing, use neutral Go APIs and mark the decisions that need a
platform. Do not run or require a cross-platform matrix.

## Local endpoints

Loopback TCP is the portable baseline for SOCKS, admin tooling, and bridges:

```text
127.0.0.1:1080
[::1]:1080
```

Unix socket behavior is platform-specific and is not release-tested on
Windows. Use it only after confirming the target and directory security. On
Unix-like systems, Ratatoskr requires a private parent directory for SOCKS Unix
sockets and refuses unsafe or replaced paths.

Do not hardcode `/tmp`, `/run`, drive letters, or path separators in common
examples. Use `os.UserCacheDir`, `os.MkdirTemp`, and `path/filepath` when the
application owns a filesystem path.

## Shutdown

The networking lifecycle is portable:

1. cancel the application context;
2. stop application servers and standalone modules;
3. call `Close` explicitly.

OS signals are executable adapters. `os.Interrupt` is a useful common input,
but signal availability and service-control behavior differ by platform.
Keep systemd, launchd, Windows Service, containers, and mobile lifecycle hooks
outside the Ratatoskr networking layer.

## Windows

- Prefer loopback TCP for local SOCKS and bridges.
- Use Windows Service control or the host application's context for service
  lifecycle; do not assume Unix signals.
- Use Go filesystem APIs instead of Unix permissions and paths.
- Windows is runtime-tested, but an individual transport can still have
  upstream platform constraints.

## Unix-like systems

- Unix sockets may be used when their parent directory is private.
- Admin endpoints are full-control interfaces; bind them to a protected local
  endpoint and never expose them to untrusted mesh traffic.
- Signal handling belongs to the executable and must still lead to explicit
  dependant-first shutdown.

## BSD

FreeBSD, OpenBSD, and NetBSD targets listed above are compile-tested. Do not
describe them as runtime-tested without evidence from the user's deployment.
Check Yggdrasil transport behavior and OS resource limits on the actual host.

## Android and iOS

The upstream gomobile example exposes lifecycle, peers, SOCKS, forwarding,
callbacks, and basic diagnostics. Its Go module builds and passes unit tests,
but Android and iOS artifacts still need target-SDK builds and runtime checks.
Treat it as experimental.

Mobile guidance:

- let the host application own start and stop;
- keep callbacks short and non-blocking;
- keep peer sets, queues, and concurrency small;
- avoid topology scans by default;
- account for background suspension, battery, and network changes;
- do not promise persistent service operation outside platform policy.

## Unknown targets

For WASM, Plan 9, and unlisted GOOS/GOARCH combinations, report the target as
unknown. Do not invent an alternate transport or claim that userspace gVisor
alone guarantees compatibility. Check Go, gVisor, Yggdrasil, and required
transport support first.

## Commands and shells

Prefer shell-neutral commands such as:

```text
go list -m github.com/voluminor/ratatoskr
go doc github.com/voluminor/ratatoskr
```

When environment variables or multiline commands are necessary, label Bash and
PowerShell variants separately. Do not present POSIX assignment syntax as a
cross-platform command.
