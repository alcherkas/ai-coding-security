# GitHub Copilot Security Hardening Guide

> Last verified: 2026-03-26

GitHub Copilot (including Copilot CLI) provides AI-assisted coding with enterprise-grade data policies but limited built-in sandboxing. This guide covers how to maximize security when using Copilot.

## Security Posture Summary

| Feature | Default | Recommended |
|---------|---------|-------------|
| OS Sandbox | None built-in | Add external sandbox |
| Permission prompts | Commands require approval | Keep enabled, review carefully |
| MCP server sandbox | Evolving | Monitor updates |
| Network restrictions | None built-in | Add external controls |
| Data retention (Business/Enterprise) | Prompts/suggestions NOT retained | Verify org policy |
| Public code matching | Configurable | Block matching (Business/Enterprise) |
| Vulnerability protection | Enabled | Keep enabled |

## Data Privacy Configuration

### Understand Your Tier

**Copilot Business & Enterprise** (via supported IDEs):
- Prompts and suggestions are NOT retained
- NOT used to train models
- User engagement data kept for 2 years
- Strongest privacy guarantees

**Copilot Free, Pro, Pro+** (starting April 24, 2026):
- Interaction data MAY be used to train/improve models
- Opt-out available in settings
- Review privacy settings immediately

### Recommended Policy Settings

For organizations:
- Set data retention to minimum
- Enable "Block suggestions matching public code"
- Disable model training on your data
- Configure which models are available to users
- Set feature availability per team/project

## Trust Directories

Copilot CLI uses trust directories to scope file access:

### Configuration
- Explicitly configure directories where Copilot can read and modify files
- Keep trust scope as narrow as possible (project directory, not home directory)
- Review trust configuration regularly

## Command Review

### Every Command Requires Review

Copilot CLI generates shell commands that require user approval before execution. This is a critical security control.

**Best practices**:
- Read the full command before approving (don't rubber-stamp)
- Check for:
  - Unexpected flags (--force, --hard, -rf)
  - Piped commands (`curl | bash`, `wget | sh`)
  - File paths outside your project
  - Network commands to unknown destinations
  - sudo/privilege escalation
- If unsure, reject and ask for an alternative

### CVE-2026-29783: Shell Expansion Bypass

**Critical vulnerability**: Shell expansion patterns can enable arbitrary code execution, bypassing approval.

**Impact**: Attacker-crafted prompts can generate commands where malicious code hides in shell substitution expressions that execute before the user sees the full expanded command.

**Mitigation**: Keep Copilot CLI updated. Review commands for shell substitution patterns ($(), ``, ${}). Be extra cautious with complex multi-part commands.

## Vulnerability Protection

Copilot includes real-time vulnerability protection that blocks insecure patterns:
- Hardcoded credentials in generated code
- SQL injection patterns
- Common vulnerability patterns

**Recommendation**: Keep this enabled. It provides a baseline safety net.

## External Security Controls (Required)

Since Copilot lacks built-in sandboxing:

### OS-Level Sandboxing
Run Copilot CLI inside a sandbox:
```bash
# Using bubblewrap on Linux
bwrap \
  --ro-bind /usr /usr \
  --ro-bind /bin /bin \
  --ro-bind /lib /lib \
  --bind /path/to/project /workspace \
  --tmpfs /tmp \
  --unshare-net \
  --die-with-parent \
  -- copilot-cli
```

### Container Isolation
```bash
# Run in Docker with network restricted
docker run -it --rm \
  --network=none \
  -v /path/to/project:/workspace \
  copilot-image
```

### Environment Scrubbing
```bash
# Don't inherit sensitive environment variables
env -i HOME=$HOME PATH=$PATH TERM=$TERM GITHUB_TOKEN=$GITHUB_TOKEN copilot
```

### Pre-Commit Hooks
Mandatory for Copilot users since there's no built-in code scanning:
```yaml
# .pre-commit-config.yaml
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

- [ ] Understand your data retention tier (Business/Enterprise vs. Free/Pro)
- [ ] Block suggestions matching public code (Business/Enterprise)
- [ ] Trust directories configured to minimum scope
- [ ] Vulnerability protection enabled
- [ ] Review every generated command before execution
- [ ] External sandbox configured (bubblewrap, Docker, or VM)
- [ ] Environment variables scrubbed
- [ ] Pre-commit hooks installed (gitleaks, semgrep)
- [ ] Keep Copilot CLI updated (CVE patches)
- [ ] Organization policies configured (if applicable)

## Known CVEs

| CVE | Severity | Description | Status |
|-----|----------|-------------|--------|
| CVE-2025-53773 | — | Security vulnerability (patched) | Fixed Nov 2025 |
| CVE-2025-64660 | — | Security vulnerability (patched) | Fixed Nov 2025 |
| CVE-2025-62449 | Medium (6.8) | Path traversal — unauthorized file access | Fixed |
| CVE-2025-62453 | Medium (5.0) | Improper output validation — code injection | Fixed |
| CVE-2026-29783 | — | Shell expansion arbitrary code execution | Fixed |

## References

- GitHub Copilot Policies — https://docs.github.com/en/copilot/concepts/policies
- Managing Copilot Policies — https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-organization/manage-policies
- GitGuardian: Copilot Privacy — https://blog.gitguardian.com/github-copilot-security-and-privacy/
- GitHub Copilot Trust Center — https://copilot.github.trust.page/faq
- Demystifying Copilot Security Controls — https://techcommunity.microsoft.com/blog/azuredevcommunityblog/demystifying-github-copilot-security-controls-easing-concerns-for-organizational/4468193
