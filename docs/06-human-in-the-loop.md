# Layer 6: Human-in-the-Loop Controls

> Last verified: 2026-03-26

## Principle

Require human approval for high-risk, destructive, or irreversible actions. AI agents should operate autonomously for routine, safe operations, but escalate to a human for decisions that could cause significant harm. The goal is to balance productivity (minimal approval friction) with safety (humans verify consequential actions).

## Why This Layer Matters

- Catches errors and attacks that automated controls miss
- Last opportunity to prevent destructive actions before they execute
- Prevents "excessive agency" — agents deciding to take broader actions than scoped
- Human judgment fills gaps where technical controls are ambiguous
- Social engineering and prompt injection ultimately target human trust — but humans can also detect things that seem "off"

## Approaches

### Permission Modes

**Claude Code permission model** (most mature):

| Mode | Behavior | Security | Productivity |
|------|----------|----------|--------------|
| Default | All file edits and commands require approval | Highest | Lowest |
| Auto-allow within sandbox | Operations inside sandbox auto-approved; outside requires approval | High | High |
| Bypass (--dangerously-skip-permissions) | No approval required | None | Highest |

Claude Code's sandbox mode achieves the best tradeoff: 84% fewer permission prompts while maintaining security boundaries. The "tell-don't-ask" model — define boundaries upfront, then operate freely within them.

**GitHub Copilot CLI**:
- Every file change and command requires explicit user approval
- No auto-approve mode
- Trust directories define the scope of allowed operations

**Cursor**:
- Auto-approve available (convenience over security)
- No granular permission levels
- Unsandboxed MCP servers by default

### Approval Workflow Patterns

**Pattern 1: Tiered approval**
```
Safe operations (read files, run tests)     → Auto-approve
Moderate operations (edit files, install deps) → Approve once per session
Dangerous operations (delete, force push, deploy) → Always require approval
```

**Pattern 2: Scope-based approval**
```
Within project directory → Auto-approve
Within home directory → Require approval
System directories → Block entirely
```

**Pattern 3: Command-based deny rules**
```json
// Claude Code settings.json
{
  "permissions": {
    "deny": [
      "Bash(rm -rf *)",
      "Bash(git push --force*)",
      "Bash(git reset --hard*)",
      "Bash(chmod 777*)",
      "Bash(curl * | bash)",
      "Bash(wget * | bash)",
      "Bash(sudo *)",
      "Bash(ssh *)",
      "Bash(scp *)"
    ]
  }
}
```

**Pattern 4: Confirmation gates for consequential actions**
- Before purchases, deployments, emails, or external API calls
- Before modifying CI/CD pipelines or infrastructure
- Before accessing production databases or services
- Before any action that affects shared state (push, PR creation, issue comments)

### Behavioral Monitoring

Beyond per-action approval, monitor aggregate agent behavior:

**Anomaly detection signals**:
- Unusual number of file modifications in a session
- Accessing files outside normal working patterns
- Rapid succession of shell commands
- Network requests to unexpected destinations
- Attempts to read credentials or sensitive files
- Tool invocation patterns that deviate from baseline

**Automated containment**:
- Pause agent and require human review when anomalies are detected
- Rate-limit operations that exceed normal patterns
- Terminate session on high-severity anomalies

### Approval Fatigue and Its Risks

**The paradox**: Too many approval prompts lead to rubber-stamping — humans approve everything without reading, defeating the purpose.

**Mitigations**:
- Use sandbox-based auto-approval to reduce prompt volume (Claude Code's 84% reduction)
- Group related approvals (approve a plan, not each step)
- Make approval prompts clear and concise — show exactly what will happen
- Highlight dangerous operations visually (red warnings, caps)
- Rotate approvers for critical operations
- Don't make approval the only control — combine with sandbox, network, and audit layers

## Implementation Guide

### Quick Start: Configure Safe Defaults

For Claude Code:
1. Enable sandbox mode (default on supported platforms)
2. Add deny rules for dangerous commands in settings.json
3. Review and approve initial operations to build trust
4. Let auto-allow handle routine operations within sandbox

For Copilot CLI:
1. Configure trust directories
2. Review every generated command before execution
3. Never pipe output directly to execution without review

### Production: Tiered Approval Policy

1. Define operation tiers (safe / moderate / dangerous)
2. Configure auto-approve for safe operations
3. Session-scoped approval for moderate operations
4. Always-require for dangerous operations
5. Log all approval decisions for audit

### Enterprise: Governance Integration

1. Role-based approval requirements (junior devs need more approval)
2. Integration with change management systems
3. Separation of duties (agent operator ≠ approver for critical operations)
4. Automated compliance reporting from approval logs

## The Anti-Pattern: --dangerously-skip-permissions

Claude Code includes a `--dangerously-skip-permissions` flag. Its name is intentional — it is dangerous.

**Why it exists**: Automated workflows (CI/CD) where no human is present to approve.

**When it's acceptable**:
- Inside a fully isolated container/VM (Layer 1)
- With network controls blocking exfiltration (Layer 3)
- With comprehensive audit logging (Layer 10)
- In non-production, non-sensitive environments

**When it's NOT acceptable**:
- On developer laptops with access to production credentials
- In environments with access to sensitive data
- Without compensating controls (sandbox + network + audit)
- "To save time" — this is how incidents happen

## Limitations

- **Approval fatigue**: Too many prompts → rubber-stamping → no real security
- **Social engineering**: Attackers can craft prompts that make malicious actions look benign
- **Speed vs. security**: Human approval adds latency; developers may disable controls for speed
- **Skill gap**: Not all developers can evaluate whether a command is safe
- **Context switching**: Approval prompts interrupt flow; developers may approve to return to work
- **Not scalable**: In automated/CI environments, human approval is impractical

## Tool-Specific Notes

### Claude Code
- Most sophisticated permission model among AI agents
- Sandbox + auto-allow reduces fatigue by 84%
- Deny rules for granular operation control
- Default: all operations require approval
- --dangerously-skip-permissions for automated contexts (use with compensating controls)

### GitHub Copilot CLI
- Every command requires approval (no auto-approve)
- Simple yes/no model
- Known CVE-2026-29783: shell expansion bypass could execute code without approval

### Cursor
- Auto-approve available (reduces friction but increases risk)
- No deny rules or granular permission control
- MCP servers unsandboxed by default — auto-approve compounds this risk

## Related CVEs

- CVE-2026-29783 (Copilot CLI shell expansion): Arbitrary code execution that bypasses approval
- CVE-2025-59536 (Claude Code hooks): Code execution through project files before user sees a prompt
- Snowflake Cortex Code CLI: Command validation bypass in shell substitution expressions

## References

- Claude Code Permissions: https://code.claude.com/docs/en/security
- Anthropic: Claude Code Sandboxing — https://www.anthropic.com/engineering/claude-code-sandboxing
- OWASP AI Agent Security Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html
- Backslash: Claude Code Security Best Practices — https://www.backslash.security/blog/claude-code-security-best-practices
