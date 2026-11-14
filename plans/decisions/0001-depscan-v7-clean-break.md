# 0001. dep-scan v7 is a clean break from v6

Status: accepted, 2026-11-14. Plans: [depscan-v7-overview.md](../depscan-v7-overview.md).

## Context

dep-scan 6.x keeps vulnerability-db on the 6.x line, cdxgen on the 12.x line and blint on 3.2 to avoid breaking users within a major version. Every one of those upstreams has since shipped a major release that dep-scan needs: vdb 7 for coverage and shards, cdxgen 13.8 for atom 4 and its callsites and memory-safety commands, blint 4 for apps, drivers and the Python API. The v6 command line is one command with about sixty flat flags, several of them in mutually exclusive groups that no longer make sense. Python 3.10 reached end of life in October 2026.

## Decision

v7 drops compatibility with v6 where keeping it would hold the design back:

- `master` becomes the v7 line at `7.0.0.dev0`. A `release/6.3.x` branch is cut from 6.3.2 and receives security and data-correctness fixes only.
- The command line moves to subcommands (`depscan scan`, `depscan vdb`, `depscan report` and so on). A v6-style invocation such as `depscan --src . --reports-dir out` exits with code 2 and prints the equivalent v7 command, so a CI job that breaks says how to fix itself. The `depscan-vdb` and `depscan-validate` console scripts are removed; their work moves to `depscan vdb` and `depscan validate`.
- vulnerability-db 7 is required. `.vdb6` databases are not read, and the v6 image names are not used.
- cdxgen 13.8 or later is required for generation. There is no fallback to the 12.x line, because its atom 2.5 lacks callsites and memory safety.
- blint 4 is required for binary and app analysis.
- Python 3.11 is the minimum.
- Report contracts change: the VDR gains `depscan:*` properties, every scan writes a scan directory with a manifest and a findings file, and exit codes follow a documented table.

## Consequences

Users must change their CI invocations, re-download the vulnerability database and expect more findings, because vdb 7 no longer drops application advisories by year and matches malicious packages at every version. The migration guide planned in [depscan-v7-reports-docs.md](../depscan-v7-reports-docs.md) D6 lists every removed flag with its replacement. Users who cannot move stay on 6.3.x, which keeps receiving fixes for security problems and wrong results.
