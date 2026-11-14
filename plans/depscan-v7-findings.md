# dep-scan v7: findings, evidence and the scan directory

| | |
| --- | --- |
| Epic | https://github.com/owasp-dep-scan/dep-scan/issues/566 |
| Owner | @prabhu |
| AI use | Auto |
| Corpus | none |

## Why

In 6.x the result of a scan is spread across a CycloneDX VDR, console tables, an HTML file produced by re-parsing rich's HTML export, `depscan-risk-<type>.json`, `license-<type>.json` and properties such as `depscan:insights`. Reachability evidence is reduced to sets of purls after each analyzer's report is converted into an atom-shaped slice. The orchestration is one function of about 600 lines (`run_depscan` in `depscan/cli.py`), and prioritisation lives in a 2,100-line `analysis_lib/utils.py`.

v7 adds several new kinds of result. Each needs a place to go that the console, HTML, SARIF, VEX, the MCP server and the skills can all read. [Decision 0002](decisions/0002-depscan-v7-one-finding-model.md) sets the model; this plan builds it.

## Scope

The finding and evidence schemas, the analyzer adapters, the scan pipeline and Python API, the scan directory, SARIF output and the VDR writer. The exploitability grade that fills a finding's axes is in [depscan-v7-exploitability.md](depscan-v7-exploitability.md). The human renderers are in [depscan-v7-reports-docs.md](depscan-v7-reports-docs.md).

## Approach

Schemas are written as JSON Schema 2020-12 files in the repository, versioned, and printed by `depscan schema <name>`. Python types are generated from them or checked against them in tests, so the schema and the code cannot drift. The fingerprint is computed from the rule or advisory id, the component purl without version, the relative file, the enclosing function and the API or symbol names on the evidence path. It contains no line numbers, so it survives unrelated edits and can be used for baselines and suppressions, the same way cdxgen's planned memory-safety contract (cdxgen/.github `plans/plugins-memory-safety.md`) defines its fingerprint.

## Deliverables

### D1. Finding schema

