

<!-- OPTIMA:START project_overview -->
## Project Overview (Optima-managed)

- **Name**: Optima
<!-- OPTIMA:END project_overview -->

<!-- OPTIMA:START security_warnings -->
## Security Warnings (Optima-managed)

Potential secrets or credentials detected in source files. Review and fix these before committing.

- **CRITICAL** `aws_key` in `Optima/test/integration.test.ts:232` — aws_key detected: AKIA****MPLE
- **CRITICAL** `aws_key` in `Optima/test/tools.test.ts:72` — aws_key detected: AKIA****MPLE
- **CRITICAL** `aws_key` in `Optima/test/tools.test.ts:79` — aws_key detected: AKIA****MPLE
- **MEDIUM** `generic_secret` in `Optima/test/file-indexer.test.ts:82` — generic_secret detected: toke****890"
<!-- OPTIMA:END security_warnings -->

<!-- OPTIMA:START architecture_rules -->
## Architecture Rules (Optima-managed)

- **ALL logging must go to stderr — stdout is exclusively reserved for MCP JSON-RPC protocol messages. Any log line written to stdout corrupts the protocol stream and breaks tool calls silently.** — Optima is a stdio MCP server. The MCP SDK reads stdout as a binary protocol stream. Mixing human-readable log lines into stdout causes JSON parse failures on the client side. (scope: `C:/dev/Optima/Optima/src`)
<!-- OPTIMA:END architecture_rules -->
