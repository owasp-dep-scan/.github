# 0003. People and agents use the same interfaces

Status: accepted, 2026-11-14. Plans: [depscan-v7-agents.md](../depscan-v7-agents.md), [depscan-v7-reports-docs.md](../depscan-v7-reports-docs.md).

## Context

Coding agents now run security tools as often as people do: they scan a repository before a change, triage results, plan dependency upgrades and write VEX statements. In 6.x an agent has to parse console tables or an HTML file written for people, and the repository's `SKILL.md` describes how to review dep-scan's code rather than how to use it. cdxgen, vulnerability-db and dosai already ship MCP servers or agent plugins, and dep-scan should fit beside them.

## Decision

The command line with JSON output is the contract. Every subcommand accepts `--format json`, writes progress and logs to stderr only, never prompts, and exits with a documented code. Every JSON output has a published schema. The Python API, the MCP server, the agent skills and plugins, the HTTP server, the HTML report and the console are views of the same scan directory and the same functions, so a capability added to one is available to all. A feature is not done until an agent can use it.

Agent integrations follow three safety rules. Reading a completed scan is always allowed. Starting a scan from an agent defaults to a dry run that shows the commands, never installs dependencies or runs build scripts unless the user turns that on, and is confined to directories the user allowed. Writing triage decisions, signing documents and refreshing databases are actions a skill offers only when the user asks for them.

No LLM runs in the scan path. dep-scan gives agents evidence; it never needs a model to produce a result, and its results do not change depending on whether an agent is involved.

## Consequences

Human-oriented output (console tables, HTML, Markdown) is rendered from the same findings file, so the renderers cannot drift from the JSON. The 6.x `--explanation-mode LLMPrompts` output is replaced by the MCP server and skills. dep-scan's agent plugin is generated from one template for each supported client, the same way cdxgen's is, and checked for drift in CI.
