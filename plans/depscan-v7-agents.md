# dep-scan v7: agents

| | |
| --- | --- |
| Epic | https://github.com/owasp-dep-scan/dep-scan/issues/573 |
| Owner | @prabhu |
| AI use | Auto |
| Corpus | none |

## Why

Coding agents run security tools as part of everyday work: before upgrading a dependency, while fixing a vulnerability, when asked whether a repository is safe to use, and in CI. dep-scan 6.x gives an agent console tables, an HTML file and a VDR to parse, plus a `depscan-prompts.md` file of LLM prompts. Its `SKILL.md` describes how to review dep-scan's own code, not how to use dep-scan. There is no MCP server and no agent plugin.

The sibling projects already have these. cdxgen ships an agent plugin with 14 skills (among them `sbom-fidelity-loop`, which scans, reads cdxgen's fidelity report, applies the ranked fixes and re-scans), a read-only `bom-reviewer` agent, a `SessionStart` hook that checks the installed cdxgen and a `PostToolUse` hook that warns when an SBOM is older than the lockfile just edited, `userConfig` settings for Dependency-Track, TEA and generation defaults, and 26 eval cases. All of it is generated for Claude Code, Codex, Cursor, Gemini, Copilot CLI, Kiro and opencode from one template. cdxgen has no MCP server and will not get one: agents and dep-scan use it as a command-line tool. vulnerability-db ships an MCP server with 15 tools, and dosai one with 10. [Decision 0003](decisions/0003-depscan-v7-people-and-agents.md) sets the rule that every dep-scan capability must be usable by an agent; this plan builds the parts that make it so.

## Scope

The agent output contract, the MCP server, the agent plugin and its skills, remediation plans, evals and CI recipes, contributor-facing agent documentation, and the handoff to chennai. Human renderers are in [depscan-v7-reports-docs.md](depscan-v7-reports-docs.md).

## Approach

Mirror cdxgen's packaging so both projects install the same way and can be maintained with the same scripts: `agent-plugin/` holds `plugin.json` (Agent Plugins 1.0.0), `.claude-plugin/plugin.json`, `gemini-extension.json`, `skills/<name>/SKILL.md`, `reference/`, `agents/`, `hooks/` and `evals/`; the repository root holds `.claude-plugin/marketplace.json`, `.agents/plugins/marketplace.json` and `.cursor-plugin/marketplace.json`; a generator writes every manifest from one template and a CI check fails on drift (cdxgen's `contrib/gen-agent-manifests.js` and `contrib/check-agent-plugin.js` are the model). Skills call the `depscan` command; they do not reimplement it. dep-scan's skills point to cdxgen's skills for BOM generation and to vdb's MCP server for raw advisory lookups, instead of copying them.

## Deliverables

### D1. Agent output contract

