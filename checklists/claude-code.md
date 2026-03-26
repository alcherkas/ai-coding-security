# Claude Code Security Hardening Guide

> Last verified: 2026-03-26

Claude Code has the most mature security model among AI coding agents, with built-in sandboxing, mandatory approval, and MCP server isolation. This guide covers how to maximize its security features.

## Security Posture Summary

| Feature | Default | Recommended |
|---------|---------|-------------|
| OS Sandbox | Enabled (macOS/Linux) | Keep enabled |
| Permission prompts | All actions require approval | Keep default or use sandbox auto-allow |
| MCP server sandbox | Enabled | Keep enabled |
| Network restrictions | Within sandbox | Add external egress controls |
| Audit logging | Enterprise tier | Enable |
| Hooks | Enabled | Review and restrict |

## Sandbox Configuration

### Verify Sandbox is Active

Claude Code sandboxing is enabled by default on supported platforms:
- **macOS**: Uses Seatbelt (sandbox-exec with SBPL profile)
- **Linux/WSL2**: Uses bubblewrap + seccomp + Landlock

The sandbox restricts:
- Filesystem access to the project directory and required system paths
- Network access to approved Anthropic servers
- Process operations

### Sandbox + Auto-Allow Mode

The recommended mode for most developers:
- Operations within sandbox boundaries are auto-approved (84% fewer prompts)
- Operations requiring network or out-of-scope access fall back to manual approval
- Provides the best balance of security and productivity

## Permission Configuration

### Deny Rules (settings.json)

Add deny rules to block dangerous operations:

```json
{
  "permissions": {
    "deny": [
      "Bash(rm -rf *)",
      "Bash(curl * | bash)",
      "Bash(wget * | bash)",
      "Bash(git push --force*)",
      "Bash(git reset --hard*)",
      "Bash(chmod 777*)",
      "Bash(sudo *)",
      "Bash(ssh *)",
      "Bash(scp *)",
      "Bash(nc *)",
      "Bash(ncat *)",
      "Bash(python -m http.server*)",
      "Bash(python -c *import*socket*)"
    ]
  }
}
```

### Never Use --dangerously-skip-permissions Without Compensating Controls

This flag disables all permission checks. Only acceptable when ALL of these are true:
- [ ] Running inside an isolated container or VM (Layer 1)
- [ ] Network egress is blocked or restricted (Layer 3)
- [ ] No access to production credentials or sensitive data
- [ ] Comprehensive audit logging is enabled (Layer 10)
- [ ] Used in CI/CD automation, not interactive development

## MCP Server Security

### Default: Sandboxed

Claude Code runs MCP servers in sandboxed environments by default. This is the strongest default among major agents.

### Vetting New MCP Servers

Before enabling any MCP server:
1. Review the server's source code
2. Check for known vulnerabilities (CVEs, security advisories)
3. Review tool descriptions for suspicious instructions (tool poisoning)
4. Pin the version in your configuration
5. Test in an isolated environment first

### Recommended MCP Server Settings

- Only enable servers you actively use
- Remove unused servers
- Pin versions (don't auto-update)
- Monitor for tool description changes

## Hook Security

Hooks execute shell commands in response to agent events. They can be exploited:

### CVE-2025-59536: RCE via Hooks
Malicious project configuration files could define hooks that execute arbitrary code.

### Recommendations
- Review all hooks defined in project `.claude/` directories
- Be cautious with hooks in cloned/forked repositories
- Disable hooks for untrusted projects
- Use allowlist approach: only enable specific, reviewed hooks

## Network Security

### Built-In Controls
- Sandbox restricts network to approved Anthropic servers
- Commands needing network fall back to permission approval

### Additional Recommendations
- Run in a container with `--network=none` for offline work
- Use a transparent proxy for network inspection
- Block cloud metadata endpoints (169.254.169.254)
- Configure DNS filtering for the agent's environment

## File Protection

### .aiignore
Create a comprehensive `.aiignore` at your repository root:
```
# Secrets
.env
.env.*
*.pem
*.key
*.p12
credentials/
secrets/
.aws/
.ssh/
*.keystore

# Infrastructure
terraform.tfstate
*.tfvars
docker-compose.override.yml
*.tfstate.backup

# CI/CD
.github/workflows/
.gitlab-ci.yml
Jenkinsfile

# Build artifacts
dist/
build/
node_modules/
```

### Environment Scrubbing
Start Claude Code with a minimal environment:
```bash
#!/bin/bash
# claude-secure.sh — wrapper for Claude Code with env scrubbing
exec env -i \
  HOME="$HOME" \
  PATH="/usr/local/bin:/usr/bin:/bin" \
  TERM="$TERM" \
  LANG="$LANG" \
  SHELL="$SHELL" \
  ANTHROPIC_API_KEY="$ANTHROPIC_API_KEY" \
  claude "$@"
```

## Security Checklist

- [ ] Sandbox is enabled (verify on your platform)
- [ ] Deny rules configured in settings.json
- [ ] .aiignore created for sensitive files
- [ ] MCP servers reviewed and pinned
- [ ] Hooks reviewed for untrusted projects
- [ ] Environment variables scrubbed
- [ ] --dangerously-skip-permissions NOT used (or used with full compensating controls)
- [ ] Pre-commit hooks installed (gitleaks, detect-secrets)
- [ ] Audit logging enabled (enterprise) or local logging configured
- [ ] Network egress controls in place for sensitive projects

## Known CVEs

| CVE | Severity | Description | Status |
|-----|----------|-------------|--------|
| CVE-2025-59536 | High (8.7) | RCE via hooks, MCP, env in project files | Fixed |
| CVE-2026-21852 | Medium (5.3) | API token exfiltration via project files | Fixed |

Keep Claude Code updated to ensure you have the latest security patches.

## References

- Claude Code Security — https://code.claude.com/docs/en/security
- Claude Code Sandboxing — https://code.claude.com/docs/en/sandboxing
- Anthropic Engineering: Sandboxing — https://www.anthropic.com/engineering/claude-code-sandboxing
- Check Point: Claude Code CVEs — https://research.checkpoint.com/2026/rce-and-api-token-exfiltration-through-claude-code-project-files-cve-2025-59536/
- Backslash: Security Best Practices — https://www.backslash.security/blog/claude-code-security-best-practices
