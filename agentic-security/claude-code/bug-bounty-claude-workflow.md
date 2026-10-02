# AI-Assisted Bug Bounty Workflow with Claude

Tags: claude-code, bug-bounty, workflow, recon, evidence, reporting

Use a staged loop that keeps scope and evidence visible:

1. Define scope, allowed assets, rate limits, and out-of-scope paths in the project instructions.
2. Separate reconnaissance, attack-surface mapping, hypothesis generation, live validation, and reporting into distinct tasks.
3. Feed captured requests, JS bundles, API schemas, and source repositories through tools rather than pasting ad hoc notes into the chat.
4. Ask the agent to rank hypotheses by attacker-controlled input, reachable sink, and business impact.
5. Require an evidence oracle for every active test. A status code or scanner label alone is insufficient.
6. Preserve command output and request/response pairs as artifacts, then synthesize only the confirmed root cause and impact.

Effective prompts name the exact artifact to inspect, expected deliverable, allowed actions, and stopping condition. Keep noisy automation bounded; switch to manual replay for final confirmation.
