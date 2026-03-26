# Layer 2: OS-Level Sandboxing

> **Last verified: 2026-03-26**

---

## Principle

Kernel-enforced restrictions on what an AI agent process can do -- which syscalls it
can make, which files it can access, and which resources it can consume. Unlike
infrastructure isolation (Layer 1), the agent runs on the host OS but within a
constrained sandbox. This is the primary security mechanism used by Claude Code and
Cursor today.

---

## Why This Layer Matters

- **Blast-radius containment.** Directly limits what a compromised agent can touch on
  the host system -- files outside the project, credentials in `~/.ssh`, cloud configs
  in `~/.aws`.
- **Low overhead.** No hypervisor, no VM boot time. Practical for local development
  where developers expect sub-second startup.
- **Kernel enforcement.** The sandbox is enforced by the OS kernel, not by the
  application. A malicious process cannot simply call `setenv("SANDBOX", "off")` to
  escape -- it must find a kernel vulnerability.
- **Defense-in-depth.** Combines with Layer 1 (infrastructure isolation) and Layer 3
  (network controls) for layered protection.
- **Reduced permission fatigue.** Claude Code's sandbox reduces permission prompts by
  84% while maintaining security, because safe operations within the sandbox boundary
  are auto-approved.

---

## Approaches

### macOS Seatbelt (sandbox-exec)

**How it works:**

macOS includes a Mandatory Access Control (MAC) framework called Seatbelt. It uses
SBPL (Seatbelt Profile Language) -- a Scheme-like declarative policy language -- to
define what a process is allowed to do. The sandbox is activated via
`sandbox-exec -f <profile> <command>`.

Controls available: file read/write per path, network access (inbound/outbound),
process execution, IPC, Mach services, sysctl reads, and signal delivery.

**Used by:** Claude Code (macOS), Cursor (agent sandboxing feature)

**Example SBPL profile structure** (simplified):

```scheme
(version 1)
(deny default)                          ; Deny everything by default

; Allow read access to project directory
(allow file-read*
    (subpath "/Users/dev/project"))

; Allow read-write to specific workspace
(allow file-write*
    (subpath "/Users/dev/project/src"))

; Block network access
(deny network*)

; Allow specific process operations
(allow process-exec
    (literal "/usr/bin/git"))
```

**Key characteristics:**

| Property | Detail |
|----------|--------|
| Default posture | Deny-all -- only explicitly allowed operations succeed |
| Path granularity | `subpath`, `literal`, `regex` matchers |
| Bypass resistance | Kernel-enforced; sandboxed process cannot disable it |
| Status | `sandbox-exec` is deprecated by Apple but remains functional and widely used |
| Documentation | Apple does not publish official SBPL docs; profiles are reverse-engineered from system profiles in `/System/Library/Sandbox/Profiles/` |

---

### Linux: bubblewrap + seccomp + Landlock

Claude Code on Linux uses a combination of three kernel mechanisms, layered together:

**bubblewrap (bwrap):**

- Lightweight unprivileged sandbox using Linux namespaces
- Creates isolated PID, mount, network, and user namespaces
- No root privileges required (uses `CLONE_NEWUSER`)
- Provides the outer container: the sandboxed process sees a restricted filesystem
  and cannot interact with host processes

**seccomp (Secure Computing Mode):**

- Filters syscalls at the kernel level using BPF (Berkeley Packet Filter) programs
- Defines an allowlist of syscalls the process may invoke
- Blocks dangerous syscalls: `ptrace`, `mount`, `reboot`, `kexec_load`,
  `init_module`, etc.
- Very low overhead -- BPF programs are JIT-compiled in the kernel

**Landlock:**

- Filesystem access control for unprivileged processes (Linux 5.13+)
- Unlike DAC permissions (which return "permission denied"), Landlock makes files
  completely invisible to the sandboxed process
- Can restrict: file read, file write, file execute, directory creation, directory
  removal, file rename, file truncate
- Stacks with traditional DAC and other Linux Security Modules

**Combined architecture** (as used by Claude Code):

