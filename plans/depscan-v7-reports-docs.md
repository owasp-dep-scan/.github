# dep-scan v7: reports, CI integration and documentation

| | |
| --- | --- |
| Epic | https://github.com/owasp-dep-scan/dep-scan/issues/574 |
| Owner | @prabhu |
| AI use | Auto |
| Corpus | none |

## Why

A person reading a dep-scan report wants three answers quickly: what to fix first, why it matters, and what was not checked. The 6.x console prints long per-package tables, and the HTML report is produced by re-parsing rich's HTML export (`packages/reporting-lib/src/reporting_lib/htmlgen.py`, 2,400 lines), which is fragile and cannot show anything the console did not print. There is no Markdown summary for pull requests, no GitHub Action, and no way to ask about one finding without re-running the scan. The documentation still describes vdb 6, has two lessons from May 2025 (one with leftover chatbot text in it), and has no page on exit codes or CI ([dep-scan#221](https://github.com/owasp-dep-scan/dep-scan/issues/221), [dep-scan#312](https://github.com/owasp-dep-scan/dep-scan/issues/312), [dep-scan#199](https://github.com/owasp-dep-scan/dep-scan/issues/199)).

## Scope

The console, HTML and Markdown renderers, `depscan explain` and `depscan query`, CI integrations and the documentation site. All renderers read the scan directory from [depscan-v7-findings.md](depscan-v7-findings.md) D4.

## Deliverables

### D1. Console output

Tracked in [dep-scan#638](https://github.com/owasp-dep-scan/dep-scan/issues/638).

Lead with a summary: counts by category and grade, the coverage panel (what was not assessed and why), and a "fix first" list grouped by remediation action, so one upgrade that resolves twelve findings is one line. Follow with per-category tables, then details on request (`--explain` shows witness paths and evidence). Keep the output within a terminal width and readable without colour. Done when the summary of a scan of the dep-scan repository fits on one screen, and a scan with a missing analyzer shows it in the coverage panel.

### D2. HTML report

Tracked in [dep-scan#639](https://github.com/owasp-dep-scan/dep-scan/issues/639).

Replace `htmlgen.py` with a renderer that reads the findings file and writes one self-contained HTML file with no external requests: summary, coverage, filters and sorting by category, grade, component and triage state, a detail view per finding with its evidence path, axes, rule and fix, and the policy result. Escape every value that comes from a scanned project. Done when the report of a corpus scan opens offline, passes an HTML validator, and a finding title containing markup is shown as text.

### D3. Markdown and pull request summaries

Tracked in [dep-scan#640](https://github.com/owasp-dep-scan/dep-scan/issues/640).

`depscan report --format markdown` writes a summary sized for a pull request comment or the GitHub step summary: new and fixed findings against a baseline, the policy result, and links to the full report artifact. Long lists are cut with a count of what was left out. Done when the summary of a corpus scan stays under GitHub's comment size limit and renders correctly on GitHub.

### D4. CI integrations

Tracked in [dep-scan#641](https://github.com/owasp-dep-scan/dep-scan/issues/641).

Add a GitHub Action (`action.yml` in the dep-scan repository) that runs a pinned dep-scan image, uploads SARIF to code scanning, writes the step summary, stores the scan directory as an artifact and optionally comments on the pull request, using a baseline from the target branch. Add templates or documented snippets for GitLab CI, Azure Pipelines and Jenkins. Done when the action runs in dep-scan's own CI on pull requests, and each snippet has been run once.

### D5. `explain` and `query`

Tracked in [dep-scan#642](https://github.com/owasp-dep-scan/dep-scan/issues/642).

`depscan explain <finding-id>` prints the evidence, axes, rule, witness path, triage state and fix for one finding from a saved scan, as text, Markdown or JSON. `depscan query <scan-dir>` filters findings by category, grade, severity, purl, CWE, rule, reach and triage state and prints them in any format. Both call the same functions as the MCP server. The 6.x explainer modes become views of `explain`. Done when both commands work on a scan directory copied to another machine without the source tree, and their JSON validates against the published schemas.

### D6. Documentation

Tracked in [dep-scan#643](https://github.com/owasp-dep-scan/dep-scan/issues/643).

Write the v7 documentation: a migration guide from v6 (vdb 7 and database download, every removed flag and its replacement, Python 3.11, exit codes, and why there are more findings), a new getting-started page, one lesson per workflow (a source repository, a container image, a binary, an Android app, an iOS app if [depscan-v7-mobile.md](depscan-v7-mobile.md) D3 lands, C and C++ memory safety, a supply-chain audit, CI gating with a baseline, and agent use through MCP and the skills), and reference pages generated from the schemas and the command help so they cannot drift. Explain `--profile` against `--bom-profile` ([dep-scan#221](https://github.com/owasp-dep-scan/dep-scan/issues/221)), add a lesson on common vulnerable repositories ([dep-scan#312](https://github.com/owasp-dep-scan/dep-scan/issues/312)), and add a contributing page ([dep-scan#199](https://github.com/owasp-dep-scan/dep-scan/issues/199)). The migration guide was started in [dep-scan#682](https://github.com/owasp-dep-scan/dep-scan/pull/682), and each breaking pull request adds to it. Remove the stale vdb 6 vocabulary and the leftover chatbot text in lesson 02. Done when every lesson runs end to end in a CI job against pinned inputs, and the issues above are closed.

## Risks and open questions

- Lessons that run real scans in CI need pinned inputs and a cached database. The corpus manifest from [depscan-v7-quality-release.md](depscan-v7-quality-release.md) D1 provides the inputs.

## Outcome

Filled in when the work ends.
