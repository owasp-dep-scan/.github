# dep-scan v7: VEX, CSAF, compliance and signatures

| | |
| --- | --- |
| Epic | https://github.com/owasp-dep-scan/dep-scan/issues/575 |
| Owner | @prabhu |
| AI use | Auto |
| Corpus | none |

## Why

Software makers now have to publish what their products contain and how each known vulnerability affects them: the EU Cyber Resilience Act asks for an SBOM, a record of exploitable vulnerabilities and how they were handled, and reporting of actively exploited ones; CISA's minimum elements set what an SBOM must carry; customers ask for VEX statements. dep-scan 6.x writes CSAF 2.0 and 2.1 with a reachability-based product status and validates CSAF offline, but it does not write CycloneDX VEX or OpenVEX on its own, cannot take triage decisions into account, does not run the CRA, CISA or SCVS benchmarks cdxgen's `cdx-validate` now has, and does not sign what it writes.

A tool can give evidence for compliance; it cannot make a product compliant. Every compliance output says so.

## Scope

VEX, CSAF, the compliance command, document signing and verification, and input and output formats. Triage decisions come from [depscan-v7-cli-policy.md](depscan-v7-cli-policy.md) D4; exploitability axes from [depscan-v7-exploitability.md](depscan-v7-exploitability.md) D1.

## Approach

VEX statements follow from evidence by a fixed table, and a triage decision by a person overrides the table with the person named:

| Evidence | CycloneDX `analysis.state` and `justification` | OpenVEX `status` and `justification` |
| --- | --- | --- |
| Presence refuted (component or vulnerable code absent) | `not_affected`, `code_not_present` | `not_affected`, `vulnerable_code_not_present` |
| Reach refuted (unreachable from every root, dead code) | `not_affected`, `code_not_reachable` | `not_affected`, `vulnerable_code_not_in_execute_path` |
| Exposure refuted (no untrusted input reaches the path) | `not_affected`, `requires_environment` or `requires_configuration`, as the evidence shows | `not_affected`, `vulnerable_code_cannot_be_controlled_by_adversary` |
| Barrier proven on every path, nothing else refuted | `in_triage`, with the barrier named in `detail` | `under_investigation` |
| Grade critical or high with a witness path | `exploitable` | `affected` |
| Any deciding axis not assessed | `in_triage` | `under_investigation` |

`in_triage` is the default. A barrier such as a sanitizer is evidence, not a guarantee, so on its own it leads to a person's decision rather than an automatic `not_affected`. A generated `not_affected` statement names its evidence ids and producers in `analysis.detail`, and a deciding axis that was not assessed can never produce one. Signing and benchmarks are delegated to cdxgen's `cdx-sign`, `cdx-verify` and `cdx-validate`.

## Deliverables

### D1. VEX from evidence and decisions

Tracked in [dep-scan#644](https://github.com/owasp-dep-scan/dep-scan/issues/644).

Implement the table, the override by triage decisions, and `depscan vex <scan-dir>`, which regenerates VEX from a saved scan when decisions change, without re-scanning. Write CycloneDX VEX (standalone and inside the VDR) and OpenVEX. Done when each table row has a test, a scan with an unassessed reach axis produces `in_triage` and never `not_affected`, and the CycloneDX output validates with `cdx-validate`.

### D2. CSAF 2.1

Tracked in [dep-scan#645](https://github.com/owasp-dep-scan/dep-scan/issues/645).

Extend the existing CSAF generator (`analysis_lib/vex/`): build `product_tree` from the BOM's dependency graph so relationships survive, set `product_status` from the VEX result, put the exploitability grade and rule in `threats`, and take `remediations` from the fix plan. Keep the offline schema and section 6.1 checks. Remove the stale `csafdemo.yml` workflow or bring it up to date. Done when the CSAF output of a corpus scan passes the existing validator, and a triaged finding shows the decision and its author.

### D3. `depscan compliance`

Tracked in [dep-scan#646](https://github.com/owasp-dep-scan/dep-scan/issues/646).

Run `cdx-validate -b cisa,cra,scvs -r json` on the input BOM and on dep-scan's VDR, and add the part that needs vulnerability data: the record of exploitable vulnerabilities and how each was handled (from findings and triage decisions). Write JSON, Markdown and SARIF, each opening with a sentence saying the output is evidence for an assessment, not a certificate. Done when a corpus scan produces a compliance report with all three benchmarks and the vulnerability section.

### D4. Signing and verification

Tracked in [dep-scan#647](https://github.com/owasp-dep-scan/dep-scan/issues/647).

Verify signatures on input BOMs with `cdx-verify --public-key` when a key is configured, and report an invalid or missing required signature as a `supply-chain` finding. Sign the VDR and VEX with `cdx-sign` (JSF for CycloneDX 1.x) when a key is configured, adding to cdxgen's signature on the BOM so a pipeline can show which stage signed what. Keys come from files or environment variables, never from the command line. Done when a test pipeline signs a BOM with cdxgen, dep-scan verifies it and counter-signs the VDR, and `cdx-verify` accepts the result.

### D5. Input and output formats

Tracked in [dep-scan#648](https://github.com/owasp-dep-scan/dep-scan/issues/648).

Accept SPDX 2.3 and 3.0 documents as input ([dep-scan#59](https://github.com/owasp-dep-scan/dep-scan/issues/59)). `cdx-convert` converts CycloneDX to SPDX but not the other way, so this needs SPDX import in cdxgen ([cdxgen#4807](https://github.com/cdxgen/cdxgen/issues/4807)), which dep-scan then calls. Write SPDX through `cdx-convert`. Keep CycloneDX 1.7 as dep-scan's default output. Reading CycloneDX 2.0 is out of scope for v7, because the schema is not ratified ([cdxgen#4594](https://github.com/cdxgen/cdxgen/issues/4594)). Done when an SPDX 2.3 and an SPDX 3.0 document scan to the same findings as the CycloneDX BOM they were made from.

## Risks and open questions

- The VEX justification for refuted exposure depends on what the analyzer actually proved. If the evidence cannot tell the two justifications apart, the statement stays `in_triage`.
- The CRA implementing acts are still being written. Benchmarks come from cdxgen and are updated there.

## Outcome

Filled in when the work ends.
