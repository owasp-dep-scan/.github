# dep-scan v7: binaries, containers and firmware

| | |
| --- | --- |
| Epic | dep-scan#TBD |
| Owner | @prabhu |
| AI use | Auto |
| Corpus | `depscan-v7.corpus.json` (binaries, images and one firmware image) |

## Why

Much of the software people need to assess arrives as binaries: container images, vendor installers, drivers, standalone executables and firmware. dep-scan 6.x hands binaries to blint 3.2 for an SBOM and matches the SBOM against vdb. It does not report what blint finds about the binary itself, does not use blint 4's Windows, installer and link-closure analysis, cannot tell whether a library listed in an image's package database is actually loaded by anything, and misses Go binaries in containers ([dep-scan#314](https://github.com/owasp-dep-scan/dep-scan/issues/314)). Container scans also report false positives from OS package data ([dep-scan#239](https://github.com/owasp-dep-scan/dep-scan/issues/239)).

blint 4 analyses ELF, PE (including .NET metadata), Mach-O, WASM, APK, AAB and IPA files, MSI, MSIX, CAB, ClickOnce and nupkg installers, and Office macros. It checks hardening, flags known vulnerable Windows drivers from the loldrivers and Microsoft blocklist data, recovers driver IOCTL handlers, resolves an ELF's dynamic link closure the way the loader would, identifies embedded runtimes (Node.js, Bun, .NET) and decoders without version banners, matches functions against blintdb, and diffs two builds. cdxgen 13.9 and blint are adding UEFI firmware.

## Scope

dep-scan's use of blint 4 for binaries, installers, drivers and images, binary evidence for the Presence axis, build-to-build diffs and firmware images. What blint itself must fix or add is in [blint-depscan-v7.md](blint-depscan-v7.md). Hardening findings are in [depscan-v7-memory-safety.md](depscan-v7-memory-safety.md) D5; mobile apps in [depscan-v7-mobile.md](depscan-v7-mobile.md).

## Deliverables

### D1. Binary and installer analysis

`depscan scan <binary|installer|directory>` with `--checks binary` runs blint's analysis through its Python API and reports blint's findings and reviews as `binary-hardening`, `supply-chain` or `code` findings by rule, with blint's coverage mapped to not assessed. Known vulnerable Windows drivers become findings with high pressure; packed binaries, unsigned or badly signed binaries and suspicious capabilities are reported by blint's rules ([dep-scan#106](https://github.com/owasp-dep-scan/dep-scan/issues/106) for executables in images). Installers (MSI, MSIX, ClickOnce, nupkg) are unpacked by blint and their contents scanned. Done when a test set of an ELF, a PE driver from the loldrivers list, a Mach-O binary and an MSI produces findings from each with blint's rule ids, and blint's coverage gaps show as not assessed.

### D2. Container images and root filesystems

For an image or rootfs, combine cdxgen's OS package inventory with blint's analysis of executables not owned by any package: Go binaries (from build info), Rust binaries (cargo-auditable data), embedded runtimes and standalone executables ([dep-scan#314](https://github.com/owasp-dep-scan/dep-scan/issues/314)). Run blint's link-closure resolution against the unpacked image (`BLINT_RESOLVE_LINK_CLOSURE` with `BLINT_LINK_ROOT` set to the image root, never the scan host), so each shared library is known to be loaded by some executable or not. Re-check the OS false-positive reports ([dep-scan#239](https://github.com/owasp-dep-scan/dep-scan/issues/239)) under vdb 7's distro feeds. Done when an image with an unpackaged Go binary reports its modules and advisories, and the link closure of a distroless image lists which libraries are loaded.

### D3. Presence corroboration from binaries

For components found in a package database or a source manifest, look for corroborating binary evidence: an exported or imported symbol, an exact or similarity blintdb match, the library in a link closure, or a vendored-source banner. Map the result to the Presence axis: an exact symbol or blintdb match is `proven`, a similarity match or banner is `likely`, a library that is installed but loaded by nothing is `unlikely` with that evidence, and an unavailable blintdb layer is `not_assessed`. Done when an image where a vulnerable library is installed but unused grades lower than one where it is linked, and the explanation cites the link-closure evidence.

### D4. Build-to-build binary diff

`depscan diff` on two binaries or two images runs blint's diff (metadata, hardening regressions, findings paired across rebuilds, functions when both sides were disassembled, and the Windows signer, driver, IOCTL and P/Invoke layer) and reports regressions as findings a release gate can select: a hardening flag lost, a new capability, a new dependency, a signer change. Done when two builds of a test program, one with RELRO turned off, produce a diff that fails a policy selecting hardening regressions.

### D5. Firmware images (Horizon Next)

Read firmware BOMs from cdxgen (`-t firmware`, planned for 13.9.0 with the romi helper, [cdxgen#4620](https://github.com/cdxgen/cdxgen/issues/4620)) and blint's UEFI module components ([blint#264](https://github.com/owasp-dep-scan/blint/issues/264)). Match EDK II modules by the identifiers cdxgen's firmware audit plan defines (`pkg:github/tianocore/edk2@<tag>#<Package>/<Module>` and the EDK II CPE), report cdxgen's firmware rules (network boot stack present, PKfail keys, stale dbx, debug builds, unsigned capsules), and grade them. EDK II advisories are repository security advisories that vdb may not ingest; check and open a vulnerability-db issue if needed. The acceptance case shared with cdxgen is a scan of an Ubuntu `QEMU_EFI.fd` that reports CVE-2026-97631 and the PixieFail CVEs against `NetworkPkg/UefiPxeBcDxe`. Done when that case passes.

## Risks and open questions

- blintdb images are large (6 to 78 GB). D3 must work without them, with the blintdb layer recorded as not assessed, and the documentation must say which images help which targets.
- Link closure on an image needs the image unpacked to disk, which costs space and time. It runs under the `deep` profile only, unless the user asks for it.

## Outcome

Filled in when the work ends.
