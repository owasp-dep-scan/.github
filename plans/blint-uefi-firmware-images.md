# UEFI firmware images in blint

| | |
| --- | --- |
| Epic | https://github.com/owasp-dep-scan/blint/issues/264 |
| Owner | @prabhu |
| AI use | Pair |
| Corpus | none |

## Why

After an EDK II firmware advisory, the first question about a firmware image is whether it contains the affected module at all. blint cannot answer that today. Given a UEFI firmware image, it finds nothing to analyse.

The case that prompted this plan is [CVE-2026-97631](https://github.com/tianocore/edk2/security/advisories/GHSA-99cj-7hg9-mp4m), a critical pre-boot code execution flaw in the EDK II PXE Base Code driver, `NetworkPkg/UefiPxeBcDxe`. Its ICMP error handlers copy messages longer than 504 bytes into a fixed field and advance the destination pointer by the cumulative rather than the per-fragment length. One ICMP error sent during a network boot can therefore overwrite DXE-phase memory. Most firmware vendors build on EDK II, and the fix ([tianocore/edk2#13302](https://github.com/tianocore/edk2/pull/13302)) reaches machines only through vendor firmware updates. It is the latest in a run of EDK II network stack flaws, after PixieFail (CVE-2023-45229 to CVE-2023-45237) and the iSCSI flaw CVE-2024-38805.

We ran blint on Ubuntu's `qemu-efi-aarch64` 2024.02 image (`QEMU_EFI.fd`) and on an iPXE option ROM. Both runs reported "No issues found" and produced empty metadata and an SBOM with no components. A short proof-of-concept script then decompressed the LZMA firmware volume, found the `UefiPxeBcDxe` file by its GUID `B95E9FDA-26DE-48D2-8807-1F9107AC5E3A`, and extracted the driver. blint analysed the extracted driver correctly: an ARM64 `EFI_BOOT_SERVICE_DRIVER` whose debug path names the `edk2-2024.02` tree and `NetworkPkg/UefiPxeBcDxe`. It also reported a HIGH `CHECK_DLL_CHARACTERISTICS` finding, which concerns Windows loader mitigations and does not apply to a UEFI driver.

## Scope

The plan adds firmware image unpacking, module components in `blint sbom`, firmware-aware PE reviews, and a data file of well-known EDK II modules. It does not add vulnerability matching against version ranges. EDK II does not publish versioned releases that vendors follow, and vendor builds rarely keep the evidence needed to place a module in a release. Bootkit and implant detection inside firmware is also out of scope.

## Approach

The formats are defined in volume 3 of the [UEFI Platform Initialization specification](https://uefi.org/specifications): firmware volumes with the `_FVH` signature, firmware file system (FFS) files, and typed sections, some of them compressed or GUID-defined and wrapping further sections. Capsules add the header defined in section 23 of the [UEFI specification](https://uefi.org/specifications). Full SPI flash dumps from Intel platforms start with a flash descriptor that gives the offset of the BIOS region.

blint will parse these in pure Python, with no new heavy dependencies. LZMA uses the standard library's `lzma` module in `FORMAT_ALONE` mode, and EFI/Tiano compression needs a small decoder. The proof of concept showed that the EDK II case takes little code. The effort goes into making the parser safe with hostile input, through limits on nesting depth, decompressed size and every length field.

Each extracted module is analysed by the existing PE code. Its identity comes from what the firmware itself records: the FFS GUID, the name and version sections, and the CodeView debug path when the build left it in. Vendor images often strip the debug path, so the GUID is the identity we can always rely on.

## Deliverables

### D1. Safe unpacking of firmware volumes, capsules and flash images (owasp-dep-scan/blint#265)

blint finds and recursively unpacks firmware volumes in `.fd` files, capsules, flash dumps and option ROMs. It enforces limits, and it records any section it cannot decompress. Debian and Ubuntu OVMF and AAVMF images unpack fully, and malformed fixtures fail cleanly.

### D2. Module components with EDK II identity (owasp-dep-scan/blint#266)

`blint sbom` emits the image as a `firmware` component, with each module nested under it. A module records its GUID, file type, name, version, subsystem and hashes, plus the EDK II release and package path when the debug path provides them.

### D3. Firmware-aware PE reviews (owasp-dep-scan/blint#267)

Windows-only rules skip EFI subsystems. A small group of firmware rules covers writable and executable sections, section alignment and stripped relocations.

### D4. Well-known EDK II network modules (owasp-dep-scan/blint#268)

A data file lists the EDK II network boot modules by GUID and package path, and matching module components are tagged. The tag does not claim that the image is vulnerable.

## Risks and open questions

Vendor images from AMI, Insyde and Phoenix may use compression schemes or containers that need more than the formats above, and the first vendor samples will show how much. Firmware images are copyrighted, so the test fixtures should come from openly licensed builds, such as distribution OVMF packages or images we build ourselves from EDK II, rather than vendor downloads. Unpacking firmware adds a new attack surface to blint, which is why the parser's limits are part of D1 and not left for later.

## Outcome

To be filled in when the work ends.
