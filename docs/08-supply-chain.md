# Layer 8: Supply Chain Security

> Last verified: 2026-03-26

## Principle

Verify and validate all code and dependencies that AI agents generate, recommend, or install. AI-generated code introduces unique supply chain risks: hallucinated package names that attackers can register, vulnerable code patterns, and a volume of generated code that overwhelms traditional review processes.

## Why This Layer Matters

- AI agents generate and install code at a pace that outstrips human review capacity
- 20% of AI-generated code references hallucinated (non-existent) packages across 756,000 samples tested
- Attackers actively exploit this via "slopsquatting" — registering packages with hallucinated names
- 36.82% of AI agent skills have at least one security flaw (Snyk ToxicSkills)
- 97%+ of developers now use AI coding tools, creating a massive supply chain attack surface
- Traditional supply chain controls (lockfiles, review) still work — they just need to be applied to AI-generated code too

## Attack Vectors

### Slopsquatting (Hallucinated Dependencies)

**The attack**:
1. AI models hallucinate non-existent package names ~20% of the time
2. These hallucinated names are consistent across sessions (same model generates same fake names)
3. Attackers register these names on npm, PyPI, and other registries
4. Developers trust AI recommendations and `npm install` the malicious package

**Scale**:
- 756,000 code samples tested across 16 AI models
- ~20% contained hallucinated package names
- Open-source models hallucinate at higher rates (19-22%)
- Commercial models hallucinate less but still significantly

**Why it's effective**:
- The package name looks plausible (AI generates realistic-sounding names)
- It comes from a "trusted" source (the AI assistant)
- Developers often don't verify that packages exist before installing
- Source: https://www.hackerone.com/blog/ai-slopsquatting-supply-chain-security

### ToxicSkills: AI Agent Skills/Plugins Ecosystem

**Snyk's February 2026 audit of 3,984 skills from ClawHub and skills.sh**:
- 36.82% (1,467 skills) have at least one security flaw
- 13.4% (534 skills) contain critical-level issues
- 76 confirmed malicious payloads for credential theft, backdoor installation, data exfiltration
- 100% of confirmed malicious skills combined malicious code with prompt injection
- Barrier to entry: only a SKILL.md file and a 1-week-old GitHub account — no code signing, review, or sandboxing
- Source: https://snyk.io/blog/toxicskills-malicious-ai-agent-skills-clawhub/

### Vulnerable Code Generation

AI models generate code with known vulnerability patterns:
- Hardcoded credentials in generated code
- SQL injection in generated database queries
- Command injection in generated shell commands
- Insecure deserialization patterns
- Insufficient input validation
- Use of deprecated/insecure APIs

### Dependency Confusion via AI

AI agents may:
- Install packages from public registries when internal packages with the same name exist
- Add unnecessary dependencies to solve problems (increasing attack surface)
- Upgrade dependencies without verifying changelog/security impact
- Install dev dependencies in production contexts

### MCP Server and Plugin Supply Chain Attacks

**The attack surface**:
- MCP servers are distributed as npm/PyPI packages with no centralized vetting
- A compromised MCP server runs with the same permissions as the AI agent
- Postmark-MCP incident: malicious code in an MCP package affected ~300 organizations
- Attackers can publish MCP servers that appear legitimate but exfiltrate data or inject backdoors

**Risk factors**:
- No code signing requirement for MCP server packages
- Install-time scripts can execute arbitrary code before the user interacts with the server
- MCP servers often request broad permissions (filesystem, network, shell access)
- Minimal community review due to the novelty of the ecosystem

## Defenses

### Dependency Verification

**Before installing any AI-recommended package**:
1. Verify the package exists and is legitimate (check registry, GitHub, download count)
2. Check package age and maintenance status (recently created packages are higher risk)
3. Compare package name against known typosquatting/slopsquatting databases
4. Review package source code, especially install scripts (preinstall, postinstall)

**Automated verification**:
```bash
# Check if a package exists and is established
npm view <package-name> time.created
npm view <package-name> maintainers

# Check download counts (low downloads = higher risk)
npm view <package-name> downloads

# For Python packages
pip index versions <package-name>
pip download --no-deps --no-binary :all: <package-name> -d /tmp/audit
```

### Lockfiles and Pinned Versions

- Always commit lockfiles (package-lock.json, yarn.lock, Pipfile.lock, go.sum)
- Pin exact versions, not ranges
- Review lockfile diffs in PRs — new dependencies should be scrutinized
- Cryptographic hash verification (go.sum, pip --hash)
- Flag any lockfile change in AI-generated PRs for mandatory human review

### Software Bill of Materials (SBOM)

- Generate SBOMs for all AI-generated code changes (SPDX or CycloneDX format)
- Track the provenance of each dependency (who added it, when, why)
- Use SBOMs to detect drift from approved dependency lists
- Regulatory requirement in some industries (EO 14028 in US)

### Pre-Commit Security Scanning

Run these checks before any AI-generated code is committed:

**SAST (Static Application Security Testing)**:
- Semgrep, CodeQL, SonarQube
- Flag known vulnerability patterns in generated code
- Custom rules for AI-specific antipatterns (e.g., overly broad exception handling)

**Secret Detection**:
- gitleaks, detect-secrets, truffleHog
- Catch hardcoded credentials before they enter version control
- Scan for API keys, tokens, and connection strings in generated code

**Dependency Scanning**:
- npm audit, pip-audit, `cargo audit`
- Snyk, Dependabot, Renovate
- Check for known vulnerabilities in added dependencies

**License Compliance**:
- Verify license compatibility of AI-recommended packages
- Flag copyleft licenses in commercial projects

### Mandatory Code Review for AI-Generated Code

