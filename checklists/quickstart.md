# Quick Start: Minimum Viable Security for AI Coding Agents

> These 10 controls provide meaningful protection regardless of which AI coding agent you use. Start here, then move to the [tool-specific guides](../README.md#tool-specific-hardening-guides) and [full layer documentation](../README.md#defense-in-depth-layer-model).

## The 10 Essential Controls

### 1. Enable OS-Level Sandboxing
**Layer**: [2 — OS Sandboxing](../docs/02-os-sandboxing.md)
**Why**: Limits what a compromised agent can do on your system (file access, syscalls, process operations).
**How**:
- **Claude Code**: Enabled by default on macOS (Seatbelt) and Linux (bubblewrap). Verify with `claude config` settings.
- **Cursor**: Enable agent sandboxing in settings (not default).
- **Copilot CLI**: No built-in sandbox — use Docker or bubblewrap externally.

### 2. Create .aiignore for Sensitive Files
**Layer**: [4 — Filesystem & Permissions](../docs/04-filesystem-permissions.md)
**Why**: Prevents the agent from reading secrets, credentials, and infrastructure configs.
**How**: Create `.aiignore` (Claude Code) or `.cursorignore` (Cursor) at your repo root:
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
**Caveat**: .cursorignore is "best effort" — not a guaranteed security boundary.

### 3. Scrub Environment Variables
**Layer**: [5 — Secrets Management](../docs/05-secrets-credentials.md)
**Why**: Agents inherit your shell environment including AWS keys, API tokens, and database credentials.
**How**: Start agents with a clean environment:
```bash
env -i HOME=$HOME PATH=$PATH TERM=$TERM ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY claude
```

### 4. Block Dangerous Commands
**Layer**: [6 — Human-in-the-Loop](../docs/06-human-in-the-loop.md)
**Why**: Prevents the agent from executing destructive or exfiltration commands.
**How** (Claude Code settings.json):
```json
{
  "permissions": {
    "deny": [
      "Bash(rm -rf *)",
      "Bash(curl * | bash)",
      "Bash(wget * | bash)",
      "Bash(git push --force*)",
      "Bash(sudo *)"
    ]
  }
}
```

### 5. Don't Open Untrusted Repos with AI Agents
**Layer**: [7 — Prompt Injection](../docs/07-prompt-injection.md)
**Why**: Malicious repos can contain prompt injection in code comments, READMEs, config files, and commit messages. Opening them with an AI agent gives the injection direct access.
**How**: Before opening a repo with an AI agent:
- Check the repo's reputation (stars, age, maintainer history)
- Review .claude/, .cursor/, .vscode/ directories for suspicious configurations
- If you must work with untrusted code, do it in a container (Layer 1)

### 6. Review AI-Generated Dependencies
**Layer**: [8 — Supply Chain](../docs/08-supply-chain.md)
**Why**: ~20% of AI-generated code references hallucinated packages. Attackers register these names.
**How**: Before running `npm install` or `pip install` on AI-generated code:
- Verify the package exists on the registry
- Check download count and age (new + low downloads = suspicious)
- Review `package.json` / `requirements.txt` diffs carefully
- Commit lockfiles and review lockfile changes

### 7. Pin MCP Server Versions
**Layer**: [9 — MCP Security](../docs/09-mcp-security.md)
**Why**: MCP tool poisoning has 72%+ attack success rate. Unpinned servers can be updated with malicious tool descriptions.
**How**:
- Only install MCP servers from trusted sources
- Pin versions in configuration (don't use `latest`)
- Review tool descriptions before enabling new servers
- Claude Code: MCP servers are sandboxed by default
- Cursor: MCP servers are NOT sandboxed by default — configure sandboxing

### 8. Run Secret Detection in Pre-Commit Hooks
**Layer**: [8 — Supply Chain](../docs/08-supply-chain.md)
**Why**: AI agents may generate code containing hardcoded credentials.
**How**:
```bash
# Install gitleaks
brew install gitleaks  # macOS
# or: apt install gitleaks  # Linux

# Add to .pre-commit-config.yaml
cat >> .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
EOF

pre-commit install
```

### 9. Use Short-Lived Credentials
**Layer**: [5 — Secrets Management](../docs/05-secrets-credentials.md)
**Why**: If an agent leaks a credential, short-lived tokens limit the window of exposure.
**How**:
- Use AWS STS temporary credentials instead of long-lived access keys
- Use GitHub App installation tokens (1-hour expiry) instead of PATs
- Use OAuth with short-lived access tokens
- Rotate any long-lived credentials regularly

### 10. Enable Audit Logging
**Layer**: [10 — Audit & Observability](../docs/10-audit-observability.md)
**Why**: When something goes wrong, logs are the only way to understand what happened.
**How**:
- Claude Code: Enterprise tier includes SOC 2-aligned audit logging
- Log agent sessions locally: redirect output to a timestamped log file
- Review logs after each significant session
- Forward to SIEM for correlation with other security events

## What's Next?

After implementing these 10 controls:
1. Read the [Threat Landscape](../docs/00-threat-landscape.md) to understand what you're defending against
2. Follow the tool-specific hardening guide for your agent:
   - [Claude Code](claude-code.md)
   - [GitHub Copilot](github-copilot.md)
   - [Cursor](cursor.md)
3. Evaluate [Infrastructure Isolation](../docs/01-infrastructure-isolation.md) for your most sensitive projects
4. Review the [full layer model](../README.md#defense-in-depth-layer-model) and implement additional layers based on your risk profile
