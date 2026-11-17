# dep-scan v7 Lodestar (overview)

| | |
| --- | --- |
| Epic | Each plan below has its own epic; this file is the map. All of them are in the [v7.0.0 Lodestar](https://github.com/owasp-dep-scan/dep-scan/milestone/2) milestone. |
| Owner | @prabhu |
| AI use | Auto: implementation agents write each deliverable, a reviewing agent verifies, refactors and opens the pull request, a maintainer approves |
| Corpus | `depscan-v7.corpus.json`, created by [depscan-v7-quality-release.md](depscan-v7-quality-release.md) D1 |

## Why

dep-scan 6.3.2 is a good vulnerability reporter for source projects and container images, with reachability for some languages. Everything around it has moved on, and dep-scan has not followed:

- It pins vulnerability-db 6.7.7, while vdb 7.0.5 has shards, flavors, coverage gaps, staleness signals, an MCP server and more advisories.
- Its container images and standalone binaries install the cdxgen 12.9 line, which uses atom 2.5. cdxgen 13.8 brings atom 4 with callsites for every atom language, `atom memory-safety` for C and C++, introspection, audit rule packs, compliance benchmarks, signing and conversion.
- It pins blint 3.2, while blint 4 analyses Android and iOS apps, Windows drivers and installers, links ELF closures, diffs builds and has a Python API.
- dosai 5, golem, rusi and kosi report exploit chains, attack surfaces, trust zones and witness paths that dep-scan reads only in part, and kosi not at all.

There is a larger gap too. Users ask dep-scan the same question about very different software: what here can be exploited, how sure is that answer, and what should be fixed first. Today it answers only for known vulnerabilities in packages. It says nothing about memory-safety defects in C and C++ code, little about supply-chain risk beyond registry metadata, and little about mobile apps, binaries or firmware. Its answers reach a person through console tables and an HTML report, and reach a coding agent only through files the agent has to parse. It has no subcommands, no suppressions, no policy gate, no SARIF and no MCP server.

v7 closes both gaps. The goal is a security tool that gives one honest, evidence-backed answer for source trees, container images, binaries, mobile apps and firmware, covering known vulnerabilities, memory-safety defects and supply-chain risk, and that gives the same answer to a person at a terminal, a CI job and a coding agent.

## What v7 is

Three ideas carry the release. Each has a decision record.

**One finding model.** Every analyzer, from vdb matching to `atom memory-safety`, blint reviews and `cdx-audit` rules, produces findings of one shape. Each finding carries its evidence, the producer and version that made it, a stable fingerprint, and a coverage state. "Not assessed" (an analyzer did not run, hit a budget or had no data) is a separate state from "assessed and not found", and no report, VEX statement or policy gate may confuse them. See [decisions/0002](decisions/0002-depscan-v7-one-finding-model.md).

**Delegate what the toolchain already does.** cdxgen owns BOM generation, evidence, audit rules, validation, signing and conversion. vdb owns vulnerability data. atom, the cdxgen plugins, dosai and blint own analysis. dep-scan owns orchestration, matching, exploitability grading, policy, triage and presentation. When a needed capability is missing upstream, the fix goes upstream. See [decisions/0004](decisions/0004-depscan-v7-delegate-to-the-toolchain.md).

**Every capability is reachable by people and agents alike.** The CLI with JSON output is the contract. The Python API, the MCP server, the agent skills and plugins, the HTTP server, the HTML report and the console are views of the same scan directory. A feature is not done until an agent can use it without parsing human text. No LLM runs in the scan path. See [decisions/0003](decisions/0003-depscan-v7-people-and-agents.md).

v7 is a clean break from v6: subcommands, vdb 7 and Python 3.11. See [decisions/0001](decisions/0001-depscan-v7-clean-break.md).

## Where things stand

| Component | dep-scan 6.3.2 uses | Current upstream (2026-11-17) |
| --- | --- | --- |
| vulnerability-db | 6.7.7 | 7.0.6 |
| cdxgen | 12.9.x (`@cyclonedx/cdxgen`), atom 2.5.6 | 13.8.0, with atom 4.0.2 and atom-parsetools 2.0.0 |
| cdxgen-plugins-bin | through cdxgen 12 | 4.3.1 (golem, rusi, kosi, dosai 5.0.0, trustinspector) |
| atom-tools | ~=0.9.1 | 1.0.5 |
| blint | ~=3.2.3 | 4.0.1 |
| dosai | report schema 4.x and 5.x | 5.0.0, report schema 5.1.0 |
| chen / atom | not called directly | 4.0.1 / 4.0.2, with `atom memory-safety` (37 rules) |
| edga | not used | 0.2.0 (EDG front end for atom's C and C++ analysis) |
| Python | >=3.10 (end of life since October 2026) | 3.11 to 3.14 supported |

## The plans

| plan | epic | what |
| --- | --- | --- |
| [depscan-v7-foundation.md](depscan-v7-foundation.md) | [dep-scan#565](https://github.com/owasp-dep-scan/dep-scan/issues/565) | open the v7 line; vdb 7, cdxgen 13.8 and atom 4, blint 4; `depscan doctor` |
| [depscan-v7-findings.md](depscan-v7-findings.md) | [dep-scan#566](https://github.com/owasp-dep-scan/dep-scan/issues/566) | finding schema, evidence store, scan pipeline and Python API, scan directory, SARIF, VDR writer |
| [depscan-v7-cli-policy.md](depscan-v7-cli-policy.md) | [dep-scan#567](https://github.com/owasp-dep-scan/dep-scan/issues/567) | subcommands, configuration, policy gates and exit codes, triage decisions, baselines |
| [depscan-v7-exploitability.md](depscan-v7-exploitability.md) | [dep-scan#568](https://github.com/owasp-dep-scan/dep-scan/issues/568) | exploitability axes and rule table, reach from atom 4, exploit signals, symbol hints, analyzer uplifts, exposure |
| [depscan-callsites.md](depscan-callsites.md) | [dep-scan#555](https://github.com/owasp-dep-scan/dep-scan/issues/555) | keep atom's callsites slice and score reachability against real labels (already planned) |
| [depscan-v7-supply-chain.md](depscan-v7-supply-chain.md) | [dep-scan#569](https://github.com/owasp-dep-scan/dep-scan/issues/569) | audit rule packs, package risk, malware, provenance, CI pipelines, AI supply chain, dependency health and licences |
| [depscan-v7-memory-safety.md](depscan-v7-memory-safety.md) | [dep-scan#570](https://github.com/owasp-dep-scan/dep-scan/issues/570) | C and C++ memory safety, native code inside packages, unsafe boundaries, Rust/Go/Kotlin findings, binary hardening, correlation with CVEs |
| [depscan-v7-binaries.md](depscan-v7-binaries.md) | [dep-scan#571](https://github.com/owasp-dep-scan/dep-scan/issues/571) | binaries, installers, drivers, container images, presence corroboration, build diffs, firmware |
| [blint-depscan-v7.md](blint-depscan-v7.md) | [blint#270](https://github.com/owasp-dep-scan/blint/issues/270) | what blint must fix or add for dep-scan v7 |
| [blint-uefi-firmware-images.md](blint-uefi-firmware-images.md) | [blint#264](https://github.com/owasp-dep-scan/blint/issues/264) | UEFI firmware images in blint (already planned) |
| [depscan-v7-mobile.md](depscan-v7-mobile.md) | [dep-scan#572](https://github.com/owasp-dep-scan/dep-scan/issues/572) | Android and iOS apps, reachability inside apps, MASVS mapping |
| [depscan-v7-agents.md](depscan-v7-agents.md) | [dep-scan#573](https://github.com/owasp-dep-scan/dep-scan/issues/573) | agent output contract, MCP server, skills and plugins, remediation plans, evals |
| [depscan-v7-reports-docs.md](depscan-v7-reports-docs.md) | [dep-scan#574](https://github.com/owasp-dep-scan/dep-scan/issues/574) | console, HTML, Markdown and PR summaries, explain and query, CI integrations, documentation |
| [depscan-v7-compliance.md](depscan-v7-compliance.md) | [dep-scan#575](https://github.com/owasp-dep-scan/dep-scan/issues/575) | VEX, CSAF 2.1, CRA and CISA benchmarks, signing, input and output formats |
| [depscan-v7-distribution.md](depscan-v7-distribution.md) | [dep-scan#576](https://github.com/owasp-dep-scan/dep-scan/issues/576) | container images, standalone binaries, offline mode, server mode, dep-scan's own release supply chain |
| [depscan-v7-quality-release.md](depscan-v7-quality-release.md) | [dep-scan#577](https://github.com/owasp-dep-scan/dep-scan/issues/577) | ground-truth corpus, scorer and gates, false-positive backlog, the 7.0.0 release |

## Order of work

```
wave 1  foundation D1-D3, D5 ‖ findings D1 ‖ blint D1, D4
wave 2  foundation D4, D6-D8 ‖ findings D2-D4 ‖ cli-policy D1-D2 ‖ agents D1 ‖ distribution D1
wave 3  agents D2-D4 ‖ findings D5-D6 ‖ cli-policy D3-D5 ‖ exploitability ‖ supply-chain
        ‖ memory-safety D1, D3, D5 ‖ binaries D1-D4 ‖ mobile ‖ compliance D1-D4 ‖ callsites (#555)
wave 4  agents D5-D7 ‖ reports-docs ‖ quality D1-D3 ‖ distribution D2-D5 ‖ memory-safety D2, D6, D7
wave 5  quality D4: release candidates and 7.0.0
next    memory-safety D4 (Rust, Go, Kotlin findings), binaries D5 (firmware), compliance D5 (CycloneDX 2.0),
        agents D8 (chennai handoff)
```

The finding schema (findings D1) and the scan pipeline (findings D3) come first because every later deliverable writes findings into them. The agent output contract (agents D1), the MCP server (agents D2) and the first skills (agents D3, D4) follow the scan directory (findings D4) and the subcommands (cli-policy D1) directly, so agents can use each later deliverable as soon as it lands. Memory-safety D1 needs only the finding schema and `depscan doctor`, so it can start in wave 3 beside them. Work in the same wave can run in parallel. The "next" row depends on upstream releases that are planned but not shipped: memory-safety findings in rusi, golem and kosi (cdxgen/cdxgen-plugins-bin#170, cdxgen/cdxgen#4777), firmware SBOMs in cdxgen 13.9 with romi, the ratified CycloneDX 2.0 schema, and chennai's symbol maps. They stay in the milestone with Horizon set to Next, and the release does not wait for them.

## Horizon and priority

Horizon Now marks work that 7.0.0 needs. Horizon Next marks work that may move to 7.1 if its upstream dependency is late. Priority P0 marks the release-blocking core (foundation, finding model, CLI and policy). P1 marks the capabilities v7 is named for. P2 marks the rest.

## How work is checked

Every deliverable is gated by tests in the repository it lands in. A deliverable that changes what dep-scan reports also runs against the corpus, and its pull request states the before and after numbers for the categories it touches. Scores are never improved by changing the product to fit the scorer: when the scorer is wrong, fix the scorer.

Three properties are checked in review on every pull request, because they are the ones that silently break a security tool:

1. A failure must not look like a clean result. A missing tool, a timeout, a coverage gap or an empty database is reported as not assessed, never as zero findings.
2. A claim needs evidence. A reachability, exploitability or `not_affected` verdict without an evidence path behind it is a bug.
3. Untrusted input stays untrusted. Scanning a repository, image or app must not run its build scripts unless the user asked for it, and server and MCP modes must not widen what a caller can read or execute.

## How agents pick up work

Each deliverable is a sub-issue of its plan's epic, in the v7.0.0 Lodestar milestone. Issues ready for an agent have Status Ready and the `needs agent` label. An issue blocked by another issue or by an upstream release lists it under "Blocked by", and its Status stays Backlog or Blocked until that clears.

An implementation agent takes one issue, reads its plan section, works on a branch named after the issue (for example `v7/foundation-vdb7`), and opens one pull request against `master` that closes the issue. A reviewing agent re-runs every claim in the pull request, checks the three properties above, and either refactors on the same branch or sends the work back. A maintainer approves and the merge queue merges. Pull requests carry the `ai-auto` label. Commit messages and pull request descriptions follow the organization's [contributing guide](../CONTRIBUTING.md).

## Upstream dependencies

| Need | Upstream issue | Needed by |
| --- | --- | --- |
| cdxgen 13.8.0 released with atom 4 | cdxgen 13.8.0 milestone | foundation D5, D6 |
| blint 4.1.0 with a working default blintdb image | blint epic in [blint-depscan-v7.md](blint-depscan-v7.md) | foundation D7 |
| `PackageReachability.EdgeIds` that exist in `CallGraph.Edges` | [dosai#129](https://github.com/owasp-dep-scan/dosai/issues/129) | exploitability D5 |
| a JSON Schema for dosai 5.x reports | [dosai#135](https://github.com/owasp-dep-scan/dosai/issues/135) | exploitability D5 |
| KEV and EPSS in vdb | [vulnerability-db#135](https://github.com/AppThreat/vulnerability-db/issues/135), [vulnerability-db#309](https://github.com/AppThreat/vulnerability-db/issues/309) | exploitability D3 |
| flavor and metadata presence in `vdb db status` | [vulnerability-db#308](https://github.com/AppThreat/vulnerability-db/issues/308) | foundation D4 |
| end-of-life data in vdb | [vulnerability-db#204](https://github.com/AppThreat/vulnerability-db/issues/204) | supply-chain D7 |
| memory-safety findings with function, purl and fingerprint | [atom#413](https://github.com/AppThreat/atom/issues/413) | memory-safety D1, D7 |
| memory-safety in rusi, golem and kosi | [cdxgen-plugins-bin#170](https://github.com/cdxgen/cdxgen-plugins-bin/issues/170), [cdxgen#4777](https://github.com/cdxgen/cdxgen/issues/4777) | memory-safety D4 |
| kosi's raw report kept for dep-scan | [cdxgen#4806](https://github.com/cdxgen/cdxgen/issues/4806) | exploitability D6 |
| SPDX input in `cdx-convert` | [cdxgen#4807](https://github.com/cdxgen/cdxgen/issues/4807) | compliance D5 |
| callsites as the default occurrence source | [cdxgen#4693](https://github.com/cdxgen/cdxgen/issues/4693) | exploitability D2 |
| firmware SBOMs | [cdxgen#4620](https://github.com/cdxgen/cdxgen/issues/4620), [blint#264](https://github.com/owasp-dep-scan/blint/issues/264) | binaries D5 |
| R8 mapping for Android bytecode | [atom#360](https://github.com/AppThreat/atom/issues/360) | mobile D4 |
| MCP paging contract shared with cdxgen | [cdxgen#4656](https://github.com/cdxgen/cdxgen/issues/4656) | agents D2 |
| C memory safety across allocation styles and library profiles | [chen#328](https://github.com/AppThreat/chen/issues/328), [chen#325](https://github.com/AppThreat/chen/issues/325) | memory-safety D1, D7 |
| memory-safety findings in system headers with a correct path | [chen#337](https://github.com/AppThreat/chen/issues/337) | memory-safety D1 |
| C and C++ front-end contexts and template identity | [chen#335](https://github.com/AppThreat/chen/issues/335), [atom#415](https://github.com/AppThreat/atom/issues/415) | memory-safety D1, D7 |
| memory-safety precision and recall measured per allocation style | [atom#414](https://github.com/AppThreat/atom/issues/414) | memory-safety D7 |
| shared memory-safety finding contract and scorer for the plugins | [cdxgen-plugins-bin#171](https://github.com/cdxgen/cdxgen-plugins-bin/issues/171) | memory-safety D4, D7 |

## Keeping the plans current

cdxgen, atom, chen, the cdxgen plugins, vulnerability-db, blint and dosai change every week, and dep-scan v7 is built on them. The plans are reviewed once a week, and the review is a pull request to this repository:

1. Update "Where things stand" from the latest releases of each component.
2. Read the issues and pull requests opened or closed that week in cdxgen, cdxgen-plugins-bin, AppThreat (atom, chen, vulnerability-db, atom-tools, edga, chennai) and owasp-dep-scan (blint, dosai), and the changes to cdxgen's `agent-plugin/`. For each new tool, option, output format or agent feature, decide whether a plan section uses it, needs to wait for it, or can ignore it.
3. Edit the affected plan sections and the "Upstream dependencies" table. A new deliverable gets a section in its plan and a sub-issue of that plan's epic in the milestone; a changed deliverable gets its issue body updated to match.
4. Add a dated entry to "Changes to the plans" below that names what changed and why.

Plans describe the work, not its progress. Which tasks are done is tracked in the issues and the project board.

## Changes to the plans

- **2026-11-17.** Memory safety: the atom 4 options dep-scan must pass (`--no-build-tools`, `--parse-report`, `--bom-file`, `--memory-api-config`), findings in system headers and template instances, and a table of the chen, atom and plugins issues opened after the plan. Agents: cdxgen's current plugin (hooks, `userConfig`, evals, the `bom-reviewer` agent), shared MCP schemas and tool naming with cdxgen, plugin directory submissions, a `depscan-coverage-loop` skill and the Copilot cloud agent recipe. Order of work: agents D2 to D4 move from wave 4 to wave 3. "Where things stand" updated to 2026-11-17.

## Non-goals

- No hosted service, no telemetry and no network access the user did not ask for. Air-gapped CI remains a first-class deployment.
- No LLM in the scan path. Agents read dep-scan's results; dep-scan never needs a model to produce them.
- No vulnerability data sources inside dep-scan. Data belongs in vulnerability-db.
- No secret scanning in v7, and no static analysis rules written in dep-scan itself. Code-level findings come from the analyzers' own rules (dosai, kosi, golem, rusi, atom and blint), as planned in [depscan-v7-exploitability.md](depscan-v7-exploitability.md) D9.

## Outcome

Filled in when the release ships.