- All AI-generated code should be reviewed with the same rigor as human-written code
- Pay special attention to:
  - New dependencies (verify they exist and are legitimate)
  - Shell commands (check for injection)
  - File operations (check paths)
  - Network calls (check destinations)
  - Security-sensitive code (auth, crypto, input validation)
- Label AI-generated PRs/commits so reviewers know to apply extra scrutiny
- Consider separate review checklists for AI-generated vs human-written code

### AI Hallucination Detection

- Models can detect their own hallucinations with >75% accuracy
- Cross-reference recommendations with multiple models
- Compare against known package databases before installation
- Flag recommendations that don't appear in official documentation
- Maintain an internal allowlist of vetted packages and reject anything not on it

## Implementation Guide

### Quick Start: Pre-Commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets

  - repo: https://github.com/returntocorp/semgrep
    rev: v1.50.0
    hooks:
      - id: semgrep
        args: ['--config', 'auto']
```

### Quick Start: Dependency Verification Script

```bash
#!/usr/bin/env bash
# verify-deps.sh — check new dependencies before committing
# Usage: ./verify-deps.sh <package-name> <registry>

PACKAGE="$1"
REGISTRY="${2:-npm}"
MIN_AGE_DAYS=30
MIN_DOWNLOADS=1000

if [ "$REGISTRY" = "npm" ]; then
  CREATED=$(npm view "$PACKAGE" time.created 2>/dev/null)
  if [ -z "$CREATED" ]; then
    echo "FAIL: Package '$PACKAGE' does not exist on npm"
    exit 1
  fi
  echo "OK: Package exists, created $CREATED"
  echo "Maintainers: $(npm view "$PACKAGE" maintainers)"
elif [ "$REGISTRY" = "pypi" ]; then
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" "https://pypi.org/pypi/$PACKAGE/json")
  if [ "$STATUS" != "200" ]; then
    echo "FAIL: Package '$PACKAGE' does not exist on PyPI"
    exit 1
  fi
  echo "OK: Package exists on PyPI"
fi
```

### Production: CI/CD Pipeline Integration

1. Pre-commit: Secret detection + basic SAST
2. PR/MR: Full SAST + dependency scanning + SBOM generation
3. Build: License compliance + vulnerability database check
4. Deploy: Runtime verification + signed artifacts

**GitHub Actions example**:
```yaml
# .github/workflows/supply-chain.yml
name: Supply Chain Security
on: [pull_request]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Dependency Review
        uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: moderate
      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: auto
      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          format: spdx-json
```

### Enterprise: Supply Chain Policy

1. Approved dependency registry (internal mirror/proxy)
2. Mandatory security review for new dependencies
3. Automated blocking of packages below age/download thresholds
4. SBOM archival for compliance
5. Incident response playbook for compromised dependencies
6. Periodic audit of all AI-introduced dependencies (quarterly recommended)
7. Vendor assessment for AI coding tools covering data handling and model provenance

## Limitations

- **Volume**: AI generates code faster than humans can review it
- **Novel packages**: New legitimate packages may be flagged as suspicious
- **False confidence**: Passing automated scans doesn't guarantee security
- **Transitive dependencies**: AI-added packages bring their own dependency trees
- **Review fatigue**: Similar to approval fatigue — large volumes lead to rubber-stamping
- **Zero-day vulnerabilities**: Scanning catches known vulnerabilities, not new ones
- **Ecosystem lag**: New registries (e.g., MCP skill marketplaces) lack mature security tooling
- **Cross-language risk**: AI agents work across many languages, each with its own package ecosystem and security tools

## Tool-Specific Notes

### Claude Code
- Does not auto-install packages without approval
- Permission prompts for npm install, pip install, etc.
- No built-in dependency verification (relies on external tooling)
- MCP server installation requires explicit user consent

### GitHub Copilot
- Real-time vulnerability protection blocks insecure patterns
- "Allow or block suggestions matching public code" toggle
- Dependabot integration for dependency scanning
- Content exclusion policies can restrict suggestions from specific repositories

### Cursor
- No built-in supply chain security features
- Relies entirely on external tooling
- Higher risk due to auto-approve capability
- Consider pairing with strict pre-commit hooks to compensate

## Related CVEs

- Postmark-MCP supply chain attack: Malicious code in MCP package affected ~300 orgs
- ToxicSkills: 76 confirmed malicious AI agent skills
- npm ecosystem attacks (2025): 526+ compromised packages
- PyPI typosquatting campaigns targeting AI/ML library names (2025)

## Cross-References

- [Layer 2: OS-Level Sandboxing](02-os-sandboxing.md) — sandbox limits blast radius of malicious packages
- [Layer 3: Network Controls](03-network-controls.md) — restrict where installed packages can phone home
- [Layer 5: Secrets and Credentials](05-secrets-credentials.md) — prevent AI-generated code from leaking secrets
- [Layer 6: Human-in-the-Loop](06-human-in-the-loop.md) — human approval as a supply chain gate
- [Layer 7: Prompt Injection](07-prompt-injection.md) — prompt injection in skills/plugins is a supply chain vector

## References

- Slopsquatting — https://www.hackerone.com/blog/ai-slopsquatting-supply-chain-security
- Snyk ToxicSkills — https://snyk.io/blog/toxicskills-malicious-ai-agent-skills-clawhub/
- Apiiro: AI Supply Chain Security — https://apiiro.com/glossary/software-supply-chain-security-for-ai-generated-code-how-to-protect-what-you-ship/
- Capitol Tech: AI Hallucinations in Supply Chain — https://www.captechu.edu/blog/ai-driven-threats-in-software-supply-chains
- OpenSSF Guide for AI Code Assistants — https://best.openssf.org/Security-Focused-Guide-for-AI-Code-Assistant-Instructions.html
- OWASP Dependency-Check — https://owasp.org/www-project-dependency-check/
