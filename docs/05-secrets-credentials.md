# Layer 5: Secrets & Credential Management

> Last verified: 2026-03-26

## Principle

Prevent AI coding agents from accessing, leaking, or exfiltrating secrets and credentials. This means: don't inherit the full host environment, use short-lived credentials, inject secrets only when needed, and assume that any secret accessible to the agent process WILL be read.

## Why This Layer Matters

- AI agents run as the developer's user — they inherit SSH keys, API tokens, cloud credentials, and environment variables
- Even sandboxed agents can access secrets through indirect channels (e.g., running `docker compose config` to extract .env values)
- Credential theft is the highest-value target for attackers — a single leaked API key can compromise entire systems
- Unlike other attack types, credential theft has immediate, real-world financial impact
- Environment variables are a major blind spot — most security guidance focuses on file access, not env vars

## The Problem: Inherited Trust

When you run an AI agent in your terminal, it typically inherits:

```
YOUR SHELL ENVIRONMENT
├── AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY
├── GITHUB_TOKEN
├── OPENAI_API_KEY / ANTHROPIC_API_KEY
├── DATABASE_URL (with embedded credentials)
├── ~/.ssh/id_rsa (SSH private key)
├── ~/.aws/credentials
├── ~/.kube/config
├── ~/.npmrc (with auth tokens)
├── ~/.docker/config.json (with registry credentials)
└── ... every other credential in your environment
```

The agent doesn't need most of these. But by default, it has access to all of them.

## Known Bypass Vectors

### The `docker compose config` Bypass
- Scenario: `.env` file is in `.aiignore` / `.cursorignore` — agent "can't" read it
- Bypass: Agent runs `docker compose config` which outputs the fully resolved configuration with all environment variables interpolated from `.env`
- Result: All secrets in `.env` are exposed despite the ignore rule
- Source: https://www.knostic.ai/blog/claude-cursor-env-file-secret-leakage

### Cursor Sandbox SSH Key Access
- Cursor's sandbox was found to allow access to `~/.ssh/id_rsa` despite filesystem restrictions
- The SSH private key is one of the most sensitive credentials on a developer's machine
- Source: https://luca-becker.me/blog/cursor-sandboxing-leaks-secrets/

### Environment Variable Inheritance
- Most agents inherit the full shell environment
- `env` or `printenv` commands expose all environment variables
- Even if direct file access is blocked, env vars often contain the same secrets
- Base64-encoded secrets in env vars bypass simple pattern matching

### Process Environment Inspection
- On Linux, `/proc/self/environ` contains all environment variables
- An agent with read access to `/proc` can extract credentials
- Sandbox must explicitly block `/proc` access

### Terraform / IaC State File Access
- `terraform output` and `terraform show` can reveal secrets stored in state
- State files (`.tfstate`) often contain database passwords, API keys, and private keys in plain text
- An agent asked to "debug the infrastructure" may legitimately read state files — exposing every secret in the stack
- Mitigation: Use remote state backends with access controls; never store `.tfstate` in the project directory

### Git History Mining
- `git log -p` or `git show` can reveal secrets that were committed and later removed
- Agents with git access can search commit history for patterns like `API_KEY=` or `password:`
- Even if the current working tree is clean, the full history may contain leaked credentials
- Mitigation: Use tools like `git-secrets` or `trufflehog` to scan and clean history

### Shell History and RC Files
- `~/.bash_history` / `~/.zsh_history` may contain inline credentials from past commands
- `~/.bashrc`, `~/.zshrc`, `~/.profile` may export secrets
- An agent that reads shell configuration to "understand the environment" gets secrets as a side effect
- Mitigation: Exclude home directory dotfiles from agent file access

## Approaches

### Environment Variable Scrubbing

**Don't inherit the full host environment**:

```bash
# BAD: Agent inherits everything
claude-code

# BETTER: Start with clean environment, add only what's needed
env -i HOME=$HOME PATH=$PATH TERM=$TERM claude-code

# BEST: Use a wrapper that explicitly sets allowed vars
#!/bin/bash
exec env -i \
  HOME="$HOME" \
  PATH="/usr/local/bin:/usr/bin:/bin" \
  TERM="$TERM" \
  LANG="$LANG" \
  ANTHROPIC_API_KEY="$ANTHROPIC_API_KEY" \
  claude-code "$@"
```

**Docker-based scrubbing**:
```bash
# Only pass the specific env vars the agent needs
docker run --rm \
  -e ANTHROPIC_API_KEY \
  -v /project:/workspace \
  agent-image
```

### Credential Brokers with Short-Lived Tokens

Instead of giving the agent long-lived credentials:

**Pattern**: Credential Broker → Short-lived Token → Agent

**Approaches**:
- **AWS STS**: Temporary credentials with session tokens (default 1 hour)
- **Vault (HashiCorp)**: Dynamic secrets with TTL
- **GitHub App tokens**: Installation tokens expire after 1 hour
- **OAuth with short-lived access tokens**: Tokens expire, refresh tokens stored securely outside agent scope

**Example — AWS temporary credentials for agent**:
```bash
# Generate temporary credentials scoped to the specific task
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::123456789:role/ai-agent-read-only \
  --role-session-name "claude-code-session" \
  --duration-seconds 3600)

# Pass only temporary credentials to the agent
docker run --rm \
  -e AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r '.Credentials.AccessKeyId') \
  -e AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r '.Credentials.SecretAccessKey') \
  -e AWS_SESSION_TOKEN=$(echo $CREDS | jq -r '.Credentials.SessionToken') \
  agent-image
```

### Secret Injection On Demand

Rather than pre-loading all credentials:
- Use a secrets manager that the agent can query for specific secrets as needed
- Each secret access is logged and auditable
- Access can be revoked mid-session if anomalous behavior is detected
- Secrets are never stored in the agent's environment — they're fetched, used, and discarded

### File-Based Secret Protection

Beyond ignore files (which can be bypassed):
- Remove sensitive files from the project directory entirely
- Use symlinks that point outside the sandbox boundary (sandbox blocks the dereference)
- Store secrets in encrypted vaults, not plain text files
- Use `git-crypt` or `sops` for encrypted secrets in repos — agent can't read without the key

### Secret Detection in Agent Output

Monitor for secrets in the agent's generated output:

```bash
# Example: scan agent-generated code for hardcoded secrets before committing
# Use trufflehog, gitleaks, or detect-secrets as a pre-commit hook
detect-secrets scan --baseline .secrets.baseline
gitleaks detect --source . --verbose
```

Key patterns to watch for:
- Agent copies a secret from env/config into source code "for testing"
- Agent hardcodes a connection string with embedded credentials
- Agent includes API keys in generated documentation or comments
- Agent writes secrets to log files or debug output

### Scope Reduction: Principle of Least Credential

For each agent session, ask: what is the minimum set of credentials needed?

| Task | Credentials needed | Credentials NOT needed |
|------|-------------------|----------------------|
| Code review | None | AWS, DB, SSH |
| Write tests | None (usually) | AWS, DB, SSH |
| Deploy to staging | AWS (scoped to staging) | Production AWS, SSH keys |
| Database migration | DB credentials (one database) | AWS, SSH, API keys |
| Git operations | GitHub token (repo-scoped) | AWS, DB, SSH |

Most coding tasks need zero external credentials. The agent needs its own API key to function, and possibly a GitHub token — nothing else.

## Implementation Guide

### Quick Start: Environment Scrubbing

1. Audit current environment: `env | grep -i -E 'key|token|secret|password|credential'`
2. Create a wrapper script that starts the agent with a clean environment
3. Add `.env`, `*.pem`, `*.key`, `credentials/` to .aiignore

### Production: Credential Broker Integration

