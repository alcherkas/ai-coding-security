# Cursor Security Hardening Guide

> Last verified: 2026-03-26

Cursor has the weakest security defaults among major AI coding agents. Multiple CVEs, unsandboxed MCP servers, "best effort" ignore files, disabled Workspace Trust, and the highest prompt injection success rate (83.4%) make hardening essential. This guide covers critical mitigations.

## Security Posture Summary

| Feature | Default | Recommended | Risk if Unchanged |
|---------|---------|-------------|-------------------|
| OS Sandbox (agent mode) | Available | Enable | High — no process isolation |
| MCP server sandbox | NOT enabled | Enable manually | Critical — CVE-2025-54135/54136 |
| .cursorignore | Best effort | Use, but don't rely solely | Medium — bypasses exist |
| Workspace Trust | Disabled | Enable immediately | Critical — silent code execution |
| Auto-approve | Available | Disable | High — compounds all other risks |
| .cursorrules validation | None | Review manually | High — direct injection vector |
| Chromium updates | Often outdated | Monitor/update | High — 94+ inherited CVEs |

## Critical: Enable Workspace Trust

**Cursor ships with VS Code's Workspace Trust disabled by default.** This means:
- Opening a folder with malicious `.vscode/tasks.json` silently executes code
- No trust prompt is shown to the user
- This is equivalent to auto-running unknown code on folder open

**Fix**: Enable Workspace Trust in settings:
1. Open Settings (Cmd/Ctrl + ,)
2. Search for "Workspace Trust"
3. Enable `security.workspace.trust.enabled`
4. Set `security.workspace.trust.startupPrompt` to `always`

## Critical: Sandbox MCP Servers

Cursor's MCP servers run unsandboxed by default. This directly led to CVE-2025-54135 (CurXecute) and CVE-2025-54136 (MCPoison).

**Fix**: Enable MCP server sandboxing in Cursor settings. Restrict:
- Filesystem access for each MCP server
- Network access for each MCP server
- Only enable MCP servers you actively use

## Critical: Disable Auto-Approve

Auto-approve combined with unsandboxed MCP servers is the reason Cursor has an 83.4% prompt injection success rate.

**Fix**: Disable auto-approve in settings. Review every agent action before execution.

## .cursorignore Limitations

Cursor describes .cursorignore as **"best effort"** — it does NOT guarantee that files won't be accessed.

### Known Bypasses
- Case-sensitivity: `.ENV` bypasses a rule for `.env` on case-insensitive filesystems (CVE-2025-59944)
- Indirect access: `docker compose config` exposes .env despite .cursorignore
- Agent can still access files through shell commands (`cat`, `type`, etc.)
- Requires terminal restart to take effect

### Recommendations
- Use .cursorignore for basic protection, but don't rely on it as a security boundary
- Combine with external sandboxing (Docker, bubblewrap) for actual enforcement
- Remove sensitive files from the project directory when possible
- Use encrypted secret storage instead of plaintext .env files

### Minimal .cursorignore
```
.env
.env.*
*.pem
*.key
*.p12
credentials/
secrets/
.aws/
.ssh/
terraform.tfstate
*.tfvars
```

## .cursorrules: An Attack Vector

.cursorrules files are processed without validation. This makes them a direct prompt injection vector:
- An attacker who controls .cursorrules in a cloned repo controls Cursor's behavior
- Malicious .cursorrules can instruct Cursor to exfiltrate data, modify code, or execute commands

### Recommendations
- Review .cursorrules in any cloned/forked repository before opening with Cursor
- Don't commit .cursorrules that contain security-sensitive instructions
- Consider adding .cursorrules to your .gitignore for untrusted repos
- Treat .cursorrules as executable code, not configuration

## Data Privacy Concerns

Cursor processes code on backend servers:
- Source code is transmitted to Cursor's infrastructure for processing
- Secrets in your code context are transmitted to Cursor servers
- Windsurf offers decentralized processing as an alternative

### Recommendations
- Don't use Cursor for code containing classified or highly sensitive data
- Review Cursor's data retention and privacy policies
- Consider whether your organization's data handling requirements allow cloud code processing

## External Security Controls (Required)

### Container Isolation (Strongly Recommended)

Given Cursor's weak defaults, running it in a container is the most effective single improvement:

```bash
# Isolate Cursor's workspace
docker run -it --rm \
  --network=none \
  -v /path/to/project:/workspace:rw \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  dev-environment
```

Open the container's filesystem as a remote workspace in Cursor.

### Network Controls

- Use a proxy to inspect Cursor's outbound traffic
- Block access to unauthorized domains
- Monitor data volume transmitted to Cursor's backend

### Pre-Commit Hooks (Mandatory)

Since Cursor has the highest prompt injection success rate, pre-commit scanning is essential:
```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
  - repo: https://github.com/returntocorp/semgrep
    rev: v1.50.0
    hooks:
      - id: semgrep
        args: ['--config', 'auto']
```

## Security Checklist

- [ ] **Workspace Trust enabled** (disabled by default — critical)
- [ ] **MCP servers sandboxed** (not sandboxed by default — critical)
- [ ] **Auto-approve disabled** (compounds all other vulnerabilities)
- [ ] .cursorignore created (best effort, but still helpful)
- [ ] .cursorrules reviewed in cloned repos
- [ ] External sandbox configured (Docker recommended)
- [ ] Network controls in place
- [ ] Pre-commit hooks installed
- [ ] Keep Cursor updated (94+ Chromium CVEs in outdated builds)
- [ ] Environment variables scrubbed
- [ ] Sensitive projects use alternative tools with stronger defaults

## Known CVEs

| CVE | Severity | Description | Status |
|-----|----------|-------------|--------|
| CVE-2025-59944 | High | Case-sensitivity file bypass for RCE | Fixed |
| CVE-2025-54135 | High | MCP trust bypass (CurXecute) — persistent RCE | Fixed |
| CVE-2025-54136 | High | MCP trust bypass (MCPoison) — tool description poisoning | Fixed |
| CVE-2025-61590 | High | RCE via prompt injection | Fixed |
| CVE-2025-61591 | High | RCE via OAuth impersonation | Fixed |
| CVE-2025-61592 | High | RCE via workspace manipulation | Fixed |
| CVE-2025-61593 | High | RCE via CLI configuration | Fixed |
| 94+ Chromium CVEs | Various | Inherited from outdated Electron/Chromium build | Varies |

## Recommendation: Consider Alternatives

For security-sensitive work, consider whether Cursor's convenience justifies its security risk profile. Claude Code and GitHub Copilot have significantly stronger security defaults. If you must use Cursor:
1. Implement ALL items on the checklist above
2. Run in a containerized environment
3. Never use auto-approve
4. Review every .cursorrules file in cloned repos
5. Keep Cursor aggressively updated

## References

- Cursor Agent Sandboxing — https://cursor.com/blog/agent-sandboxing
- Cursor Vulnerability CVE-2025-59944 — https://www.lakera.ai/blog/cursor-vulnerability-cve-2025-59944
- Cursor MCP Vulnerability — https://research.checkpoint.com/2025/cursor-vulnerability-mcpoison/
- 94 Vulnerabilities in Cursor and Windsurf — https://www.ox.security/blog/94-vulnerabilities-in-cursor-and-windsurf-put-1-8m-developers-at-risk/
- Cursor Sandboxing Leaks Secrets — https://luca-becker.me/blog/cursor-sandboxing-leaks-secrets/
- Knostic .env Leakage — https://www.knostic.ai/blog/claude-cursor-env-file-secret-leakage