Tracked in [dep-scan#586](https://github.com/owasp-dep-scan/dep-scan/issues/586).

`depscan-finding.schema.json` with: `id` (the fingerprint), `category` (vulnerability, malware, memory-safety, supply-chain, binary-hardening, mobile, code, license, crypto, health), `rule` (advisory id or rule id), `title`, `severity`, `confidence`, `cwes`, `components` (purls), `locations` (file, line, column, function, binary offset), `evidence` (ids into the evidence store), `coverage` (`assessed` or `not_assessed` with reasons), `exploitability` (axes and grade, filled by the exploitability plan), `triage` (state, justification, author, expiry), `fix` (fixed versions, upgrade path), `producer` (name and version) and `references`. Include the fingerprint algorithm, with test vectors. Done when the schema validates the examples for every category, the fingerprint test vectors pass, and `depscan schema finding` prints it.

### D2. Evidence store and analyzer adapters

Tracked in [dep-scan#587](https://github.com/owasp-dep-scan/dep-scan/issues/587).

An evidence schema and `analysis_lib/evidence/`, with one adapter per producer: atom (usages, callsites, reachables, OpenAPI `x-atom-usages`), golem, rusi, dosai, kosi (through the `cdx:kosi:*` properties and its raw report), blint, cdxgen (`--bom-audit` annotations, occurrences, callstacks, introspection) and tracebom. Each adapter turns a native report into evidence records and is the only code that knows that report's format. It checks the producer's schema version and refuses an unknown major version with a not-assessed reason instead of guessing. Evidence kinds include negative evidence (a sanitizer that stopped a flow, dead code, a symbol absent from a binary). Caveats the producer reports (budgets, timeouts, collapsed edges, unresolved call sites, coverage gaps) are kept on every record and on every summary built from it. Records are written to `evidence.jsonl` in the scan directory with an index by component and by finding.

The reachability analyzers read evidence instead of the converted atom-shaped slices, and `golem_slices.py`, `rusi_slices.py` and `dosai_slices.py` are removed. The dosai adapter reads `ExternalCallGraphNode` entries, which the 6.x converter drops ([dep-scan#561](https://github.com/owasp-dep-scan/dep-scan/issues/561)); fix that bug on `release/6.3.x` as well. Done when every adapter has tests on recorded reports from the current release of its producer, reachability results on the repotests repositories match or beat 6.3.2 with differences explained, and #561 has a regression test.

### D3. Scan pipeline and Python API

Tracked in [dep-scan#588](https://github.com/owasp-dep-scan/dep-scan/issues/588).

Split `run_depscan` into stages with one function each: inventory (BOM generation or loading), enrichment (evidence and analyzer runs), matching (vulnerability lookup), analysis (each check pack), grading, decisions (triage and policy) and rendering. Each stage reads and writes the scan directory, so a later stage can be re-run without repeating earlier ones (`depscan report` re-renders, `depscan vex` re-decides). Expose them as a documented Python API in `depscan.api` (`scan`, `load_scan`, `query`, `explain`, `diff`), which the CLI, the server and the MCP server call. Move prioritisation and insight logic out of `analysis_lib/utils.py` into modules named for what they do, and remove the three duplicated analyzer blocks in `depscan/lib/bom.py`. Done when the CLI, server and tests go only through `depscan.api`, no function in `depscan/cli.py` is longer than 80 lines, and the public API has docstrings and a reference page.

### D4. Scan directory and manifest

Tracked in [dep-scan#589](https://github.com/owasp-dep-scan/dep-scan/issues/589).

Every scan writes one directory with a documented layout: `depscan.manifest.json` (dep-scan version, inputs with SHA-256 digests, tool versions and capabilities from `depscan doctor`, database flavor and coverage, the stages that ran and their status, options used), `depscan.findings.json`, `evidence.jsonl`, the BOMs and VDR, and each analyzer's raw report under `analysis/<producer>/`. The layout and manifest schema are versioned, and every renderer and integration reads only from here. Done when the schema is published, a scan directory written by one dep-scan release can be read by `depscan report` from the same major version, and the manifest of a scan where an analyzer was missing names the capability and reason.

### D5. SARIF 2.1.0

Tracked in [dep-scan#590](https://github.com/owasp-dep-scan/dep-scan/issues/590).

Write SARIF natively for every finding category ([dep-scan#31](https://github.com/owasp-dep-scan/dep-scan/issues/31)), replacing the contrib Jinja templates. Each rule appears once with its help text, CWE tags and default level. Results carry `partialFingerprints` from the finding id, `codeFlows` from evidence paths, `suppressions` from triage decisions, and locations relative to `%SRCROOT%`. Findings without a source location (a vulnerable package, a binary hardening gap) point at the manifest or binary that declares them. The output validates against the SARIF 2.1.0 schema and stays within GitHub code scanning's upload limits, splitting into several runs when needed. Done when a scan of a test repository uploads to GitHub code scanning in CI and shows the expected alerts, and a suppressed finding appears as suppressed.

### D6. VDR writer on the finding model

Tracked in [dep-scan#591](https://github.com/owasp-dep-scan/dep-scan/issues/591).

Write the CycloneDX VDR from vulnerability and malware findings, keeping the 6.x contract (affects, ratings, advisories, fix recommendations) and adding `depscan:*` properties for the exploitability grade, axes, unassessed axes, rule id and evidence ids. Write `analysis` from triage decisions and the VEX mapping in [depscan-v7-compliance.md](depscan-v7-compliance.md) D1. Keep full evidence paths out of the VDR; they stay in the evidence file. Validate the VDR against CycloneDX 1.7 and 1.6. Done when the VDR of a test scan validates with `cdx-validate`, and consumers of the 6.x VDR (Dependency-Track import, the 6.x property names that stay) are tested.

## Risks and open questions

- The pipeline refactor (D3) touches most of the code base. It should land early and in small pull requests, each keeping the suite green, before the feature plans build on it.
- Fingerprints for vulnerability findings have no function or file, so two copies of the same vulnerable package in a monorepo get the same id unless the manifest path is part of it. D1 must decide and document this with an example.

## Outcome

Filled in when the work ends.
