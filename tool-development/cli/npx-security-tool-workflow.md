# Build Security CLI Tools with an NPX-First Workflow

Tags: security-tooling, cli, npx, nodejs, claude-code, scaffold, testing

Define the smallest useful product before generating code: purpose, target input, required output, and success criteria. Put constraints in project instructions, then generate and review a plan before writing files.

The scaffold should include one executable entry point, argument parsing, target validation, a core operation module, human and JSON output, predictable exit codes, and focused tests. Keep dependencies minimal. Review missing flags and unsafe defaults before implementation.

For active security tools, require explicit targets, conservative concurrency, timeouts, and a dry-run or list-only mode where practical. Never let convenience flags silently expand scope. Test the packaged `bin` path instead of only invoking source files directly.
