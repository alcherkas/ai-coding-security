# Layer 11: Governance & Frameworks

> Last verified: 2026-03-26

## Principle

Align AI coding agent security with established industry frameworks, standards, and organizational policies. Governance provides the organizational structure, processes, and accountability that ensure technical controls (Layers 1-10) are deployed, maintained, and enforced consistently.

## Why This Layer Matters

- Technical controls alone are insufficient without organizational commitment
- Regulatory landscape is evolving rapidly (NIST AI Agent Standards Initiative, Feb 2026)
- Frameworks provide structured approaches that prevent ad-hoc, incomplete security implementations
- Compliance requirements increasingly cover AI-assisted development
- Cross-team alignment on security policies reduces inconsistent protection

## Key Frameworks

### OWASP

**AI Agent Security Cheat Sheet**:
- Practical guidance for securing AI agents with tool access
- Covers: input validation, authorization, monitoring, data protection
- Source: https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html

**Top 10 for Agentic Applications (2026)**:
- Ranked list of most critical risks for AI agent systems
- Includes: excessive agency, prompt injection, supply chain, data leakage
- Source: https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/

**LLM Prompt Injection Prevention Cheat Sheet**:
- Specific guidance for preventing prompt injection
- Source: https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html

### OpenSSF

**Security-Focused Guide for AI Code Assistant Instructions**:
- Best practices for configuring AI code assistants securely
- Focus on: system prompt security, output validation, scope restrictions
- Source: https://best.openssf.org/Security-Focused-Guide-for-AI-Code-Assistant-Instructions.html

### Google SAIF (Secure AI Framework)

- Defense-in-depth architecture specifically for generative AI workloads
- Focus on agents: https://saif.google/focus-on-agents
- Covers: model security, data protection, infrastructure hardening, operational security

### NIST AI Agent Standards Initiative (February 2026)

- Announced in response to escalating AI agent security incidents
- Priority areas for standardization:
  - Agent identity and authentication
  - Authorization and access control
  - Security architecture for autonomous agents
- Represents a major regulatory acknowledgment that current systems lack fundamental security infrastructure
- Source: https://www.nist.gov/caisi/ai-agent-standards-initiative

### AWS Well-Architected Generative AI Lens

- Extends AWS Well-Architected Framework to cover generative AI
- Security pillar includes least privilege for AI agents (GENSEC05-BP01)
- Source: https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/gensec05-bp01.html

### Red Hat Defense-in-Depth for AI

- Layered security controls addressing both traditional and AI-specific threats
- Source: https://www.redhat.com/en/blog/model-context-protocol-mcp-understanding-security-risks-and-controls

### Cisco AI Security Reference Architecture

- Design patterns for LLM application security
- Network-centric approach to AI security

## Implementation Stages

Security controls should be integrated into the development lifecycle:

| Stage | Controls | Tools |
|-------|----------|-------|
| **Pre-commit** | Secret detection, basic SAST, .aiignore validation | gitleaks, detect-secrets, Semgrep |
| **PR/MR Review** | Full SAST, dependency scanning, code review | CodeQL, Snyk, human review |
| **Build** | SBOM generation, license compliance, vulnerability DB check | Syft, Grype, npm audit |
| **Deployment** | Runtime verification, signed artifacts, environment validation | Sigstore, OPA, admission controllers |
| **Runtime** | Behavioral monitoring, audit logging, anomaly detection | SIEM, custom monitoring |

## Organizational Policies

### AI Agent Usage Policy

Every organization using AI coding agents should have a policy covering:
- Which agents are approved for use
- Which projects/repos can use AI agents
- Classification of data that AI agents may access
- Approval requirements for new MCP servers/plugins
- Incident response procedures for AI agent security events
- Training requirements for developers using AI agents

### Risk Assessment Framework

For each AI agent deployment, assess:
- What data can the agent access?
- What actions can the agent take?
- What is the blast radius of a compromised agent?
- What compensating controls are in place?
- Who is accountable for agent security?

### Regular Review Cadence

- Monthly: Review agent audit logs for anomalies
- Quarterly: Update approved agent/MCP server list
- Semi-annually: Red team AI agent configurations
- Annually: Full security assessment of AI agent infrastructure

## References

- OWASP AI Agent Security Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html
- OWASP Top 10 for Agentic Applications — https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/
- OpenSSF AI Code Assistant Guide — https://best.openssf.org/Security-Focused-Guide-for-AI-Code-Assistant-Instructions.html
- Google SAIF — https://saif.google/focus-on-agents
- NIST AI Agent Standards — https://www.nist.gov/caisi/ai-agent-standards-initiative
- AWS Generative AI Lens — https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/gensec05-bp01.html
- AI Red Teaming (HackerOne) — https://www.hackerone.com/blog/ai-red-teaming-explained-by-red-teamers/
