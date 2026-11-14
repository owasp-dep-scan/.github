# dep-scan v7: corpus, quality gates and the 7.0.0 release

| | |
| --- | --- |
| Epic | dep-scan#TBD |
| Owner | @prabhu |
| AI use | Auto |
| Corpus | `depscan-v7.corpus.json`, created by D1 |

## Why

A security tool is judged by its misses and its noise, and both can only be measured against inputs whose answers are known. dep-scan 6.x has real-repository workflows (`repotests.yml` and the lifecycle tests) that check that scans run, and a reachability evaluation that compares results with its own previous snapshot ([dep-scan#553](https://github.com/owasp-dep-scan/dep-scan/issues/553)). It has no labelled corpus for any category, and a backlog of false-positive reports, some from 2023, that nobody has re-checked against current data.

## Scope

The ground-truth corpus for every finding category, the scorer and CI gates, the false-positive backlog, and the 7.0.0 release itself. The reachability labels from the callsites ground truth are planned in [depscan-callsites.md](depscan-callsites.md) D2 and reused here.

## Deliverables

### D1. Ground-truth corpus

`plans/depscan-v7.corpus.json` in this repository, following cdxgen's corpus manifest schema: for each item, where it came from, its SHA-256, its licence and where the bytes are stored. Items cover each finding category: projects with known reachable and unreachable vulnerable dependencies (from the callsites ground truth), C and C++ code with labelled memory-safety defects and negative controls, supply-chain fixtures (a malicious version, an unpinned vulnerable action, a model download, a bidirectional control character), binaries and images with known contents, open-source Android and iOS apps, and one firmware image. Each item lists the findings expected and the findings that must not appear. Done when the manifest validates against the schema and every item can be fetched and checked by digest.

### D2. Scorer and gates

A scorer in `contrib/` that runs dep-scan over corpus items and reports precision and recall per category and per item, comparing against the labels. A small subset runs in CI on pull requests that touch analysis code, failing on any finding on a negative control or a drop below recorded floors; the full corpus runs on a schedule and before each release. When the scorer is wrong, fix the scorer, never the product. Done when the CI subset is a required check for analysis paths and the scheduled run publishes its results as an artifact.

### D3. False-positive and data-quality backlog

Re-check each open false-positive and data-quality report against v7 with vdb 7: [dep-scan#446](https://github.com/owasp-dep-scan/dep-scan/issues/446), [#385](https://github.com/owasp-dep-scan/dep-scan/issues/385), [#353](https://github.com/owasp-dep-scan/dep-scan/issues/353), [#316](https://github.com/owasp-dep-scan/dep-scan/issues/316), [#315](https://github.com/owasp-dep-scan/dep-scan/issues/315), [#299](https://github.com/owasp-dep-scan/dep-scan/issues/299), [#290](https://github.com/owasp-dep-scan/dep-scan/issues/290), [#286](https://github.com/owasp-dep-scan/dep-scan/issues/286), [#250](https://github.com/owasp-dep-scan/dep-scan/issues/250), [#442](https://github.com/owasp-dep-scan/dep-scan/issues/442) and [#482](https://github.com/owasp-dep-scan/dep-scan/issues/482). For each, fix it in dep-scan, fix the data in vulnerability-db, add it to the corpus as a negative control, or close it with an explanation of why the result is correct. Done when each issue has an answer and the ones that were real defects have regression tests or corpus entries.

### D4. Release 7.0.0

Publish pre-releases from `master` (7.0.0a1 onwards) once the foundation, findings and CLI plans have landed, so early users can try the new command line. Before the final release: the full corpus run, the migration guide, release notes that list breaking changes first, the 6.3.x support statement (security and data-correctness fixes until a date the maintainers set), and an announcement in the dep-scan discussions. Items with Horizon Next that are not done move to the 7.1.0 milestone. Done when 7.0.0 is on PyPI, GHCR, the standalone-binary release and Homebrew, with the attestations and VEX from the distribution plan D5.

## Risks and open questions

- Some ground truth (the callsites labels) lives in a private harness in AppThreat. D1 needs it published or reproducible from a manifest; [depscan-callsites.md](depscan-callsites.md) tracks the same risk.

## Outcome

Filled in when the work ends.
