# Layer 10: Audit & Observability

> Last verified: 2026-03-26

## Principle

Log, monitor, and analyze all AI agent actions to detect threats, investigate incidents, and maintain compliance. Unlike preventive layers (1-9), this layer is detective — it doesn't stop attacks but ensures they are detected and can be investigated. Without observability, you're flying blind.

## Why This Layer Matters

- Preventive controls fail — when they do, audit logs are the only way to understand what happened
- Many attacks (credential theft, data exfiltration) are not immediately visible
- Compliance requirements (SOC 2, ISO 27001, HIPAA) mandate audit trails for automated actions
- Behavioral analysis can detect anomalies that individual action checks miss
- Post-incident forensics depend on comprehensive logging
- AI agents operate at machine speed — manual observation is impossible

## What to Log

### Agent Identity and Context
- Which agent (Claude Code, Copilot, Cursor)
- Agent version
- Session ID
- User identity (who initiated the session)
- Authentication context (which credentials/tokens are active)
- Working directory and project context

### Actions and Tool Usage
- Every tool invocation (read, write, execute, network)
- Tool parameters and arguments
- Tool responses (or at minimum, response size and status)
- MCP server interactions (which server, which tool, what input/output)
- File operations: path, operation type, content hash
- Shell commands: full command string, exit code, output summary
- Git operations: commits, pushes, branch operations

### Reasoning and Decisions
- Agent's reasoning path (if available via API)
- Permission decisions: what was auto-approved vs. manually approved vs. denied
- Escalation events: when the agent requested human approval
- Error conditions and retry attempts

### Data Access Patterns
- Which files were read and when
- Volume of data processed
- Network requests: destination, payload size, response size
- Any interaction with external services

## Monitoring Approaches

### Real-Time Behavioral Analysis

**Baseline establishment**:
- Normal number of file operations per session
- Typical network request patterns
- Expected working directory scope
- Standard tool invocation sequences

**Anomaly detection signals**:
- File access outside normal scope
- Unusual volume of read operations (possible exfiltration preparation)
- Network requests to new/unusual destinations
- Rapid succession of privileged operations
- Accessing credential files or environment variables
- Attempting to disable or modify audit logging

**Automated response triggers**:
- Pause agent and require human review
- Rate-limit operations
- Terminate session on high-severity anomalies
- Alert security team

### Historical Analysis

- Trend analysis: increasing scope of operations over time
- Cross-session patterns: agent behavior changes after configuration updates
- Correlation: agent actions that coincide with security events
- Compliance reporting: periodic summaries of agent activities

## Tools and Platforms

### Built-In Logging

**Claude Code**:
- Comprehensive audit logging available
- SOC 2-aligned audit capabilities in enterprise tier
- Zero-data-retention options available

**GitHub Copilot**:
- User engagement data retained for 2 years
- Enterprise: audit log integration available
- Prompts and suggestions NOT retained (Business/Enterprise via supported IDEs)

### MCP Audit Logging

- **Entro MCP Audit Plugin**: Compliance-focused MCP action tracking
- **Tetrate MCP Audit Logging**: AI agent action tracing for service mesh environments
- **Custom logging**: Implement logging at the MCP proxy/gateway level

### External Monitoring

- **Zenity.io**: Compliance-focused AI agent monitoring
- **Rubrik**: Data security monitoring for AI workloads
- **SIEM integration**: Forward agent logs to Splunk, ELK, Datadog for correlation with other security events

## Implementation Guide

### Quick Start

1. Enable built-in logging for your AI agent (check agent documentation)
2. Log to a file that's outside the agent's write scope (prevent log tampering)
3. Review logs after each significant agent session
4. Set up alerts for known dangerous patterns (credential file access, force push)

### Production

1. Centralized log aggregation (ELK, Datadog, Splunk)
2. Real-time alerting on anomaly detection
3. Baseline establishment for normal agent behavior
4. Weekly review of agent activity summaries
5. Retention policy aligned with compliance requirements

### Enterprise

1. SIEM integration with correlation rules for agent-specific threats
2. Automated behavioral analysis with ML-based anomaly detection
3. Compliance dashboards for SOC 2 / ISO 27001 reporting
4. Incident response playbooks that incorporate agent logs
5. Cross-agent correlation (detect multi-agent attack patterns)
6. Regular red team exercises to validate detection capabilities

## Limitations

- **Volume**: AI agents generate massive log volumes — storage and analysis costs
- **Privacy**: Logging code context may capture sensitive data
- **Latency**: Real-time analysis adds processing overhead
- **False positives**: Anomaly detection requires tuning to reduce noise
- **Log integrity**: If the agent has write access to logs, it could tamper with them
- **Detective, not preventive**: Logging detects attacks after they happen, not before

## References

- Auditing and Logging AI Agent Activity — https://www.loginradius.com/blog/engineering/auditing-and-logging-ai-agent-activity
- Tetrate MCP Audit Logging — https://tetrate.io/learn/ai/mcp/mcp-audit-logging
- Claude Code Security — https://code.claude.com/docs/en/security
- OWASP Logging Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