1. Set up HashiCorp Vault or AWS Secrets Manager
2. Create agent-specific policies with minimal permissions
3. Generate short-lived tokens per session
4. Pass only session tokens to the agent environment
5. Enable audit logging for all secret access

### Enterprise: Zero Standing Privileges

1. No long-lived credentials in developer environments
2. All access through just-in-time credential provisioning
3. Agent credentials expire after task completion
4. Automated secret rotation on schedule
5. Real-time monitoring for credential usage anomalies

### Validation: Testing Your Secret Boundaries

After setting up protections, verify they work:

```bash
# Test 1: Can the agent see sensitive env vars?
# Run inside the agent's environment:
env | grep -i -E 'aws|token|secret|key|password'

# Test 2: Can the agent read credential files?
cat ~/.aws/credentials 2>&1
cat ~/.ssh/id_rsa 2>&1

# Test 3: Can the agent use indirect access?
docker compose config 2>&1 | grep -i password
terraform output -json 2>&1

# Test 4: Can the agent access shell history?
cat ~/.bash_history 2>&1 | grep -i -E 'token|key|password'

# Test 5: Can the agent read /proc for env vars? (Linux only)
cat /proc/self/environ 2>&1
```

If any of these tests return credentials, your secret boundaries have gaps.

## Incident Response: When Secrets Are Exposed

If you suspect an agent has accessed or leaked credentials:

1. **Rotate immediately**: Assume the credential is compromised. Rotate the secret before investigating.
2. **Check audit logs**: Review agent session logs, API call logs, and network traffic for exfiltration attempts.
3. **Scope the blast radius**: Determine what the leaked credential grants access to. A read-only GitHub token is different from an AWS root key.
4. **Review generated code**: Search agent-generated commits for hardcoded secrets that may have been pushed.
5. **Scan for persistence**: Check if the agent created new credentials, API keys, or access tokens using the compromised credential.

## Limitations

- **Indirect access**: Agents can access secrets through tool execution (docker compose config, terraform output, etc.)
- **Memory access**: Secrets loaded into process memory can potentially be extracted
- **Log leakage**: Agents may log secrets in debug output or error messages
- **Developer friction**: Scrubbed environments require more setup; developers may bypass for convenience
- **Credential broker complexity**: Adds infrastructure and operational overhead
- **Agent needs some credentials**: The agent itself needs API keys (Anthropic, GitHub) to function
- **Evolving bypass techniques**: New indirect access methods are discovered regularly; static allowlists and blocklists require ongoing maintenance

## Tool-Specific Notes

### Claude Code
- Sandbox restricts file access but environment variables are inherited
- Network controls prevent exfiltration of accessed secrets
- Recommendation: Use env scrubbing wrapper + sandbox + network controls

### GitHub Copilot CLI
- No environment scrubbing
- No sandbox isolation for secrets
- Relies entirely on user discipline and external controls

### Cursor
- Known to access ~/.ssh/id_rsa through sandbox
- .cursorignore for .env is "best effort"
- `docker compose config` bypass documented
- Processes code on backend — secrets in code context are transmitted to Cursor servers

## Related CVEs

- CVE-2026-21852 (Claude Code API token exfiltration): Project files could access and exfiltrate Claude API credentials
- CVE-2025-59944 (Cursor case-sensitivity bypass): Could access .env via case variation (.ENV)
- Multiple Cursor sandbox bypasses enabling secret access

## References

- Knostic: .env Secret Leakage — https://www.knostic.ai/blog/claude-cursor-env-file-secret-leakage
- Cursor Sandboxing Leaks Secrets — https://luca-becker.me/blog/cursor-sandboxing-leaks-secrets/
- HashiCorp Vault — https://www.vaultproject.io/
- AWS STS — https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html
- HashiCorp: Confused Deputy Problem for AI — https://www.hashicorp.com/en/blog/before-you-build-agentic-ai-understand-the-confused-deputy-problem
- OWASP Secrets Management Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
