# dep-scan and atom's callsites slice

| | |
| --- | --- |
| Epic | https://github.com/owasp-dep-scan/dep-scan/issues/555 |
| Owner | @prabhu |
| AI use | Auto |
| Corpus | [callsites-corpus.corpus.json](https://github.com/AppThreat/.github/blob/main/plans/callsites-corpus.corpus.json) in AppThreat/.github |

## Why

dep-scan decides which dependencies a project really uses from the `evidence.occurrences` that
cdxgen writes into the BOM. cdxgen is moving those occurrences from atom's `usages` slice to the
`callsites` slice, which records every external use site of every atom language with much higher
recall and precision (see the cdxgen plan below). Two things in dep-scan stand in the way.

`set_slices_args` in `packages/xbom-lib/src/xbom_lib/cdxgen.py` passes cdxgen the paths of the
deps, usages, data-flow, reachables and semantics slices, but not callsites, so the slice lands at
cdxgen's default path and is not kept with the other reports. And the reachability evaluation script
`contrib/reachability_eval.py` compares dep-scan's output with its own previous snapshot, so it
cannot tell whether a change made reachability better or worse.

The slice is planned and built in AppThreat (`callsites-overview.md` and the language plans in
[AppThreat/.github](https://github.com/AppThreat/.github/tree/main/plans)). cdxgen's part is
[cdxgen-callsites-evidence.md](https://github.com/cdxgen/.github/blob/main/plans/cdxgen-callsites-evidence.md)
in cdxgen/.github. This plan covers dep-scan only.

## Scope

dep-scan's cdxgen invocation and its reachability evaluation. dep-scan keeps reading occurrences
from the BOM; it does not read slices for occurrences itself.

## Approach

Pass the callsites slice path the same way as the other slices, so the file is kept next to them in
the reports directory. Score the evaluation against the ground-truth corpus that the callsites work
already has: each row names a file, a line and the ecosystem package used there, which is what
`direct_purls` and reached purls can be checked against.

## Deliverables

### D1. Pass the callsites slice (owasp-dep-scan/dep-scan#552)

`set_slices_args` passes `--callsites-slices-file <bomdir>/<type>-callsites.slices.json` for a single
project type, as it does for the other slices. Done with a test beside
`test/test_cdxgen_source_path.py` and one `depscan --profile research -t python` run that leaves the
slice next to the other slices.

### D2. Real labels for the reachability evaluation (owasp-dep-scan/dep-scan#553)

`contrib/reachability_eval.py` scores `direct_purls` and reached purls against the ground-truth
corpus instead of its previous snapshot. Done when the script reports recall and precision per
project and the numbers are reproducible from the corpus manifest.

## Risks and open questions

- D1 has an effect only once cdxgen writes the slice for the language (the cdxgen plan's D1), and
  that needs an atom 4 release.
- The ground truth lives in a private harness in AppThreat; D2 needs the labels it uses to be
  published or reproducible from the corpus manifest.

## Outcome

Filled in when the work ends.
