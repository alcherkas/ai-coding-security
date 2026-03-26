# Layer 1: Infrastructure Isolation

> Last verified: 2026-03-26

## Principle

Hardware-enforced boundaries between the AI agent and the host system.
Even a fully compromised agent cannot escape a VM or hardware isolation boundary.
This is the strongest protection layer because it is enforced by the hypervisor
or hardware, not by the application.

---

## Why This Is the Strongest Layer

| Property | Explanation |
|---|---|
| **Prevents all classes of host compromise** | File exfiltration, credential theft, and lateral movement are blocked at the hardware boundary |
| **Blast-radius containment** | If prompt injection succeeds, damage is confined to the isolated environment |
| **Trusted computing base** | Isolation is enforced by the hypervisor, not by the AI model or its tooling |
| **Ephemeral by default** | Environments can be destroyed after each session, eliminating persistence and implants |

Every inner layer (permissions, prompt hardening, output filtering) can fail.
Infrastructure isolation ensures that when they do, the attacker is still trapped
inside a disposable boundary with no path to production systems or developer machines.

---

## Approaches

### Remote Cloud Development Environments

#### GitHub Codespaces

- **Architecture**: Each codespace runs in a freshly provisioned VM, not a shared
  container. The VM is rebuilt from a devcontainer spec on every restart.
- **Isolation**: VM-level -- each codespace is isolated from others and receives
  the latest security patches on boot.
- **Agent integration**: Claude Code and Copilot run inside Codespaces with full
  terminal access.
- **Security model**: The agent operates on cloud-hosted code and never touches
  the developer's local machine.
- **Tradeoffs**: Latency to cloud, cost per compute-hour, requires internet
  connectivity, full OS overhead.
- **Best for**: Teams that need strong isolation and can tolerate a cloud
  development workflow.

#### Gitpod

- **Architecture**: Container-based (lighter weight than VM-based Codespaces).
- **Isolation**: Linux namespace isolation, but containers share the host kernel.
- **Tradeoffs**: Faster startup, lower cost per seat, but weaker isolation than
  VM-based approaches.
- **Caveat**: Container-based isolation has known escape vectors (shared kernel).
  Not recommended as a sole security boundary for untrusted agent workloads.

> **Key decision**: VM-based (Codespaces) vs. container-based (Gitpod).
> For security-critical workloads, VM isolation is significantly stronger.

---

### MicroVM Isolation

MicroVMs combine the security of full VMs with the speed and density of
containers. They are the preferred approach for high-throughput agent platforms
that need per-session isolation.

#### Firecracker (AWS)

- Purpose-built lightweight VM monitor (VMM).
- Boot time: **<125 ms**, memory overhead: **<5 MB** per microVM.
- Used in production by AWS Lambda and Fargate.
- Provides full kernel isolation without traditional VM overhead.
- Each agent session gets its own kernel, filesystem, and network namespace.

#### Kata Containers

- OCI-compatible container runtime backed by lightweight VMs.
- Pluggable hypervisor: Cloud Hypervisor, QEMU, or Firecracker.
- Each container runs in its own VM kernel.
- Compatible with Kubernetes via containerd -- drop-in replacement for runc in
  existing orchestration pipelines.

#### E2B (purpose-built for AI agents)

- Boots sandbox servers globally in **<200 ms**.
- Isolated kernels, filesystems, and network namespaces per session.
- Designed specifically for AI agent code execution use cases.
- Provides an API for programmatic sandbox lifecycle management.
- **Best for**: AI agent platforms that need rapid, disposable execution
  environments without managing hypervisor infrastructure.

---

### Container-Based Isolation

#### Docker Sandboxes for AI Agents

- Docker has published specific guidance for running coding agents in containers.
- **Architecture**: Each agent gets an isolated container with its own Docker
  daemon (Docker-in-Docker or sidecar pattern).
- Agents can spin up test containers and modify their environment without
  affecting the host.
- Volume mounts control which host directories are accessible.
- Network policy can be restricted or fully air-gapped.

#### DevContainers

- VS Code / IDE-native feature for containerized development.
- `.devcontainer/devcontainer.json` defines the environment declaratively.
- Process-level isolation, directory access limits, filesystem containment.
- AI agents run inside the dev container rather than on the host, limiting their
  reach to the mounted workspace.

> **Critical caveat**: Containers are NOT sandboxes in the security sense. They
> share the host kernel, and container escape vulnerabilities are regularly
> discovered. Use containers as a defense-in-depth layer, not as a sole security
> boundary.

---

## Implementation Guide

### Quick Start: Docker-Based Agent Isolation

The following example runs Claude Code in a locked-down container with no network
access and a read-only root filesystem. Only the project directory is mounted.

```bash
# Build or pull a base image with Claude Code installed
# (see Anthropic's sandbox-runtime repo for a reference Dockerfile)

# Run Claude Code in an isolated container
docker run -it --rm \
  --name claude-sandbox \
  --network=none \
  --read-only \
  --tmpfs /tmp:rw,noexec,size=512m \
  -v /path/to/project:/workspace:rw \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --memory=4g \
  --cpus=2 \
  claude-code-image
```