Tracked in [dep-scan#630](https://github.com/owasp-dep-scan/dep-scan/issues/630).

Make every subcommand usable without a person: `--format json` everywhere, schemas for every JSON output printed by `depscan schema`, logs and progress on stderr only, no prompts, `NO_COLOR` and non-TTY detection, documented exit codes, and file paths in output that are absolute or relative to a stated root. Write `docs/agents.md`, a reference for agent authors. Done when a test runs every subcommand with `--format json` under a non-TTY, validates its stdout against its schema, and checks stderr carries no JSON.

### D2. MCP server

Tracked in [dep-scan#631](https://github.com/owasp-dep-scan/dep-scan/issues/631).

`depscan mcp` runs an MCP server over stdio. Read tools work on scan directories: `scan_summary`, `list_findings` (filter by category, grade, severity, purl, CWE, rule, reach and triage state), `get_finding`, `explain_finding`, `get_evidence`, `coverage`, `policy_status`, `list_components`, `remediation_plan`, `diff_scans`, `doctor` and `vdb_status`. `run_scan` starts a scan and defaults to a dry run that returns the planned commands; a real run never installs dependencies or runs build scripts unless the server was started with that allowed, and only inside allowed roots (MCP roots or `--allow-dir`). `record_triage` writes a triage decision and is off unless enabled. Responses are capped at 32 KB, with cursor paging and larger results written to a file in the scan directory and returned as a path. Tool input and output schemas are JSON Schema files printed by `depscan schema`, and tool names follow a verb and noun pattern. The server never needs another MCP server: `run_scan` runs cdxgen, atom, blint and dosai as command-line tools through `depscan.api`, the same path as `depscan scan`. Resources expose scan directories and findings by URI, and prompts cover triage and fix planning. Done when the server passes the MCP inspector's checks, every tool has a test, path confinement rejects `..` and symlinks out of the roots, and the server registers in the MCP registry as `io.github.owasp-dep-scan/dep-scan`.

### D3. Agent plugin package

Tracked in [dep-scan#632](https://github.com/owasp-dep-scan/dep-scan/issues/632).

Create `agent-plugin/` and the root marketplace files, generated from one template with a CI drift check, for Claude Code, Codex, Cursor, Gemini, Copilot CLI, Kiro and opencode. Include the MCP server configuration so installing the plugin makes the tools available, a `SessionStart` hook that warns when `depscan` or its tools are missing or older than the plugin, and an opt-in hook that suggests a re-scan after a lockfile changes. Hooks only warn; they never run a scan. Add `userConfig` settings, as cdxgen does, for the vulnerability database directory and flavor, the dep-scan server URL, the policy file and the cdxgen command, so a user sets them once for every skill. The plugin's README names cdxgen's plugin as its companion for BOM generation, and dep-scan skills refer to cdxgen skills by name instead of repeating their commands. Submit the plugin to the same directories as cdxgen's (the Codex plugin directory and awesome-copilot, following [cdxgen#4664](https://github.com/cdxgen/cdxgen/issues/4664) and [cdxgen#4665](https://github.com/cdxgen/cdxgen/issues/4665)). Done when installation is tested in Claude Code and Codex in CI or by a recorded manual run, and the drift check is a required job.

### D4. Skills

Tracked in [dep-scan#633](https://github.com/owasp-dep-scan/dep-scan/issues/633).

One skill per job, each with frontmatter `name` and a `description` that says what it does and when to use it, a decision table, a link to `reference/safety.md`, and command blocks:

| Skill | Use |
| --- | --- |
| `depscan-scan` | scan a repository or directory and summarise what to fix first |
| `depscan-container-image` | scan an image or rootfs, including unpackaged binaries |
| `depscan-binary` | analyse executables, installers and drivers |
| `depscan-mobile-app` | analyse an APK, AAB or IPA |
| `depscan-memory-safety` | find memory-safety defects in C and C++ and unsafe boundaries elsewhere |
| `depscan-supply-chain` | audit packages, provenance, CI workflows and AI components |
| `depscan-triage` | explain findings and record triage decisions with justifications |
| `depscan-fix-plan` | plan dependency upgrades that resolve findings |
| `depscan-vex-compliance` | produce VEX, CSAF and compliance reports from a scan |
| `depscan-ci-setup` | add dep-scan to a CI pipeline with a policy and baseline |
| `depscan-vdb` | check and refresh the vulnerability database |
| `depscan-coverage-loop` | scan, read the not-assessed entries in the scan manifest and the `depscan doctor` report, fix the missing tool, data or option each one names, and re-scan until coverage stops improving (modelled on cdxgen's `sbom-fidelity-loop`); it installs dep-scan's own tools and data only with the user's consent and never installs the scanned project's dependencies or runs its build scripts |

Skills that change state (`depscan-triage` writing decisions, `depscan-vdb` refreshing, `depscan-ci-setup` editing workflows, `depscan-coverage-loop` installing tools) set `disable-model-invocation: true` so they run only when the user asks. Add a read-only `vuln-triager` agent with the triage, scan-reading and fix-plan skills preloaded. `reference/safety.md` covers untrusted repositories (no installs, secure mode, no build scripts), secrets in output, and never claiming `not_affected` without evidence. Done when every skill passes the frontmatter and link checks and has eval cases (D6).

### D5. Remediation plans

Tracked in [dep-scan#634](https://github.com/owasp-dep-scan/dep-scan/issues/634).

`depscan fix --plan` (and the `remediation_plan` MCP tool) turns findings into upgrade sets: for each direct dependency, the smallest version that fixes all its findings and those of the transitive dependencies it pulls in, whether that crosses a major version, which findings it resolves, and transitive findings that need a parent upgrade or an override. Output is JSON and a table, with the package manager command for each ecosystem shown as text. dep-scan never edits files; an agent or person applies the plan. Done when the plan for a test npm, Python and Maven project resolves every fixable finding when applied by a test harness, and findings with no fix are listed as such.

### D6. Evals and CI recipes

Tracked in [dep-scan#635](https://github.com/owasp-dep-scan/dep-scan/issues/635).

Add eval cases for every skill in the shape cdxgen uses (`select-*` for when a skill should be chosen, `not-invoked-*` for when it should not, `safety-*` for refusing unsafe actions), run with `claude plugin eval` and recorded in CI where possible. Add CI recipes for running dep-scan through an agent in GitHub Actions (Claude Code action, Codex, and the Copilot cloud agent as in [cdxgen#4671](https://github.com/cdxgen/cdxgen/issues/4671)), with read-only permissions, pinned versions and no installs. Done when the eval suite runs with a threshold and its last result is linked from `docs/agents.md`, and each recipe has been run once on a test repository.

### D7. Contributor agent documentation

Tracked in [dep-scan#636](https://github.com/owasp-dep-scan/dep-scan/issues/636).

The root `SKILL.md` describes how to review dep-scan's code. Move it to `.agents/skills/depscan-maintainer/SKILL.md` with proper frontmatter, so users installing the plugin get the user skills and contributors get the maintainer skill. Update `AGENTS.md` for the v7 layout (the pipeline, adapters, schemas, scan directory) and the review rules from the overview. Done when the plugin checks pass with both skill sets and `AGENTS.md` names every package and its tests.

### D8. chennai handoff (Horizon Next)

Tracked in [dep-scan#637](https://github.com/owasp-dep-scan/dep-scan/issues/637), which is off the v7.0.0 milestone until a maintainer decides on it.

chennai, the interactive analysis tool built on the same analyzers, plans a symbol-map format (`chennai.symbolmap.v1`), a session file that opens a finding with the analysis already done, and an MCP server. Agree the session contract in both repositories first, then add `depscan explain <finding> --chennai`, which writes a session file from the scan directory. Done when chennai opens a dep-scan finding without re-running analysis. This needs a maintainer decision on chennai's schedule before it starts.

## Risks and open questions

- MCP tools that start scans run build tools on whatever path they are given. The defaults (dry run, no installs, allowed roots) must hold even when an agent asks for more; only the person who starts the server can widen them.
- Skill descriptions decide when agents pick a skill. The eval suite is the only reliable check, so D6 is part of done for D4, not an extra.

## Outcome

Filled in when the work ends.
