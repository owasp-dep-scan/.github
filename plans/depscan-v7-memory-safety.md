# dep-scan v7: memory safety

| | |
| --- | --- |
| Epic | https://github.com/owasp-dep-scan/dep-scan/issues/570 |
| Owner | @prabhu |
| AI use | Auto |
| Corpus | `depscan-v7.corpus.json` (C and C++ memory-safety fixtures with negative controls) |

## Why

Memory-safety defects (out-of-bounds writes, use after free, integer overflows that size an allocation) are still the largest class of exploited vulnerabilities in native code, and most of them are in code that never gets a CVE before it is exploited. A dependency scanner that only reports known advisories says nothing about them, even though they sit in the same C and C++ libraries it reports on, in native extensions of Python, Node.js and Ruby packages, in cgo and `-sys` crates, and in every binary a container ships.

atom 4.0 and chen 4.0 ship `atom memory-safety`, which checks C and C++ code for 37 rules in the bounds, integer and conversion, allocation-state, null, escape, format-string, initialisation, invalidation, TOCTOU, division, shift, pointer and overlap families (rule ids `MS-BOUND-001` to `MS-OVERLAP-001`, each with a CWE and a default confidence). It writes JSON or SARIF with a flow for each finding, and can use the EDG front end through edga with a fallback to CDT. Memory-safety findings for Rust (rusi), Go (golem) and Kotlin and JVM native boundaries (kosi) are planned in cdxgen-plugins-bin with a shared finding contract. blint already checks binaries for the hardening that limits what a memory-safety bug can do. dep-scan v7 brings these together, ranks them by whether untrusted input can reach them, and connects them to the advisories it already reports.

## Scope

Running and ingesting memory-safety analysis for C and C++ source, for native code inside dependencies, and from the cdxgen plugins; binary hardening posture; correlation with advisories; and a corpus gate. dep-scan does not add memory-safety rules; rules belong in chen and the plugins.

## Approach