```
┌──────────────────────────────────┐
│ bubblewrap (namespace isolation) │
│ ┌──────────────────────────────┐ │
│ │ seccomp (syscall filtering)  │ │
│ │ ┌──────────────────────────┐ │ │
│ │ │ Landlock (fs restriction)│ │ │
│ │ │ ┌──────────────────────┐ │ │ │
│ │ │ │   AI Agent Process   │ │ │ │
│ │ │ └──────────────────────┘ │ │ │
│ │ └──────────────────────────┘ │ │
│ └──────────────────────────────┘ │
└──────────────────────────────────┘
```

**Reference implementation:** Anthropic's open-source `sandbox-runtime`
- GitHub: https://github.com/anthropic-experimental/sandbox-runtime

---

### gVisor

- Google's user-space kernel implementation (project `runsc`)
- Intercepts application syscalls via ptrace or KVM and re-implements them in a
  sandboxed user-space kernel called the Sentry
- Stronger isolation than containers (the host kernel never directly processes
  application syscalls)
- Lighter than full VMs (no hypervisor, no guest OS boot)
- Tradeoff: syscall compatibility -- not all Linux syscalls are implemented, and some
  behave slightly differently
- Used by Google Cloud Run, GKE Sandbox, and some CI providers
- Good middle ground when container isolation is insufficient but VMs are too expensive

---

## Implementation Guide

### Claude Code: Sandbox Mode

Sandboxing is enabled by default on supported platforms. No configuration is required.

| Platform | Mechanism | Enabled by default |
|----------|-----------|-------------------|
| macOS | Seatbelt (sandbox-exec) | Yes |
| Linux | bubblewrap + seccomp + Landlock | Yes |

**What the sandbox permits:**

- Read/write access to the current project directory and its subdirectories
- Read access to required system paths (`/usr`, `/lib`, tool binaries)
- Execution of common developer tools (`git`, `node`, `python`, etc.)
- Outbound network to Anthropic API endpoints (for model communication)

**What the sandbox blocks:**

- File access outside the project directory (e.g., `~/.ssh`, `~/.aws`, `~/.config`)
- Arbitrary outbound network connections
- Dangerous syscalls (`ptrace`, `mount`, `reboot`, etc. on Linux)
- Process inspection or manipulation of non-child processes

**Auto-allow mode:** When the sandbox is active, Bash commands that fall within the
sandbox boundary are automatically approved without prompting. Commands that require
access outside the sandbox (e.g., network requests, file access beyond the project)
fall through to the standard permission prompt.

**Verify sandbox is active:**

```bash
# Check Claude Code sandbox status
claude --version --verbose
# Look for "sandbox: enabled" in output

# On macOS, verify a process is sandboxed:
ps -eo pid,comm | grep claude   # get PID
sandbox-check <PID>             # returns 0 if sandboxed
```

---

### Custom Seatbelt Profile (macOS)

To create a custom SBPL profile for an AI agent:

1. Start from a deny-all baseline: `(version 1) (deny default)`
2. Add read access to the project directory and required system paths
3. Add write access only to the project workspace
4. Explicitly allow needed process executions (git, node, python)
5. Deny all network access unless the agent requires API connectivity
6. Test iteratively -- sandbox violations are logged to the system log

```bash
# View sandbox violations in real time:
log stream --predicate 'subsystem == "com.apple.sandbox"' --level error
```

Profiles are stored as `.sb` files. Apply with:

```bash
sandbox-exec -f /path/to/profile.sb /usr/local/bin/node agent.js
```

---

### Custom bubblewrap Configuration (Linux)

A practical bwrap command for sandboxing an AI agent:

```bash
bwrap \
  --ro-bind /usr /usr \
  --ro-bind /lib /lib \
  --ro-bind /lib64 /lib64 \
  --ro-bind /bin /bin \
  --ro-bind /etc/resolv.conf /etc/resolv.conf \
  --bind /path/to/project /workspace \
  --tmpfs /tmp \
  --proc /proc \
  --dev /dev \
  --unshare-all \
  --new-session \
  --die-with-parent \
  -- /usr/bin/node agent.js
```

Key flags:

