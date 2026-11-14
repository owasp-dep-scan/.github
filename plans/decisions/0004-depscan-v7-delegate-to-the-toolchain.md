# 0004. Delegate to the toolchain, fix gaps upstream

Status: accepted, 2026-11-14. Plans: [depscan-v7-overview.md](../depscan-v7-overview.md).

## Context

dep-scan 6.x implements its own registry risk audit, licence lookup, BOM validation and vulnerability database download, and converts analyzer reports itself. Meanwhile cdxgen 13 ships `cdx-audit` and `--bom-audit` rule packs, `cdx-validate` with CISA, CRA and SCVS benchmarks, `cdx-sign` and `cdx-verify`, `cdx-convert` and introspection. vulnerability-db 7 ships `vdb db` with staged, validated, atomic refreshes. Two implementations of the same thing drift, and users get different answers from tools of the same family.

## Decision

dep-scan owns orchestration, vulnerability matching, exploitability grading, policy, triage decisions and presentation. It delegates:

- BOM generation, evidence, audit rules, validation benchmarks, signing and conversion to cdxgen and its binaries;
- vulnerability data, database download, shards and coverage to vulnerability-db;
- program analysis to atom, the cdxgen plugins (golem, rusi, kosi), dosai and blint.

dep-scan runs these tools as subprocesses or through their Python APIs, with argument lists built in code, never through a shell, and reads their documented outputs. When dep-scan needs something a tool does not provide, or a tool's output is wrong, the fix is an issue and a pull request in that tool's repository. A workaround in dep-scan is allowed only behind a named feature check, with a link to the upstream issue, and is removed when the upstream release ships.

dep-scan's own registry risk audit (`depscan/lib/audit.py`) is retired once `cdx-audit` reaches parity for the ecosystems dep-scan audits today.

## Consequences

dep-scan depends on exact versions of its tools, checked at startup by `depscan doctor`, and its container images pin them. Upstream release timing affects dep-scan's plans; the overview lists each dependency and marks work that waits on one with Horizon Next.
