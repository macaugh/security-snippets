# Security Snippets

Focused security-testing recipes indexed by category and technique. This repository is intended for RedAmon Tradecraft Lookup and human reference.

## Catalog

- [JWT `alg:none` authentication bypass](web/authentication/jwt/alg-none-bypass.md)
- [Claude Code security engineering workflow](agentic-security/claude-code/security-engineering-workflow.md)
- [Claude Code security skill stack](agentic-security/claude-code/security-skill-stack.md)
- [Building security CLIs with an NPX-first workflow](tool-development/cli/npx-security-tool-workflow.md)
- [AI-assisted IDOR testing with captured HTTP traffic](web/authorization/idor/ai-assisted-testing.md)
- [Web cache deception through cache/origin route confusion](web/cache/cache-deception/route-confusion-testing.md)
- [Multi-layer auth bypass via gateway/parser differential](web/authentication/multi-layer/gateway-parser-differential-bypass.md)
- [AI-assisted bug bounty workflow with Claude](agentic-security/claude-code/bug-bounty-claude-workflow.md)

## Organization

Paths follow `<domain>/<category>/<technology>/<technique>.md`. Keep the most important search terms in the path, heading, and `Tags` line so RedAmon can select the right entry from its repository sitemap.

## Entry Format

Each recipe should explain when the technique applies, how to construct or execute it, which fields need target-specific changes, and how to confirm the result. Prefer portable commands and include the expected success condition so an automated agent can distinguish a verified finding from a failed probe.

Suggested sections are `Applicability`, `Generate` or `Test`, `Send`, `Verify`, `Variants`, and `Report`. Add defensive caveats only when they clarify interpretation, such as common false positives or behavior that looks exploitable but is not.

## RedAmon Usage

Add this repository as an enabled Tradecraft Resource in RedAmon. When a hunting task reaches exploitation or post-exploitation, `tradecraft_lookup` can select a recipe by its path and title and retrieve the complete Markdown entry. Refresh the resource after new recipes are published so the repository sitemap contains their paths.