| Flag | Purpose |
|------|---------|
| `--ro-bind` | Mount host path as read-only inside the sandbox |
| `--bind` | Mount host path as read-write (use only for the project dir) |
| `--tmpfs` | Create a temporary filesystem (isolated from host) |
| `--unshare-all` | Create new PID, mount, network, user, IPC, UTS namespaces |
| `--new-session` | Prevent the sandboxed process from using terminal job control |
| `--die-with-parent` | Kill the sandbox if the parent process dies |

To add seccomp filtering, generate a BPF filter and pass it via `--seccomp <fd>`.
To add Landlock, apply it programmatically inside the sandboxed process at startup
(Landlock is self-imposed and inherited by child processes).

---

## Limitations

- **macOS Seatbelt deprecation.** Apple has deprecated `sandbox-exec` but has not
  removed it or provided a public replacement. Future macOS versions may break
  compatibility without warning.
- **Linux kernel version requirements.** Landlock requires Linux 5.13+. File truncate
  restrictions require 5.19+. Some CI environments run older kernels.
- **Syscall compatibility.** seccomp and gVisor may block syscalls that legitimate
  developer tools need. Overly restrictive profiles break builds; overly permissive
  profiles miss threats.
- **Configuration complexity.** Writing correct SBPL or seccomp-bpf profiles requires
  deep knowledge of OS internals. Mistakes silently weaken the sandbox.
- **Bypass vectors.** Sandbox escapes exist, though they typically require kernel
  vulnerabilities or exploiting allowed operations in unintended ways.
- **No network granularity.** OS sandboxing provides coarse network control (allow all
  or deny all). Fine-grained network policies (allow specific hosts/ports) require
  Layer 3 (network-level controls).
- **Race conditions at startup.** If the agent executes hooks or configuration files
  before the sandbox is fully applied, those operations run unsandboxed (see
  CVE-2025-59536).

---

## Tool-Specific Notes

### Claude Code

- Most mature sandbox implementation among AI coding agents
- macOS: Seatbelt with project-scoped file access and selective network
- Linux: bubblewrap + seccomp + Landlock (three-layer defense)
- Open-source reference: https://github.com/anthropic-experimental/sandbox-runtime
- Sandbox reduces permission prompts by 84% via auto-allow for safe operations

### Cursor

- Added agent sandboxing in 2025 (blog: https://cursor.com/blog/agent-sandboxing)
- Uses macOS Seatbelt on Mac
- Known bypass: CVE-2025-59944 -- case-sensitivity mismatch in file path checks
  allowed reading files that should have been blocked
- `.cursorignore` is described as "best effort" -- the sandbox is stronger but still
  not comprehensive

### GitHub Copilot CLI

- No built-in OS-level sandboxing as of this writing
- Relies on trust directories and manual permission approval
- Must be sandboxed externally via container or OS-level tools if stronger isolation
  is required

---

## Related CVEs

| CVE | Tool | Summary |
|-----|------|---------|
| CVE-2025-59944 | Cursor | Case-sensitivity bypass in sandbox file path checks. The sandbox blocked `/Users/dev/.ssh/id_rsa` but allowed `/Users/Dev/.ssh/id_rsa` on case-insensitive filesystems. Demonstrates that sandbox path checks must match filesystem behavior. |
| CVE-2025-59536 | Claude Code | Hooks and MCP server configurations executed before the sandbox fully constrained the process, allowing pre-sandbox code execution. |

---

## References

- Anthropic Engineering: Claude Code Sandboxing -- https://www.anthropic.com/engineering/claude-code-sandboxing
- Claude Code Sandboxing Docs -- https://code.claude.com/docs/en/sandboxing
- Anthropic sandbox-runtime (open source) -- https://github.com/anthropic-experimental/sandbox-runtime
- Cursor Agent Sandboxing Blog -- https://cursor.com/blog/agent-sandboxing
- A Deep Dive on Agent Sandboxes (Pierce Freeman) -- https://pierce.dev/notes/a-deep-dive-on-agent-sandboxes
- bubblewrap -- https://github.com/containers/bubblewrap
- Landlock LSM -- https://landlock.io/
- gVisor -- https://gvisor.dev/
- Cursor Sandboxing Leaks Secrets (Luca Becker) -- https://luca-becker.me/blog/cursor-sandboxing-leaks-secrets/
