# Layer 4: Filesystem & Permissions

> Last verified: 2026-03-26

## Principle

Restrict which files and directories an AI coding agent can read, write, and execute. Apply the principle of least privilege — the agent should only access what it needs for the current task. This layer is enforced at the application level through configuration files, allowlists, and tool capability scoping.

## Why This Layer Matters

- Prevents agents from reading sensitive files (credentials, SSH keys, production configs)
- Limits blast radius of compromised agents to a specific directory scope
- First line of defense that doesn't require infrastructure changes (easy to deploy)
- Complements OS sandboxing (Layer 2) with application-level granularity
- Essential even with sandbox — some agents have known sandbox bypasses

## Approaches

### Ignore Files (.aiignore, .cursorignore, .codeiumignore)

These files tell AI agents which files/directories to exclude from their context and operations.

**Claude Code — .aiignore**:
- Syntax follows .gitignore patterns
- Files matched are excluded from Claude's context and file operations
- Place at repository root or in subdirectories
- Example:
```
# Secrets and credentials
.env
.env.*
*.pem
*.key
credentials/
secrets/

# Infrastructure
terraform.tfstate
*.tfvars
docker-compose.override.yml

# CI/CD (prevent modification of pipelines)
.github/workflows/
.gitlab-ci.yml
Jenkinsfile

# Build artifacts
dist/
build/
node_modules/
```

**Cursor — .cursorignore**:
- Similar .gitignore syntax
- CRITICAL CAVEAT: Cursor describes .cursorignore as "best effort" — it is NOT guaranteed to prevent access
- Requires terminal restart to take effect
- Known issues: .env contents can still leak despite being in .cursorignore
- Source: https://luca-becker.me/blog/cursor-sandboxing-leaks-secrets/

**Codeium / Windsurf — .codeiumignore**:
- Similar pattern-based ignore file
- Behavior varies by tool version

**Key limitation**: Ignore files are advisory in many tools, not enforcement boundaries. They reduce accidental exposure but should not be relied upon as a sole security control.

### Filesystem Allowlisting

Rather than blacklisting sensitive files, explicitly whitelist which directories the agent can access.

**Approaches**:
- Claude Code sandbox: Restricts filesystem access to the project directory and required system paths
- Docker volume mounts: Only mount the specific project directory, not the home directory
- OS sandbox (Layer 2): Seatbelt/Landlock can enforce at kernel level

**Example — restrictive volume mount**:
```bash
# GOOD: Mount only the project
docker run -v /home/dev/project:/workspace agent

# BAD: Mount the entire home directory
docker run -v /home/dev:/home/dev agent
```

### Permission Models

**Conservative defaults (Claude Code)**:
- All file modifications require explicit user approval
- All shell commands require explicit user approval
- Auto-allow mode: operations within sandbox boundaries auto-approved
- Operations outside sandbox fall back to manual approval

**GitHub Copilot CLI**:
- Trust directories: explicitly configured directories where Copilot can operate
- Tool permissions: control which tools the CLI can use
- Every file change and command requires explicit user approval

**Cursor**:
- Auto-approve available (less secure than manual approval)
- .cursorrules processed without validation (attack vector)
- Workspace Trust disabled by default (see CVE section)

### Tool Capability Scoping (Least Privilege)

Beyond file access, restrict what operations the agent can perform:

**Principle**: Grant the minimum data, tool, and action rights needed for the task.

**Approaches**:
- **Deny rules**: Explicitly block dangerous operations
  ```json
  // Claude Code settings.json example
  {
    "permissions": {
      "deny": [
        "Bash(rm -rf *)",
        "Bash(curl*)",
        "Bash(wget*)",
        "Bash(git push --force*)"
      ]
    }
  }
  ```
- **Tool restrictions**: Limit which tools/commands the agent can invoke
- **Read-only mode**: Some tasks only need code reading, not writing
- **Time-scoped access**: Grant permissions for a session, not permanently

**FINOS Agent Authority Least Privilege Framework**:
- Granular access controls with dynamic privilege management
- Contextual access restrictions based on task scope
- Key difference from traditional RBAC: AI agents make runtime decisions, so permissions must be dynamically scoped
- Source: https://air-governance-framework.finos.org/mitigations/mi-18_agent-authority-least-privilege-framework.html

## Implementation Guide

### Quick Start: Create .aiignore

Step-by-step for creating an effective ignore file for your project.

1. Start with the template above (secrets, infrastructure, CI/CD, build artifacts)
2. Add project-specific sensitive paths (e.g., `config/production.yml`)
3. Commit the .aiignore to version control so all team members inherit it
4. Verify coverage: run `git ls-files` and confirm no sensitive files are missing from the ignore list
5. Test by asking the agent to read an ignored file — confirm it refuses or omits the content

### Production: Layered Permission Strategy

1. Start with deny-by-default
2. Add allowlists for specific directories
3. Configure deny rules for dangerous operations
4. Enable auto-allow only within sandbox boundaries
5. Audit permission grants regularly

### Enterprise: Centralized Policy

- Distribute .aiignore templates via repository templates
- Enforce via pre-commit hooks that check for .aiignore presence
- Central policy management for tool permissions across teams

## Limitations

- **.cursorignore is "best effort"**: Not a security boundary — files can still be accessed
- **Ignore files don't prevent indirect access**: Agent can run `cat .env` or `docker compose config` to read ignored files
- **Symlink bypasses**: Symlinks can point outside the allowed directory tree
- **Dynamic file creation**: Agent can create files that bypass static ignore patterns
- **Configuration drift**: Ignore files may not be updated when new sensitive files are added
- **No runtime enforcement**: Most ignore files are checked at context-loading time, not at every file operation

## Tool-Specific Notes

### Claude Code
- .aiignore with .gitignore syntax
- Sandbox enforces filesystem restrictions at OS level (stronger than ignore files)
- Permission deny rules in settings.json for operation-level control
- Recommended: Use both .aiignore AND sandbox for defense-in-depth

### GitHub Copilot CLI
- Trust directories configuration
- Every file change requires approval
- No ignore file mechanism — relies on trust boundaries

### Cursor
- .cursorignore (best effort, not guaranteed)
- .cursorrules is an attack vector — processed without validation
- Workspace Trust disabled by default — enable it
- Known secret leakage through sandbox bypasses

## Related CVEs

- CVE-2025-59944 (Cursor case-sensitivity bypass): Security checks on file paths didn't match filesystem behavior, allowing access to "ignored" files via case variation
- CVE-2025-62449 (Copilot path traversal): Path construction allowed reading files outside the trust boundary
- CVE-2025-59536 (Claude Code): Project config files (.claude/) executed before permission checks

## References

- Claude Code Permissions: https://code.claude.com/docs/en/security
- Cursor .cursorignore: https://docs.cursor.com/context/ignore-files
- Cursor Secret Leakage: https://luca-becker.me/blog/cursor-sandboxing-leaks-secrets/
- Knostic .env Leakage: https://www.knostic.ai/blog/claude-cursor-env-file-secret-leakage
- FINOS Agent Authority Framework: https://air-governance-framework.finos.org/mitigations/mi-18_agent-authority-least-privilege-framework.html
- Strata.io Rethinking Least Privilege: https://www.strata.io/blog/why-agentic-ai-forces-a-rethink-of-least-privilege/
