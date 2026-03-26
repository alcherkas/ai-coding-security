# Layer 3: Network Controls

> Last verified: 2026-03-26

## Principle

Restrict and monitor all network communication by AI coding agents. Even if an agent is compromised (via prompt injection or tool poisoning), network controls prevent data exfiltration, command-and-control communication, and unauthorized API calls. This layer works even if the agent escapes its OS sandbox — without network access, stolen data has nowhere to go.

## Why This Layer Matters

- **Data exfiltration prevention**: The primary goal of many attacks is to steal source code, credentials, or proprietary data. Network controls are the last line of defense before data leaves the environment.
- **C2 blocking**: Prevents compromised agents from receiving instructions from attacker infrastructure.
- **SSRF prevention**: Stops agents from making requests to internal services or metadata endpoints (e.g., cloud instance metadata at 169.254.169.254).
- **Independent of model behavior**: Network restrictions are enforced by infrastructure, not by the AI model's compliance with instructions.

## Approaches

### Egress Filtering

**Host-Level Firewall Rules**:
- Block all outbound connections by default
- Allowlist only required endpoints:
  - AI provider APIs (api.anthropic.com, api.github.com, etc.)
  - Package registries (registry.npmjs.org, pypi.org)
  - Version control (github.com)
- Block cloud metadata endpoints (169.254.169.254)
- Can be implemented with:
  - macOS: `pfctl` (Packet Filter)
  - Linux: `iptables` / `nftables`
  - Both: Application-level firewalls (Little Snitch, OpenSnitch)

**Example iptables rules for AI agent isolation**:
```bash
# Create a chain for AI agent traffic
iptables -N AI_AGENT

# Allow DNS resolution
iptables -A AI_AGENT -p udp --dport 53 -j ACCEPT
iptables -A AI_AGENT -p tcp --dport 53 -j ACCEPT

# Allow Anthropic API
iptables -A AI_AGENT -d api.anthropic.com -p tcp --dport 443 -j ACCEPT

# Allow GitHub
iptables -A AI_AGENT -d github.com -p tcp --dport 443 -j ACCEPT

# Block cloud metadata
iptables -A AI_AGENT -d 169.254.169.254 -j DROP

# Block everything else
iptables -A AI_AGENT -j DROP

# Apply to the agent's user or cgroup
iptables -A OUTPUT -m owner --uid-owner ai-agent -j AI_AGENT
```

**IP/CIDR Allowlisting**:
- Resolve provider domains to IP ranges and allowlist at the IP level
- More resilient than domain-based filtering (DNS can be manipulated)
- Requires maintenance as provider IPs change
- Consider automating IP range updates via provider status pages or API

### Agent Proxy Architecture

**Transparent Proxy Pattern**:
The most robust approach — all agent network traffic is forced through a proxy that inspects, logs, and controls requests.

Architecture:
```
AI Agent → [No direct internet] → Transparent Proxy → Internet
                                      │
                                      ├── Allowlist check
                                      ├── DLP scanning
                                      ├── SSRF detection
                                      ├── Request logging
                                      └── Rate limiting
```

**Key properties**:
- Agent cannot bypass proxy (network access is blocked; only proxy can reach internet)
- All requests are inspectable and auditable
- Can perform content-level inspection (DLP) before data leaves
- Can inject authentication headers for approved services
- Can strip sensitive data from outbound requests

**DLP patterns to detect in outbound traffic**:
- AWS access keys (`AKIA[0-9A-Z]{16}`)
- Private keys (`-----BEGIN .* PRIVATE KEY-----`)
- GitHub tokens (`ghp_`, `gho_`, `ghs_` prefixes)
- Database connection strings
- High-entropy strings that may be secrets
- Large code payloads sent to unauthorized endpoints

### Pipelock — Purpose-Built Agent Firewall

Pipelock is a firewall specifically designed for AI agents. Key features:
- **DLP scanning**: Detects sensitive data (credentials, PII) in outbound requests before DNS resolution
- **SSRF protection**: Blocks requests to internal/private IP ranges
- **MCP scanning**: Inspects MCP protocol traffic for malicious payloads
- **Prompt injection blocking**: Detects prompt injection attempts in network traffic
- **Allowlist enforcement**: Domain and URL path-level allowlisting
- GitHub: https://github.com/nicholasgasior/pipelock (reference the actual project if URL is different)

**Key insight from Pipelock's architecture**: DLP scanning runs before DNS resolution. This means even if an attacker tricks the agent into sending data to a domain that resolves to an internal IP, the DLP check catches the sensitive content first.

### Network Policies in Kubernetes/Container Environments

For containerized agents:
- **Cilium**: eBPF-based network policies with L7 visibility
- **Calico**: Network policy enforcement for Kubernetes
- **NetworkPolicy resources**: Kubernetes-native egress/ingress rules

```yaml
# Example: Restrict AI agent pod to only reach specific endpoints
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ai-agent-egress
spec:
  podSelector:
    matchLabels:
      app: ai-agent
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.0.0/16  # Block metadata
              - 10.0.0.0/8      # Block internal
              - 172.16.0.0/12   # Block internal
              - 192.168.0.0/16  # Block internal
      ports:
        - protocol: TCP
          port: 443
```

