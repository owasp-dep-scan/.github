# dep-scan v7: mobile apps

| | |
| --- | --- |
| Epic | dep-scan#TBD |
| Owner | @prabhu |
| AI use | Auto |
| Corpus | `depscan-v7.corpus.json` (open-source Android and iOS apps) |

## Why

A mobile app is a bundle of other people's code: SDKs for analytics, ads and payments, native libraries for each CPU architecture, cross-platform runtimes such as Flutter and React Native, and the app's own bytecode. Reviewers want to know which SDKs and trackers it carries, which bundled libraries have advisories, what the app can reach from an exported component or a deep link, and whether it meets store and privacy requirements. dep-scan 6.x sends an APK to blint 3.2 for an SBOM and nothing more, and cannot scan an IPA.

blint 4 parses APK, APKM, AAB, DEX and IPA files, reviews Dalvik bytecode for dynamic code loading, reflection, native command execution, weak cryptography and cleartext traffic, detects tracker and service SDKs, identifies native frameworks per ABI and joins DEX to native code through JNI, checks Android hardening (BTI, PAC, MTE, 16 KB pages), and reviews iOS privacy manifests, required-reason APIs, App Transport Security and fingerprinting. atom 4 builds callsites and reachables from APK and DEX with `--app-prefix` for the app's own package, and kosi reads Android manifests, components and deep links.

## Scope

Scanning Android and iOS apps as a target kind, the findings and report sections for them, reachability inside the app, and a mapping to the OWASP MASVS. Source projects of mobile apps are scanned like any other source project.

## Deliverables

### D1. App target and mobile profile

`depscan scan app.apk` (and `.aab`, `.apkm`, `.apks`, `.xapk`, `.ipa`) detects the app, chooses the `mobile` profile and runs blint's deep analysis (DEX parsing and disassembly), cdxgen's inventory of the app, and atom's APK callsites with `--app-prefix` derived from the manifest package unless the user sets it. The help text says deep mode is slow and needs the full install. Done when an open-source Android app and an open-source iOS app from the corpus produce a BOM, findings and a mobile report section, and a missing Android SDK or Apple tool shows as not assessed.

### D2. Android findings

Report the bundled SDK and tracker inventory, blint's Dalvik behavioural reviews, Android hardening of each native library per ABI (including the 16 KB page alignment Google Play requires for apps targeting Android 15 and later), advisories in native libraries joined to the DEX code that loads them, and exported components and deep links as exposure (from kosi or blint's manifest data). Done when a corpus app with a known vulnerable bundled native library, a tracker SDK and an exported activity reports all three, and the native library finding names the ABI.

### D3. iOS findings

Report privacy manifest posture (missing `PrivacyInfo.xcprivacy`, undeclared required-reason API use), App Transport Security exceptions, entitlements, fingerprinting and cross-app tracking reviews, and advisories in embedded frameworks (for example a bundled OpenSSL or SQLite). Done when a corpus IPA with an embedded framework that has an advisory and an ATS exception reports both, and an app without a privacy manifest is flagged.

### D4. Reachability inside apps

Use atom's APK callsites and reachables to decide whether a vulnerable SDK or library is called from the app's own code, and from which component. Obfuscated apps limit this: with an R8 or ProGuard `mapping.txt` supplied, names can be restored once atom supports it ([atom#360](https://github.com/AppThreat/atom/issues/360)); without one, calls into obfuscated code are not assessed. atom's planned move to SootUp for bytecode will change method names and improve lambda handling, so the adapter must not depend on lambda name shapes. Done when a vulnerable SDK method called from an exported component grades higher than the same SDK linked but unused, on a corpus app.

### D5. MASVS mapping

Map each mobile finding to the OWASP MASVS v2 control groups it bears on (storage, crypto, auth, network, platform, code, resilience, privacy), and add a report view grouped by those controls. The mapping lives in a data file with a schema, so it can be reviewed and corrected. Done when every blint mobile rule dep-scan reports has a mapping or an explicit "no MASVS control", and the report view shows the grouping.

## Risks and open questions

- Deep analysis of large apps is slow and memory hungry. Budgets per stage must be recorded in the manifest when they cut analysis short.
- Commercial apps cannot go in the corpus. The corpus uses open-source apps (for example from F-Droid) whose licences allow redistribution.

## Outcome

Filled in when the work ends.
