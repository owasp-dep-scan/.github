# 0002. One finding model with evidence and coverage

Status: accepted, 2026-11-14. Plans: [depscan-v7-findings.md](../depscan-v7-findings.md), [depscan-v7-exploitability.md](../depscan-v7-exploitability.md).

## Context

In 6.x each analyzer's output is converted into an atom-shaped reachables slice (`golem_slices.py`, `rusi_slices.py`, `dosai_slices.py`), and reachability becomes a set of purls. The reasons behind a verdict, the analyzer's own caveats and anything an analyzer could not assess are lost on the way. v7 adds memory-safety defects, supply-chain rules, binary hardening, mobile reviews and code-level findings, none of which are vulnerabilities with an advisory id. Without a shared model each would grow its own report table, its own suppression logic and its own JSON.

## Decision

Every result dep-scan reports is a finding of one schema, published as JSON Schema and printed by `depscan schema finding`. A finding has a category (vulnerability, malware, memory-safety, supply-chain, binary-hardening, mobile, code, license, crypto or health), a rule or advisory id, the affected component purls and locations, severity, confidence, a line-free fingerprint, the producer and its version, references to evidence records, a coverage state and a triage state.

Evidence is stored once per scan, in an append-only evidence file keyed by id. Each evidence record names its producer, its kind (occurrence, call path, taint flow, endpoint, runtime observation, binary symbol, configuration or negative evidence), its confidence by provenance (observed, proven, derived or heuristic) and every caveat the producer reported. Summaries carry the union of their inputs' caveats.

Coverage is explicit. "Not assessed" means an analyzer did not run, failed, hit a budget, or lacked data (for example a vdb shard that is missing or stale). It is a different state from "assessed and nothing found". A policy gate, a VEX statement or a report summary must never treat the first as the second. In particular dep-scan never emits `not_affected` for a finding whose deciding evidence was not assessed.

The CycloneDX VDR remains the interchange format for vulnerabilities. The findings file is the contract for everything else and is what the console, HTML, SARIF, Markdown, MCP server and skills read.

## Consequences

Analyzer adapters become the only place that knows an analyzer's report format, including version-drift workarounds. Adding an analyzer means writing one adapter. The three atom-shaped converters are retired once the reachability analyzers read evidence directly. Suppressions, baselines and policy gates work the same way for every category.
