# Sandbox

A sandbox is the runtime boundary where agent code executes. It is created by a
compute runtime and managed inside the workload by `openshell-sandbox`, the
sandbox supervisor.

## Runtime Model

Each sandbox workload has two trust levels:

| Process | Role |
|---|---|
| Supervisor | Starts as root inside the workload, prepares isolation, runs the proxy, fetches config, injects credentials, serves the relay socket, and launches child processes. |
| Agent child | Runs as an unprivileged user with filesystem, process, and network restrictions applied. |

The supervisor keeps enough privilege to manage the sandbox, but the agent child
loses that privilege before user code runs.

## Startup Flow

1. The compute runtime starts the workload with sandbox identity, callback
   endpoint, TLS or secret material, image metadata, and initial command.
2. The supervisor loads policy and runtime settings from local files or the
   gateway, depending on mode.
3. It prepares filesystem access, process restrictions, network namespace
   routing, trust stores, provider credential resolution, and inference routes.
4. It starts the policy proxy and local SSH server.
5. It opens a supervisor session back to the gateway for connect, exec, file
   sync, config polling, and log push.
6. It launches the agent command as the restricted sandbox user.

## Isolation Layers

OpenShell uses overlapping controls rather than a single sandbox primitive:

| Layer | Purpose |
|---|---|
| Filesystem policy | Landlock restricts the paths the agent can read or write. |
| Process policy | The child process runs as a non-root user with reduced privileges. Supplemental groups can be injected to give the sandbox agent read access to host-owned directories mounted via volume mounts. |
| Seccomp | Blocks dangerous syscalls, including raw socket paths that bypass the proxy. |
| Network namespace | Forces ordinary agent egress through the local CONNECT proxy. |
| Policy proxy | Evaluates destination, binary identity, TLS/L7 rules, SSRF checks, and inference interception. |

The supervisor may enrich baseline filesystem allowances for runtime-required
paths, such as proxy support files or GPU device paths when a GPU is present.

## Volume Mounts

Host directories can be mounted into a sandbox at creation time using
`volume_mounts` on the `SandboxTemplate`. Each mount specifies a `host_path`,
a `container_path`, and a `read_only` flag.

- The Docker driver binds each mount with `:ro,Z` or `:rw,Z`. The `:Z` label
  applies a private SELinux context so each sandbox gets its own label.
- The gateway validates mounts before forwarding to the compute driver: paths
  must be absolute, non-empty, free of null bytes, within the 4096-byte limit,
  and there must be no duplicate `container_path` entries. A maximum of 16
  mounts per sandbox is enforced.
- When `include_volume_mounts` is set on `FilesystemPolicy`, the gateway
  expands the Landlock allowlist from the sandbox's volume mounts at
  `GetSandboxConfig` time. Container paths for read-only mounts are added to the
  read allowlist; read-write mounts go to the read-write allowlist. This
  expansion is dynamic — the stored policy is not mutated.
- At the same time the Landlock allowlist is expanded, the gateway `stat()`s
  each `host_path` and adds the owning GID (as a decimal string) to
  `ProcessPolicy.supplemental_groups`. The supervisor merges these GIDs with the
  result of `initgroups()` via `setgroups()` before `setuid()`, so the sandbox
  agent can access host-owned directories without requiring world-readable
  permissions. GID 0 is never injected. `stat()` failure is non-fatal (logged as
  a warning and skipped). The stored policy is not mutated.
- Creating a sandbox with non-empty `volume_mounts` requires the `sandbox:mount`
  scope. The CLI warns the user and prompts before submitting mounts of sensitive
  host paths (e.g. `/etc`, `/root`, `.ssh`, `.aws`). Non-interactive callers
  must pass `--no-mount-warnings` to skip prompts; the flag is rejected if no
  dangerous paths are present.

## Network and Inference

All ordinary agent egress is routed through the sandbox proxy. The proxy
identifies the calling binary, checks trust-on-first-use binary identity, rejects
unsafe internal destinations, and evaluates the active policy.

`https://inference.local` is special. It bypasses OPA network policy and is
handled by the inference interception path:

1. The proxy terminates the local TLS connection with the sandbox CA.
2. It detects known OpenAI, Anthropic, and compatible inference request shapes.
3. It strips caller-supplied credentials and disallowed headers.
4. It forwards through `openshell-router` using the route bundle fetched from
   the gateway.

External inference endpoints that do not use `inference.local` are treated like
ordinary network traffic and must be allowed by policy.

## Credentials

Provider credentials are stored at the gateway and fetched by the supervisor at
runtime. The supervisor injects resolved environment variables into the initial
agent process and SSH child processes. Driver-controlled environment variables
override template values so sandbox images cannot spoof identity, callback, or
relay settings.

Credential placeholders in proxied HTTP requests can be resolved by the proxy
when policy allows the target endpoint. Secrets must not be logged in OCSF or
plain tracing output.

## Connect and Logs

The supervisor runs an SSH server on a Unix socket inside the sandbox. The
gateway reaches it through the outbound supervisor relay, not by dialing the
sandbox workload directly. The relay supports:

- Interactive shell sessions.
- Command execution.
- Tar-based file sync.
- Port forwarding where supported by the CLI/TUI surface.

Sandbox logs are emitted locally and can also be pushed back to the gateway.
Security-relevant sandbox behavior uses OCSF structured events; internal
diagnostics use ordinary tracing.

## Failure Behavior

- If gateway config polling fails, the sandbox keeps its last-known-good policy.
- If a live policy update is invalid, the supervisor rejects it and keeps the
  current policy.
- Existing raw byte streams are connection scoped. Dynamic policy changes apply
  to new connections or the next parsed HTTP request where the proxy can safely
  re-evaluate.
- If the supervisor relay drops, the sandbox can keep running, but connect and
  exec operations fail until the supervisor registers again.
