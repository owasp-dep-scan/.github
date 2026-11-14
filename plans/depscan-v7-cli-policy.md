# dep-scan v7: commands, configuration and policy

| | |
| --- | --- |
| Epic | dep-scan#TBD |
| Owner | @prabhu |
| AI use | Auto |
| Corpus | none |

## Why

The 6.x `depscan` command is one entry point with about sixty flat flags, some in mutually exclusive groups that block valid combinations (`--bom-engine` cannot be used with `--vulnerability-analyzer`). Users have asked for years for three things it does not do: fail a build on a severity or policy threshold ([dep-scan#395](https://github.com/owasp-dep-scan/dep-scan/issues/395)), suppress findings with a recorded reason ([dep-scan#396](https://github.com/owasp-dep-scan/dep-scan/issues/396), [dep-scan#110](https://github.com/owasp-dep-scan/dep-scan/issues/110)), and report only what a change introduced. Findings never change the exit code today, so CI has no way to gate on them.

## Scope

The command structure, the configuration file, exit codes and policy gates, triage decisions and suppressions, and baselines. Each feature plan adds its own subcommand on this structure.

## Approach

Subcommands follow the verb-per-job shape that cdxgen's binaries and `vdb db` already use. One `scan` verb handles every target kind, detected from the target (a directory, a git URL, a purl, a BOM, a container image or archive, a binary, an app or a firmware image), with `--checks` choosing what to analyse and profiles naming common sets. The planned commands:

```
depscan scan [targets...]     scan source, images, binaries, apps, BOMs or purls
depscan report <scan-dir>     render console, HTML, Markdown, SARIF or JSON from a saved scan
depscan explain <finding>     evidence and reasoning for one finding
depscan query <scan-dir>      filter findings for scripts and agents
depscan diff <old> <new>      new, fixed and changed findings between two scans
depscan fix <scan-dir>        remediation plan (agents plan D5)
depscan triage                record a triage decision with a justification
depscan vex | csaf            VEX or CSAF from a saved scan (compliance plan)
depscan compliance            CISA, CRA and SCVS benchmarks plus the vulnerability record
depscan audit <bom>           supply-chain audit of a BOM
depscan validate <file>       validate a BOM, VDR, VEX or CSAF document
depscan convert               CycloneDX and SPDX conversion
depscan sign | verify         document signatures
depscan vdb status|refresh    vulnerability database
depscan doctor                tools, data and capabilities
depscan serve                 HTTP server
depscan mcp                   MCP server
depscan schema [name]         JSON schemas of every output
depscan config init|show|validate
depscan version
```

Triage decisions use the CycloneDX VEX vocabulary (`analysis.state` and `analysis.justification`) so they map directly onto VEX and CSAF without translation.

## Deliverables

### D1. Subcommand command line

Build the command structure on `depscan.api` from [depscan-v7-findings.md](depscan-v7-findings.md) D3, with `scan`, `report`, `vdb`, `doctor`, `schema`, `config`, `serve` and `version` working, and the other verbs registered by their own plans. Target detection picks the right inventory path for each target kind and says which it chose. `--checks` takes `vulnerabilities`, `reachability`, `supply-chain`, `memory-safety`, `binary`, `mobile`, `code`, `license`, `crypto` and `all`; profiles (`quick`, `ci`, `deep`, `research`, `offline`) name sets of checks and analyzer settings. Every verb supports `--format json`, writes logs to stderr and never prompts. A v6-style invocation exits with code 2 and prints the equivalent v7 command. Remove the `depscan-vdb` and `depscan-validate` console scripts.

Fix the 6.x defects found on the way: `--profile license-compliance` without a `license` type raises `ValueError` in `cli.py` when it removes a missing list item, and `--exclude-type` passed through `--cdxgen-args` does not stop OS and container scans ([dep-scan#450](https://github.com/owasp-dep-scan/dep-scan/issues/450)); v7 gives exclusions a first-class option. Done when every v6 flag in `cli_options.py` has a documented v7 equivalent or a documented removal, the translator covers the flags used in the repository's own workflows, and each verb has a test that runs it end to end.

### D2. Configuration file

`depscan.toml` gains sections (`[scan]`, `[checks]`, `[policy]`, `[triage]`, `[vdb]`, `[tools]`, `[report]`, `[server]`) and a published JSON Schema. Precedence is command line, then environment, then the project file, then the user file (`~/.config/depscan/depscan.toml`), then defaults; `depscan config show` prints the merged result with the source of each value. A 6.x flat file still loads, with one warning per key naming its new place. Done when `depscan config validate` rejects an unknown key with its line number, and the precedence order has a test for each level.

### D3. Policy gates and exit codes

A `[policy]` section and `--fail-on` expressions that select findings by category, severity, exploitability grade, reachability, KEV listing, malware, licence id, confidence, rule id and "new since baseline". The exit codes are: 0 the scan finished and no policy rule matched, 1 the scan could not run (no BOM, no database, a required tool failed), 2 usage or configuration error, 3 a policy rule matched, 4 a coverage requirement was not met (for example `--require reachability` when reachability was not assessed). These match the numbering cdxgen's `cdx-validate` and `cdx-audit` use for findings (3). The JSON output names every rule that matched and the findings that matched it. Done when each exit code has a test, the CI documentation page shows four common policies, and [dep-scan#395](https://github.com/owasp-dep-scan/dep-scan/issues/395) is closed by it.

### D4. Triage decisions and suppressions

A triage file (`.depscan/triage.toml`, or a `[triage]` section) where each entry selects findings by fingerprint, or by advisory id and purl range, and records a CycloneDX analysis state (`not_affected`, `false_positive`, `exploitable`, `in_triage`, `resolved`), a justification from the CycloneDX list when the state is `not_affected`, a response when the risk is accepted or a fix is planned (`will_not_fix`, `can_not_fix`, `update`, `rollback`, `workaround_available`), free-text detail, the author and an optional expiry date. `depscan triage` writes entries from the command line or from an agent. Decisions change the finding's triage state, suppress it from policy gates (except expired ones, which warn), and flow into the VDR `analysis`, VEX, CSAF and SARIF suppressions. External VEX documents (CycloneDX VEX, OpenVEX and CSAF VEX) can be loaded as decisions with `--vex`. A decision whose selector no longer matches anything is reported as stale. Done when [dep-scan#396](https://github.com/owasp-dep-scan/dep-scan/issues/396) and [dep-scan#110](https://github.com/owasp-dep-scan/dep-scan/issues/110) are closed by it, and each output format shows a suppressed finding correctly.

### D5. Baselines and scan diffs

`depscan diff <old> <new>` compares two scan directories by fingerprint and reports new, fixed and changed findings (changed means the same fingerprint with a different severity, grade or triage state). `--baseline <scan-dir|findings.json>` on `scan` marks findings as new or existing, so a pull request gate can fail only on new findings. Done when a diff of two scans of consecutive commits of a test repository lists exactly the findings the commit added and removed, and the "new since baseline" policy selector uses it.

## Risks and open questions

- A severity threshold gate was declined in 2025 in favour of reachability-based triage. Here it returns as one selector among several, with the exploitability grade as the recommended gate in the documentation. A maintainer should confirm this direction before D3 merges.
- Fingerprint stability decides how useful baselines and suppressions are. D5 depends on the fingerprint decision in the findings plan D1.

## Outcome

Filled in when the work ends.