### DNS-Level Controls

- **Pi-hole / CoreDNS filtering**: Block resolution of unauthorized domains
- **DNS-over-HTTPS blocking**: Prevent agents from bypassing DNS filters via DoH/DoT
- **Split DNS**: Internal DNS only resolves approved domains for agent environments
- **DNS logging**: Monitor all DNS queries from agent processes for anomaly detection
- **Response policy zones (RPZ)**: Return NXDOMAIN for blocked domains rather than silently dropping, which produces clearer error messages for debugging

## Implementation Guide

### Quick Start: Docker with No Network

The simplest and most effective network control — run the agent with no network at all:

```bash
docker run --network=none -v /project:/workspace agent-image
```

This is appropriate when the agent only needs to read/write local files. Git operations can be performed outside the container.

For agents that need limited network access, create a custom network with restrictions:

```bash
# Create an isolated network with no external access
docker network create --internal agent-net

# Run the agent on the isolated network
docker run --network=agent-net -v /project:/workspace agent-image
```

### Production: Transparent Proxy with mitmproxy

Set up mitmproxy as a transparent proxy for agent traffic with TLS inspection and allowlisting:

1. **Install mitmproxy** on the proxy host (separate from the agent environment)
2. **Generate a CA certificate** and install it in the agent's trust store
3. **Configure iptables** to redirect agent traffic through the proxy:
```bash
# Redirect HTTP/HTTPS traffic from agent user to mitmproxy
iptables -t nat -A OUTPUT -m owner --uid-owner ai-agent \
  -p tcp --dport 443 -j REDIRECT --to-port 8080
```
4. **Write a mitmproxy addon** that enforces the domain allowlist and logs all requests
5. **Monitor the proxy logs** for anomalous patterns (large payloads, unusual destinations, encoded data)

### Enterprise: eBPF-Based Enforcement

For Kubernetes environments, Cilium provides fine-grained network policy enforcement:

1. **Deploy Cilium** as the CNI plugin for the cluster
2. **Define CiliumNetworkPolicy** resources with L7 rules (HTTP method, path, header matching)
3. **Enable Hubble** for network flow observability
4. **Set up alerts** for policy violations from agent pods

eBPF-based enforcement has lower overhead than proxy-based approaches because policies are enforced in-kernel, avoiding userspace context switches for each packet.

## Monitoring and Alerting

Effective network controls require ongoing monitoring:

- **Connection volume**: Alert on unusual spikes in outbound connections from agent processes
- **Data volume**: Track bytes sent per destination; flag large transfers to any single endpoint
- **New destinations**: Alert when an agent connects to a domain or IP not seen before
- **Failed connection attempts**: High rates of blocked connections may indicate a compromised agent probing for exfiltration paths
- **DNS query patterns**: Watch for high-entropy subdomain queries (potential DNS tunneling)
- **Time-of-day anomalies**: Agent network activity outside expected hours may indicate compromise

## Limitations

- **TLS inspection complexity**: Inspecting HTTPS traffic requires TLS termination at the proxy, adding certificate management overhead
- **Latency**: Proxy-based approaches add latency to every network request
- **DNS manipulation**: If DNS is not controlled, agents may use DNS tunneling for exfiltration
- **Allowlist maintenance**: Provider IPs and domains change; allowlists require ongoing maintenance
- **WebSocket/streaming**: Some AI agent protocols use persistent connections that complicate proxy inspection
- **Encrypted exfiltration**: Data can be encoded/encrypted before exfiltration, making DLP less effective
- **Steganography**: Sophisticated attacks could hide data in legitimate-looking requests (e.g., within code comments sent to an allowed API)

## Tool-Specific Notes

### Claude Code
- Sandbox restricts network to approved Anthropic servers
- Commands needing network access fall back to permission approval flow
- Can be combined with external proxy for additional control
- MCP server connections may require additional allowlist entries

### GitHub Copilot CLI
- Requires network access to GitHub API
- No built-in network restrictions
- Must be controlled via external firewall rules
- Telemetry endpoints should be evaluated against data policy

### Cursor
- Processes code on backend servers (data leaves the local machine by design)
- No egress controls built-in
- Windsurf supports decentralized processing as an alternative
- Consider data classification before sending code to external servers

## Related CVEs

- Multiple data exfiltration CVEs across platforms would be mitigated by network controls
- SSRF vectors (cloud metadata access) blocked by IP-based egress filtering
- Prompt injection attacks that attempt exfiltration via HTTP requests are contained
- DNS rebinding attacks used to access internal services are blocked by IP-range egress rules

## References

- Claude Code Security: https://code.claude.com/docs/en/security
- Pipelock (Agent Firewall): reference URL
- Cilium: https://cilium.io/
- NVIDIA Practical Security for Sandboxing: https://developer.nvidia.com/blog/practical-security-guidance-for-sandboxing-agentic-workflows-and-managing-execution-risk/
- mitmproxy: https://mitmproxy.org/
- Docker Networking: https://docs.docker.com/engine/network/
- Kubernetes Network Policies: https://kubernetes.io/docs/concepts/services-networking/network-policies/
