# dep-scan v7: supply-chain risk

| | |
| --- | --- |
| Epic | https://github.com/owasp-dep-scan/dep-scan/issues/569 |
| Owner | @prabhu |
| AI use | Auto |
| Corpus | `depscan-v7.corpus.json` (supply-chain fixtures) |

## Why

Most software compromises of the last few years did not exploit a known CVE. They came through a malicious or hijacked package, an install script, a build pipeline with too many permissions, an unpinned action, a package pulled from an unexpected source, or more recently a model file or an agent skill. dep-scan 6.x reports malicious-package advisories and runs a registry metadata audit for npm, PyPI and Cargo (`depscan/lib/audit.py`), and for a supplied BOM audits only npm components.

cdxgen 13 now ships 26 audit rule packs (`data/rules/*.yaml`), among them `package-integrity`, `dependency-source`, `ci-permission`, `ai-agent`, `ai-provenance`, `mcp-server`, `container-risk`, `rootfs-hardening` and `chrome-extension`, a predictive audit of upstream repositories for npm and PyPI in `cdx-audit`, Gradle verification metadata checks, and AI-BOM and MCP inventories. vdb 7 matches malicious packages at every version and includes the GitHub Actions ecosystem. atom 4 reports Unicode look-alikes, bidirectional controls and hidden code in source (`sourceIntegrity`). This plan brings all of that into dep-scan's findings, policy and reports.

## Scope

Supply-chain findings, the move from dep-scan's registry audit to `cdx-audit`, provenance and integrity checks, CI pipeline risk, AI supply-chain risk, dependency health and licence policy. Signature verification of BOMs is in [depscan-v7-compliance.md](depscan-v7-compliance.md) D4. Malicious executables in images are in [depscan-v7-binaries.md](depscan-v7-binaries.md) D1.

## Approach

dep-scan does not write supply-chain rules of its own ([decision 0004](decisions/0004-depscan-v7-delegate-to-the-toolchain.md)). It runs cdxgen's rule packs, maps their findings into the finding model with the rule id, category, severity and the components they name, and applies policy, triage and presentation. A rule that is missing or wrong is fixed in cdxgen's `data/rules`.

## Deliverables

### D1. cdxgen audit findings