`atom memory-safety` refuses to reuse an atom built by another command, so dep-scan runs it as a separate step with its own atom file, under `ATOM_TIMEOUT`, when `--checks memory-safety` is on and the project has C or C++ sources. cdxgen does not run it, by design, so dep-scan calls atom directly, with an argument list built in code. edga is used when `depscan doctor` finds it (atom's container images include it). A compilation database makes results much better; dep-scan finds an existing `compile_commands.json`, can export one from CMake when the user allows a configure step, and otherwise uses atom's `--auto-defines`.

## Deliverables

### D1. C and C++ memory-safety scan

Tracked in [dep-scan#613](https://github.com/owasp-dep-scan/dep-scan/issues/613).

Run `atom memory-safety -l c|cpp --format json` (and SARIF for pass-through) with `--c-frontend edg-fallback` when edga is present, `--compile-commands` when a compilation database exists, and `--min-confidence medium` by default (configurable). Map each finding into a `memory-safety` finding: rule id, CWE, confidence, severity, message, and the flow as an evidence path. Attribute each file to a component: files under vendored paths that cdxgen marks (`cdx:vendored:path`, `cdx:cmake:sourceDir`) belong to that component's purl, everything else to the application. A timeout or a file atom could not parse makes the affected files not assessed. Done when the C and C++ fixtures in the corpus produce their labelled findings at the default confidence, the negative controls produce none, and a vendored library's findings carry its purl.

### D2. Native code inside dependencies

Tracked in [dep-scan#614](https://github.com/owasp-dep-scan/dep-scan/issues/614).

Many packages carry C or C++ sources: Python sdists and wheels with extension modules, node-gyp packages in `node_modules`, Ruby gems with native extensions, Rust `-sys` crates and Go modules with cgo. With `--checks memory-safety` and `--memory-safety-deps`, find those sources in installed dependency trees (cdxgen's occurrences and the package manager's install location tell where), run D1 on each package directory with a per-package time budget, and attribute findings to the package purl. Done when a fixture project that installs a package with a known memory-safety defect in its C extension reports it against that package, and the per-package budget is recorded when it cuts a run short.

### D3. Unsafe boundary inventory

Tracked in [dep-scan#615](https://github.com/owasp-dep-scan/dep-scan/issues/615).

golem, rusi and kosi already report where code leaves memory-safe territory: Go `unsafe` and `native-interop` signals and cgo, Rust `unsafe-code` and `native-interop`, and kosi's JNI, JNA and foreign-function boundaries. cdxgen writes them as `cdx:golem:*`, `cdx:rusi:*` and `cdx:kosi:*` properties today. Report a per-component inventory of these boundaries (count, kinds and locations) as informational `memory-safety` findings, so users can see which dependencies carry unsafe code before full findings exist. Done when the Go, Rust and Kotlin test projects show their unsafe boundaries per component, and the inventory can be filtered out by policy.

### D4. Rust, Go and Kotlin memory-safety findings (Horizon Next)

Tracked in [dep-scan#616](https://github.com/owasp-dep-scan/dep-scan/issues/616).

Once rusi, golem and kosi produce memory-safety findings ([cdxgen-plugins-bin#170](https://github.com/cdxgen/cdxgen-plugins-bin/issues/170)) and cdxgen maps them to `cdx:<tool>:memorySafety:*` properties and a SARIF file beside the BOM ([cdxgen#4777](https://github.com/cdxgen/cdxgen/issues/4777)), read the raw findings (`memory_safety_findings` from rusi, `memorySafetyFindings` from golem and kosi) through the adapters, using the contract's fingerprints. Done when each plugin's golden findings from plugins-bin appear as dep-scan findings with their rule ids (`RUSI-MS-*`, `GOLEM-MS-*`, `KOSI-MS-*`) and fingerprints.

### D5. Binary hardening posture

Tracked in [dep-scan#617](https://github.com/owasp-dep-scan/dep-scan/issues/617).

Report blint's hardening checks as `binary-hardening` findings for every binary dep-scan analyses: PIE, NX, RELRO, stack canaries, FORTIFY, RPATH, writable and executable segments, Windows ASLR, DEP and CFG, Mach-O hardened runtime, and on Android BTI, PAC, MTE and 16 KB pages. Feed the same facts into the Barrier axis of [depscan-v7-exploitability.md](depscan-v7-exploitability.md), so a memory-safety advisory in a hardened binary is distinguished from one in an unhardened binary. Sanitizer, CFI and CET facts for non-Android ELF files need blint work ([blint-depscan-v7.md](blint-depscan-v7.md) D3). Done when the `gobintests.yml` binaries and a Windows test binary show hardening findings, and the Barrier axis cites them.

### D6. Correlation with advisories and reachability

Tracked in [dep-scan#618](https://github.com/owasp-dep-scan/dep-scan/issues/618).

Connect memory-safety findings to the rest of the report in two directions. When an advisory's CWE is a memory-safety class and a memory-safety finding sits in the affected component (or the affected function when a symbol hint exists), mark the advisory's Presence as corroborated and link the finding. When a memory-safety finding in first-party code is on a path from an entry point (atom reachables and the exposure data from the exploitability plan), raise its grade; when no entry point reaches it, say so. Done when a fixture with a vulnerable vendored function called from a request handler ranks above the same defect in unreachable code, and the explanation shows both the finding and the path.

### D7. Fingerprints and corpus gate

Tracked in [dep-scan#619](https://github.com/owasp-dep-scan/dep-scan/issues/619).

atom's JSON has no fingerprint, function or purl yet; [atom#413](https://github.com/AppThreat/atom/issues/413) asks for them, aligned with the plugins contract. Until it ships, dep-scan computes a line-free fingerprint from the rule, relative file, enclosing function (from the flow) and API names. Add a small C and C++ fixture set to dep-scan's tests, with labelled findings and negative controls, whose licences allow redistribution, and run it in CI: any finding on a negative control fails the run, and recall below the recorded floor fails it too. Done when the CI job runs on every pull request that touches the memory-safety code, and a baseline built from fingerprints survives a reformatting commit.

## Risks and open questions

- The low-confidence rules (null dereference, integer length overflow, leaks) are noisy on large trees. The default stays at medium; users who want more choose it.
- A second atom build doubles the analysis cost for C and C++ projects. If atom gains a way to emit memory-safety findings from the same build as usages and reachables, dep-scan should use it; [atom#413](https://github.com/AppThreat/atom/issues/413) asks for that too.
- D2 depends on installed dependency trees. A scan without installs reports native dependencies as not assessed rather than clean.

## Outcome

Filled in when the work ends.
