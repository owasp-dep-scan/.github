# blint for dep-scan v7

| | |
| --- | --- |
| Epic | blint#TBD |
| Owner | @prabhu |
| AI use | Auto |
| Corpus | none |

## Why

dep-scan v7 moves from blint 3.2 to blint 4 and uses it for binaries, installers, drivers, images, Android and iOS apps (see [depscan-v7-binaries.md](depscan-v7-binaries.md) and [depscan-v7-mobile.md](depscan-v7-mobile.md)). Four things in blint stand in the way.

The default blintdb image for linux-amd64, `ghcr.io/appthreat/blintdb-vcpkg:v4` (`blint/config.py`), does not exist on ghcr, and neither do `blintdb-vcpkg-musl:v4` and `blintdb-meson-musl:v4`; the meson and arm64 images do. A blintdb lookup on the most common CI platform therefore fails. blint's Python API (`blint/api.py`) covers analysis of a single file, but not SBOM generation or diffs, so dep-scan calls internal functions (`run_sbom_mode`, `BlintOptions`, `blintdb_setup`) that carry no compatibility promise, and reviews have no stable ids to suppress or baseline by. Sanitizer and CFI detection runs only for Android ELF files, and x86 CET and stack-clash facts are not collected for any ELF. And 52 commits on `main` since 4.0.1 are unreleased.

## Scope

blint changes that dep-scan v7 needs, released as blint 4.1.0. UEFI firmware support is planned separately in [blint-uefi-firmware-images.md](blint-uefi-firmware-images.md).

## Deliverables

### D1. Default blintdb images that exist

Publish the missing `:v4` images or change the defaults in `blint/config.py` to images that exist, for every platform the defaults cover. Add a CI check that requests the manifest of every default image URL and fails when one is missing, so this cannot recur unnoticed. Done when `blint sbom --use-blintdb` on linux-amd64, linux-amd64 musl, linux-arm64 and darwin-arm64 downloads its default image, and the CI check is required.

### D2. A stable API for SBOMs, diffs and reviews

Add `blint.api.sbom(...)` and `blint.api.diff(...)` entry points with typed results, accept a list of files or a directory, and give each review a stable id the way findings have `finding_id`. Document which functions are covered by semantic versioning, and keep the global lock out of callers' way where it is not needed. Done when dep-scan's `xbom_lib/blint.py` can be written against `blint.api` alone, the API reference lists every public function, and the tests call only the public API.

### D3. Memory-safety facts for every ELF

Report sanitizer instrumentation (ASan, HWASan, UBSan, TSan, MSan) and Clang CFI for all ELF files, not only Android ones, and add x86 CET (IBT and shadow stack from `GNU_PROPERTY`), stack-clash protection and the FORTIFY level where it can be determined. Done when test binaries built with and without each option are told apart, and the new facts appear in metadata and in the hardening checks that dep-scan reads.

### D4. Release blint 4.1.0

Remove the unused `oras` dependency, check the 52 unreleased commits, and release 4.1.0 to PyPI, Homebrew and the container images, with D1 to D3 included. Done when the release is published and dep-scan's foundation plan D7 can pin it.

## Risks and open questions

- Publishing missing blintdb images needs the blint-db builders to run for those platforms; if a platform cannot be built soon, change its default to a smaller image that exists and say so in the release notes.

## Outcome

Filled in when the work ends.
