# dep-scan v7: foundation

| | |
| --- | --- |
| Epic | https://github.com/owasp-dep-scan/dep-scan/issues/565 |
| Owner | @prabhu |
| AI use | Auto |
| Corpus | none |

## Why

dep-scan 6.3.2 runs on tool versions that are one major release behind each of its upstreams (see the table in the [overview](depscan-v7-overview.md#where-things-stand)). Every other v7 plan needs the current versions: callsites and memory safety come with atom 4 in cdxgen 13.8, coverage and staleness come with vulnerability-db 7, and app and driver analysis come with blint 4. This plan moves dep-scan onto them and adds `depscan doctor`, which tells users and agents which tools and data a scan can rely on.

## Scope

Versions, pins, API migrations, the cdxgen calling contract, and the capability report. New features built on the new versions belong to the other plans. Container images and standalone binaries pick up the pins in [depscan-v7-distribution.md](depscan-v7-distribution.md) D1 and D2.

## Approach

Follow vulnerability-db's `docs/MIGRATING_TO_V7.md` section 3, cdxgen's `docs/ENV.md`, `docs/EVINSE.md` and `docs/INTROSPECTION.md`, and blint's `blint/api.py`. Where the upstream guide is wrong or incomplete, fix the guide upstream in the same round. The vdb guide, for example, names `analysis_lib/utils.py` for `version_compare`, but the import is in `packages/analysis-lib/src/analysis_lib/helpers.py`, and it does not mention `db6` or `PLACEHOLDER_EXCLUDE_VERSION`.

Every tool version lives in one module (`depscan/lib/config.py` or a new `depscan/lib/toolchain.py`), and the Dockerfiles, standalone-binary hooks (`build/sea-hooks/verify_versions.py`) and CI read or check it, so a version is changed in one place.

## Deliverables

### D1. Open the v7 line

Tracked in [dep-scan#578](https://github.com/owasp-dep-scan/dep-scan/issues/578).

Cut `release/6.3.x` from the 6.3.2 tag for 6.x fixes. On `master`, set all five distributions (`owasp-depscan`, `ds-xbom-lib`, `ds-analysis-lib`, `ds-reporting-lib`, `ds-server-lib`) to `7.0.0.dev0` with matching inter-package pins. Raise `requires-python` to `>=3.11`, drop the 3.10 classifier and CI jobs, remove the `tomli` conditional dependency, and set pyrefly's target to 3.11. Run `ruff check` and pyrefly in CI on every pull request; today they are dev dependencies that no workflow runs. Done when `uv run pytest` passes on 3.11 to 3.14 and the lint and type jobs are required checks.

### D2. vulnerability-db 7

Tracked in [dep-scan#579](https://github.com/owasp-dep-scan/dep-scan/issues/579).

Pin `appthreat-vulnerability-db>=7.0.5,<8` without the `[oras]` extra, which vdb 7 no longer uses. Replace `from vdb.lib import db6` with `vdb.lib.db` (`depscan/cli.py`, `depscan/vdb_cli.py`, `analysis_lib/search.py`). Replace `vdb.lib.utils.version_compare` in `analysis_lib/helpers.py` with `vdb.lib.vers.vers_compare`, building scheme-aware `vers:` strings rather than generic ones. Remove uses of `config.PLACEHOLDER_EXCLUDE_VERSION` (`analysis_lib/normalize.py`, `analysis_lib/utils.py`). Treat `source_data` as a plain dict. When vdb reports an empty or unreadable database, stop with a clear error instead of reporting no findings.

Test fixtures whose counts change are re-recorded with a note per change: application advisories are no longer dropped by year, malicious packages match every version, and npm `@latest` no longer matches fixed advisories. Done when the test suite passes on vdb 7.0.5, `depscan --purl` smoke tests in `pythonapp.yml` find the expected advisories, and a scan against an empty database exits non-zero with a message that names the database path.

### D3. Database management through `vdb db`

Tracked in [dep-scan#580](https://github.com/owasp-dep-scan/dep-scan/issues/580).

Replace the direct `orasclient.download_image` calls and `resolve_vdb_image` (`depscan/lib/config.py`) with vdb's `db_cmd.refresh_full_image`, `refresh_shards` and `store_status`. The vdb flavors `full`, `app-only`, `app-extended`, `app-10y` and `app-10y-extended` replace the `--vdb-scope`, `--vdb-time` and `--vdb-extended` triple; the old flags are accepted with a warning that names the flavor until the subcommand CLI lands. When the purl types of a scan are known before matching, dep-scan can use shards for those types instead of the full database; this is opt-in through configuration in 7.0. `VDB_AUTO_FETCH` stays off, so a scan never downloads data mid-run. Refreshes keep dep-scan's retries and `VDB_AGE_HOURS` check. Done when a fresh environment can download, refresh and report status for both a full flavor and a shard set, and an interrupted download leaves the previous database usable.

### D4. Coverage and staleness as scan state

Tracked in [dep-scan#581](https://github.com/owasp-dep-scan/dep-scan/issues/581).

vdb's batch searches mark components with `coverage_gap`, and report uncovered purl types and stale shards. Record these in the scan manifest planned in [depscan-v7-findings.md](depscan-v7-findings.md) D4, and mark every affected component's vulnerability assessment as not assessed with the reason. Report the database flavor, build date and whether metadata tables are present, so features that need metadata (symbol search, full-text search) say when they cannot run. vdb's `vdb db status --json` does not yet report the flavor or metadata presence; this is [vulnerability-db#308](https://github.com/AppThreat/vulnerability-db/issues/308), and until it ships dep-scan reads `vdb.meta` directly behind a version check. Done when a scan with a missing shard and a stale shard shows both, separately, in the console and the manifest, and neither is counted as a clean component.

### D5. cdxgen 13.8 calling contract

Tracked in [dep-scan#582](https://github.com/owasp-dep-scan/dep-scan/issues/582).

Move to the `@cdxgen/cdxgen` npm package and `ghcr.io/cdxgen/*` images, pinned to an exact 13.8 release in the toolchain module. At startup, check the cdxgen version and refuse an older one with a message naming the version found and the version needed. Build cdxgen's argument list to match its current rules: evidence options, profiles and all `*-slices-file` paths go on the command line, because cdxgen no longer accepts them from configuration files. Pass `--callsites-slices-file` (this is [dep-scan#552](https://github.com/owasp-dep-scan/dep-scan/issues/552), planned in [depscan-callsites.md](depscan-callsites.md)). Write slices and reports outside the scanned tree, because cdxgen's secure mode relocates a slices file that already exists inside it. Map cdxgen's exit codes: 0 success, 1 no BOM, 4 introspection score below the gate, 5 a deferred extractor failure with a BOM still written. Run with `--introspect --introspect-json` and keep the fidelity tier and report in the scan directory. Use cdxgen's secure mode for server scans and for remote repositories.

Done when the argument list is asserted by unit tests for each profile and lifecycle, a scan of the dep-scan repository with cdxgen 13.8 produces a BOM, slices and an introspection report next to each other in the reports directory, and a cdxgen older than 13.8 is refused.

### D6. atom 4 and atom-tools 1.x

Tracked in [dep-scan#583](https://github.com/owasp-dep-scan/dep-scan/issues/583).

cdxgen 13.8 runs atom 4, whose slices changed shape: `usages` has one entry per external call site, tagged library calls in `reachables` carry the callee's purls, and atom exits 124 when `ATOM_TIMEOUT` stops it. Update `FrameworkReachability` and `SemanticReachability` (`analysis_lib/reachability.py`), the explainer's slice handling (`depscan/lib/explainer.py`) and the OpenAPI handling for atom 4 output, and bump `atom-tools` from `~=0.9.1` to the 1.x line. Files that atom's `--parse-report` lists as dropped or partly read are recorded as not assessed. Done when the reachability tests pass on slices recorded from atom 4.0.2, and the results on the `repotests-lifecycle-java` and `repotests` repositories are compared with 6.3.2 in the pull request, with every difference explained.

### D7. blint 4

Tracked in [dep-scan#584](https://github.com/owasp-dep-scan/dep-scan/issues/584).

Pin blint 4.1.0, the release planned in [blint-depscan-v7.md](blint-depscan-v7.md) D4. Move `packages/xbom-lib/src/xbom_lib/blint.py` from blint's internal functions (`run_sbom_mode`, `BlintOptions`, `blintdb_setup`) to `blint.api`, which in 4.1.0 has `analyze`, `analyze_many`, `sbom` and `diff` with typed errors such as `BlintdbUnavailableError` ([blint#272](https://github.com/owasp-dep-scan/blint/issues/272)); a `BlintdbUnavailableError` becomes a not-assessed reason for the blintdb layer. Pass the cdxgen BOM to blint as `src_dir_boms`, so blint overlays source evidence. Respect blint's `internal:version_source`, so an assembly version is never matched as a package version. Done when the `gobintests.yml` binaries and an Android APK produce SBOMs and findings with blint 4, and the blintdb lookup works on linux-amd64.

### D8. `depscan doctor`

Tracked in [dep-scan#585](https://github.com/owasp-dep-scan/dep-scan/issues/585).

A command that reports, as text or JSON, every tool and data source a scan may use: cdxgen, atom, edga, the cdxgen plugins and their platforms, blint and its blintdb images, dosai and the .NET runtime it needs, the vulnerability database flavor, shards, age and metadata presence, and the Python and Node.js runtimes. For each v7 capability (for example `memory-safety:c`, `reachability:dotnet`, `mobile:android`) it says whether it can run here and why not. Scans record the same report in the manifest, and every analysis step checks it before running, so a missing tool becomes a not-assessed reason instead of an exception. Done when `depscan doctor --format json` validates against its published schema on Linux, macOS and Windows runners, and removing a plugin binary changes exactly the capabilities that depend on it.

## Risks and open questions

- blint 4.1.0 still names `ghcr.io/appthreat/blintdb-vcpkg:v4` as the default blintdb image for linux-amd64, which does not exist on ghcr ([blint#271](https://github.com/owasp-dep-scan/blint/issues/271)). D7 can pin 4.1.0, but its blintdb check on linux-amd64 waits for that fix.
- Shards by default (D3) would cut download size for single-ecosystem scans a great deal, but a mixed project could miss a type. 7.0 keeps it opt-in; a maintainer decides the default for 7.1.

## Outcome

Filled in when the work ends.
