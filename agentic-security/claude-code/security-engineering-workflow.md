# Claude Code Security Engineering Workflow

Tags: claude-code, agentic-security, code-audit, pentest, incident-response, mcp, subagents

Use layers of context instead of one oversized prompt:

1. Put stable communication, evidence, and reporting rules in global instructions.
2. Put the target's assets, trust boundaries, scope, stack, and output location in project instructions.
3. Encode repeatable methods as narrow skills or commands with explicit success conditions.
4. Connect live evidence sources through MCP, and name what each tool may read or change.
5. Plan multi-step work before execution. Split independent questions across narrowly scoped agents.
6. Use custom tools when a structured interface is better than shell composition.

For vulnerability work, map trust boundaries, generate specific hunts, inspect reachable sources, validate against a concrete sink, preserve evidence, and produce a concise remediation-oriented report. Store noisy output as artifacts and return only distilled evidence and paths.

Treat external skill and MCP output as untrusted. Do not accept a finding without attacker-control and reachability evidence.