Tracked in [dep-scan#606](https://github.com/owasp-dep-scan/dep-scan/issues/606).

Run `--bom-audit` with a chosen set of `--bom-audit-categories` when dep-scan generates the BOM, and `cdx-audit --direct-bom-audit -r json` when the user supplies one. Map cdxgen's annotations and JSON findings into `supply-chain` findings (and `crypto` for the `cbom-security` pack, `binary-hardening` for `rootfs-hardening`). Choose default categories per profile and document them. Done when a fixture BOM triggers at least one rule from each default category, the findings appear in every output with the cdxgen rule id, and a supplied BOM is audited for all ecosystems, not only npm.

### D2. Package risk through `cdx-audit`

Tracked in [dep-scan#607](https://github.com/owasp-dep-scan/dep-scan/issues/607).

Compare dep-scan's registry risk audit with `cdx-audit`'s predictive mode on the same projects, ecosystem by ecosystem. Where `cdx-audit` matches or exceeds it, switch to it and remove dep-scan's code; where it does not (Cargo today), open a cdxgen issue and keep dep-scan's audit for that ecosystem until it ships. Private namespaces (`--private-ns`) keep working for dependency confusion checks. Network lookups follow the offline rules in [depscan-v7-distribution.md](depscan-v7-distribution.md) D3. Done when the parity table is in the pull request, `--risk-audit` is replaced by the `supply-chain` check, and an offline scan makes no registry requests.

### D3. Malicious packages

Tracked in [dep-scan#608](https://github.com/owasp-dep-scan/dep-scan/issues/608).

Report malicious packages as `malware` findings that fail the default policy, from vdb's `MAL-` advisories (which vdb 7 matches at every version) and from cdxgen's `package-integrity` rules (install-time scripts, tarballs that differ from the repository, recently changed maintainers where the rules detect it). The re-check of the malware false-positive reports ([dep-scan#385](https://github.com/owasp-dep-scan/dep-scan/issues/385), [dep-scan#446](https://github.com/owasp-dep-scan/dep-scan/issues/446)) is tracked in [depscan-v7-quality-release.md](depscan-v7-quality-release.md) D3. Done when a fixture with a known malicious version produces a malware finding that fails the default CI policy.

### D4. Provenance and integrity

Tracked in [dep-scan#609](https://github.com/owasp-dep-scan/dep-scan/issues/609).

Report missing or invalid provenance and integrity evidence as findings: npm provenance and PyPI attestations or trusted publishing where the registry records them ([dep-scan#301](https://github.com/owasp-dep-scan/dep-scan/issues/301)), lockfile integrity hashes, Gradle verification metadata mismatches (cdxgen rule INT-016), and dependencies fetched from git, URLs or local paths (`dependency-source` rules). Where cdxgen does not collect a signal, open a cdxgen issue rather than collecting it in dep-scan. Done when each signal has a fixture, and the report shows, per ecosystem, how many components had provenance, had none, or could not be checked.

### D5. CI pipelines and build risk

Tracked in [dep-scan#610](https://github.com/owasp-dep-scan/dep-scan/issues/610).

Inventory the repository's CI workflows (GitHub Actions, and others cdxgen reads) and report: actions with advisories (vdb includes the GitHub Actions ecosystem), actions not pinned to a commit, excessive workflow permissions and untrusted input in `run:` steps (cdxgen's `ci-permission` rules), and build inputs from formulation. From cdxgen 13.9, a committed `.github/workflows/actions.lock` changes what counts as pinned: cdxgen adds `cdx:github:lock:*` and `cdx:github:action:isLocked` to locked steps, lists the transitive actions of locked composite actions as components, records offline drift between the workflows and the lockfile, stops CI-001, CI-003 and INT-012 from flagging locked references, and adds rules CI-025 to CI-028 for unlocked actions, workflows the lockfile leaves out, reusable workflows on mutable refs and unreadable lockfiles ([cdxgen#4733](https://github.com/cdxgen/cdxgen/issues/4733)). dep-scan reports those rules as cdxgen gives them and does not judge pinning itself. Sub-action references become repository purls with a subpath (`pkg:github/actions/cache@v4#save`), so advisory matching and policy use the repository purl. Done when a fixture workflow with an unpinned, vulnerable action and `pull_request_target` misuse produces the expected findings with file and line.

### D6. AI and agent supply chain

Tracked in [dep-scan#611](https://github.com/owasp-dep-scan/dep-scan/issues/611).

Run cdxgen's AI inventory (`-t ai`, `mcp`, `ai-skill`, `agentic`) under `--checks supply-chain` when a project has AI components, and report the `ai-agent`, `ai-provenance`, `ai-governance` and `mcp-server` rules: unpinned or unverified models, MCP servers run from unpinned packages, agent instruction files and skills with risky permissions. Report atom's `sourceIntegrity` entries (Trojan Source, look-alike identifiers, hidden code) as findings, and flows from `ai-model-artifact` sources to `sandbox-render` sinks. Done when a fixture project with a Hugging Face model download, an MCP configuration and a skill directory produces findings from each rule pack, and a file with a bidirectional control character is reported with its location.

### D7. Dependency health and licence policy

Tracked in [dep-scan#612](https://github.com/owasp-dep-scan/dep-scan/issues/612).

Report deprecated, archived, yanked and end-of-life dependencies as `health` findings, using registry metadata cdxgen collects and end-of-life data once vdb has it ([vulnerability-db#204](https://github.com/AppThreat/vulnerability-db/issues/204)). Move licence detection to cdxgen's licence data and keep the policy (allowed, denied and review lists, copyleft scope) in `depscan.toml`, reporting violations as `license` findings that policy gates can select. Retire dep-scan's own licence lookup where cdxgen covers it. Done when a fixture with a deprecated package and a denied licence produces one finding each, and the licence policy has a documented example.

## Risks and open questions

- Some supply-chain signals need registry or GitHub API calls. They must be off in offline mode and must not need tokens by default.
- Retiring `depscan/lib/audit.py` changes results for users who rely on its scores. The parity table in D2 decides it per ecosystem.

## Outcome

Filled in when the work ends.
