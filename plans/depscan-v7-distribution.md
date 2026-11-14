# dep-scan v7: distribution, server mode and operations

| | |
| --- | --- |
| Epic | dep-scan#TBD |
| Owner | @prabhu |
| AI use | Auto |
| Corpus | none |

## Why

dep-scan reaches users as PyPI packages, two container images (AlmaLinux 10 and 9), standalone binaries for Linux, musl, macOS and Windows, and a Homebrew tap. In 6.3.2 the images and CI install `@cyclonedx/cdxgen` with no version, so they get whatever the 12.x line publishes, while the standalone binaries pin cdxgen 12.9.1 and the code expects 12.9.1 images. None of them carry atom 4, edga or the cdxgen plugins at known versions. Air-gapped use is supported but not tested with the network off. Server mode lacks reachability, CSAF, filters and database freshness checks that the command line has.

v7 needs every distribution to carry the same pinned toolchain, to work offline when asked, and to be as verifiable as the software it scans.

## Scope

Container images, standalone binaries, Homebrew, offline operation, server mode and dep-scan's own release supply chain.

## Deliverables

### D1. Container images

Rebuild `depscan` and `depscan-al9` on the pinned toolchain from [depscan-v7-foundation.md](depscan-v7-foundation.md): `@cdxgen/cdxgen` at the pinned 13.8 release, atom 4 with edga, cdxgen-plugins-bin at the version cdxgen pins, blint 4, dosai's self-contained build (which needs no .NET runtime), vulnerability-db 7 and Python 3.12 or later. Install npm packages with `--ignore-scripts` where they allow it. Run `depscan doctor` as the image smoke test and fail the build when a capability the image is meant to have is missing. Publish an SBOM and a build provenance attestation for each image. Done when both images pass the smoke test on amd64 and arm64, and the attestations verify with `gh attestation verify`.

### D2. Standalone binaries and Homebrew

Update the four standalone-binary workflows to bundle cdxgen 13.8 and keep `build/sea-hooks/verify_versions.py` checking that the bundled versions match the toolchain module. Automate the Homebrew formula update on release. Done when each binary runs `depscan doctor` and a `--purl` scan in its workflow, and the tap formula updates from the release workflow.

### D3. Offline and air-gapped operation

`--offline` (and the `offline` profile) guarantees no network access: no registry lookups, no vdb auto-fetch, cdxgen without installs or remote metadata, blintdb only if already present, and an error rather than a silent skip when a requested check needs the network. Document how to prepare an offline bundle (database image, blintdb images, container image) and how to refresh it. Test it in CI by running a scan in a container with networking disabled. Done when that CI job passes and a check that needs the network reports itself as not assessed with the reason.

### D4. Server mode

Rebuild `depscan serve` on the pipeline and scan directories: submit a scan as a job, poll its status, fetch the findings, SARIF, VDR, VEX or HTML, and delete it. Keep every 6.x protection (API key, host and path allowlists, cross-site and DNS rebinding checks, private URL blocking, size caps, Windows path checks) and add per-key rate limits and a database freshness check. Reachability and every check pack work the same as on the command line, with the server's own allowed checks and profiles. Done when the server tests cover every 6.x protection again plus the job API, and an end-to-end test runs a reachability scan through the server.

### D5. dep-scan's own release supply chain

Sign release artifacts and publish attestations for the wheels, images and standalone binaries; publish an SBOM for each standalone binary; and publish a VEX document for each release made with `depscan vex` on dep-scan's own scan, so dep-scan's users can check dep-scan the way it checks other software. Done when the 7.0.0 release carries attestations and a VEX document, and the release notes say how to verify them.

## Risks and open questions

- Images that carry atom 4, edga, the plugins, blint and .NET are large. Consider a slim image without the native analyzers for vulnerability-only scans, if users ask for it.

## Outcome

Filled in when the work ends.