**What each flag does**:

| Flag | Purpose |
|---|---|
| `--network=none` | Air-gaps the container -- no outbound or inbound network access |
| `--read-only` | Root filesystem is immutable; agent cannot modify system binaries |
| `--tmpfs /tmp:rw,noexec` | Writable scratch space without execute permission |
| `-v ...:/workspace:rw` | Only the target project is mounted; nothing else on the host is visible |
| `--cap-drop=ALL` | Drops every Linux capability (no `ptrace`, no raw sockets, etc.) |
| `--security-opt=no-new-privileges` | Prevents privilege escalation via setuid binaries |
| `--memory` / `--cpus` | Resource limits prevent denial-of-service against the host |

### Production: MicroVM Setup with Firecracker

For production agent platforms that execute many concurrent sessions:

1. **Install Firecracker** on a bare-metal or nested-virtualization-enabled host.
2. **Prepare a root filesystem** image containing the agent toolchain (Claude Code,
   language runtimes, project dependencies).
3. **Launch one microVM per agent session** using the Firecracker API:
   - Assign a unique tap device for network isolation.
   - Mount the project code as a read-only block device (or copy it in).
   - Set resource limits (vCPUs, memory) per microVM.
4. **Destroy the microVM** when the session ends. No state persists.

Firecracker's sub-125 ms boot time makes per-request isolation feasible even for
interactive coding sessions.

### Enterprise: Codespaces with Organization Policies

GitHub Codespaces supports organization-level policy controls:

- **Allowed machine types**: Restrict to specific VM sizes to control cost and
  surface area.
- **Idle timeout**: Auto-stop codespaces after a configurable idle period (e.g.,
  30 minutes).
- **Retention period**: Auto-delete stopped codespaces after N days.
- **Secret management**: Org-scoped secrets are injected at runtime, never baked
  into the image.
- **Allowed repositories**: Limit which repos can spawn codespaces.
- **Network restrictions**: Use GitHub's IP allowlists or VPN integration to
  restrict codespace egress.

Configure these via **Organization Settings > Codespaces > Policies** in the
GitHub admin console.

---

## Limitations

| Limitation | Impact |
|---|---|
| **Latency** | Remote environments add network round-trip time to every file operation and terminal command |
| **Cost** | A VM or microVM per session is more expensive than bare local execution |
| **Offline unavailability** | Cloud-based solutions require internet connectivity |
| **Persistent storage** | Any mounted volume bridges the isolation boundary -- mount only what is needed, prefer read-only |
| **Configuration complexity** | MicroVM orchestration (Firecracker, Kata) is non-trivial to deploy and operate at scale |
| **Developer experience** | Some developers resist switching away from local development workflows |

---

## Tool-Specific Notes

### Claude Code

- Runs inside Docker containers, Codespaces, or any Linux environment.
- Anthropic's **sandbox-runtime** open-source project provides a reference
  container configuration purpose-built for agent isolation.
- GitHub: <https://github.com/anthropic-experimental/sandbox-runtime>

### GitHub Copilot CLI

- Designed primarily for local terminal use.
- Runs inside Codespaces natively (no extra configuration).
- No official container isolation guidance published by GitHub.

### Cursor

- Desktop application with DevContainer support.
- Can connect to remote containers via SSH Remote.
- No built-in microVM support; relies on external isolation if needed.

---

## Related CVEs

| CVE | Description | How Infrastructure Isolation Helps |
|---|---|---|
| **CVE-2025-59536** | Claude Code RCE via malicious project files | Malicious code executes inside the VM/container, not on the developer's host machine |
| **CVE-2025-54135 / 54136** | Cursor MCP server trust bypass | Infrastructure isolation prevents escalation from a compromised MCP server to the host OS |

In both cases, the attacker gains code execution -- but infrastructure isolation
ensures that execution is confined to a disposable, unprivileged environment with
no access to production credentials or the developer's local filesystem.

---

## References

- [GitHub Codespaces Security](https://docs.github.com/en/codespaces/reference/security-in-github-codespaces)
- [Docker Sandboxes for AI Agents](https://www.docker.com/blog/docker-sandboxes-run-claude-code-and-other-coding-agents-unsupervised-but-safely/)
- [Firecracker MicroVM](https://firecracker-microvm.github.io/)
- [E2B Sandboxes](https://e2b.dev/)
- [Kata Containers](https://katacontainers.io/)
- [Anthropic sandbox-runtime](https://github.com/anthropic-experimental/sandbox-runtime)
- [Northflank: Best Code Execution Sandbox for AI Agents](https://northflank.com/blog/best-code-execution-sandbox-for-ai-agents)
- [NVIDIA: Practical Security for Sandboxing Agentic Workflows](https://developer.nvidia.com/blog/practical-security-guidance-for-sandboxing-agentic-workflows-and-managing-execution-risk/)
