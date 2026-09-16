BETA BRANCH RESEARCH - CPU LOAD FIX VERIFICATION
================================================

This folder is for comparing the beta-branch CPU fix against the known
pre-fix findings from the main CPU research programme.

Scope
-----
- Check whether the beta build changed the underlying CPU/tick model or only
  suppressed one symptom.
- Re-run the earliest, highest-signal calibration tests first.
- Record each run in the same format as the main research programme.
- Keep the analysis strictly inside the game model: no real-machine OS
  explanations, no unsupported speculation.

Priority order
--------------
1. T01 - clock and external-instrument validation
2. T02 - tick ruler / base tier measurement
3. T03 - bare-script noise floor
4. T04 - get_cpu() response and quantization
5. T06 - kill catchability / engine-level termination
6. T11 - lethal burst boundary and repeated-burst variance

Method
------
- Run the pre-fix script exactly as originally written.
- Record raw output verbatim.
- Compare against the archived baseline in:
  blackdoor-dashboard/Blackdoor documentation/Bug Reports/Investigation/CPU Research/results.txt
  blackdoor-dashboard/tests/README.txt
- If behavior changed, classify it as one of:
  - changed root cause
  - changed symptom only
  - no material change
- If behavior is unchanged, keep the beta branch investigation anchored to
  the old findings instead of inventing a new explanation.

Output protocol
---------------
- One record per run in results.txt, following the same fields as the main
  programme.
- Keep a short verdict in this README or a per-run notes file if the beta
  branch materially changes the model.
- Update the main CPU research index only when the beta branch result alters
  the interpretation of earlier findings.

Status
------
INVESTIGATION STATUS: CONCLUDED 2026-09-14. This folder's active comparison
work stopped here - see CLOSING NOTE at the end of this file before resuming
it, rather than re-deriving the state from scratch.

Current status: T01, T02, T03, T04, T06, T07, T08, T09, T10, and T11 completed
on beta branch (all in one continued session for T04 onward - see results.txt
for the exact session boundaries). The core clock/tick behavior still matches
the pre-fix baseline for T01-T04, but T06 retains the same non-catchable kill
shape while showing a lower pre-burst CPU reading than the archived result,
and T07, T08, T09, and T10 all diverge materially from the archived
main-branch results: beta shows a starting floor at 0 instead of ~10, a
different phase ladder for the same short-gap probes, and a lower/equal-zero
post-burst residue pattern than the main branch, with the same patterns
reproduced across fresh runs. T10 additionally shows zero sub-ceiling "dip"
readings across 60 samples and zero extended-survival runs across 10 (vs the
main branch's ~19% extended-survival rate tied to those dips), plus a ~18%
faster per-iteration cost through the identical death-trace loop body.

T11 is the most consequential result so far: the lethal burst boundary itself
moved down (beta's ladder dies at n=18, where the main branch survived n=18
and only died at n=21), and n=18 in beta's hold-mode runs is now a clean,
reproducible step function (survive twice, die on the third repeat, 3/3 times)
rather than the stochastic "same burst size both survives and dies" pattern
the main branch found at that boundary. Most importantly, the specific
accumulation mechanism the main-branch programme used to explain menu.bd's own
crash-time variance - repeated near-lethal bursts (n=17) silently building a
hidden danger that an apparently-adequate settle/throttle does not clear, so
every main-branch hold-mode attempt at n=17 eventually died - did NOT
reproduce on beta across 50 consecutive held repeats at that same n=17, with
zero deaths and rock-steady burst timing throughout, despite running in a
session that already carried 10 prior overload kills from the T10 batch
immediately before it.

Current verdict: the beta branch has now shown a real, multi-test model
difference across T06 through T11, not a single-symptom cosmetic change. Two
of those differences point in opposite directions for menu.bd risk: the
lethal boundary sits LOWER (less margin per burst), but the SPECIFIC
hidden-accumulation mechanism identified as the leading explanation for
menu.bd's unpredictable crash timing has not been reproduced at all on beta so
far. This is not yet a settled "beta fixed the variance mystery" conclusion -
see results.txt's T11 entry for the caveats (only one 50-repeat n=17 hold-mode
run so far, no settle=8 beta variant yet, n=18's new step-function pattern not
yet re-tried after a genuine fresh game boot). Recommended next beta steps, in
priority order: (1) repeat the n=17 hold-mode run 1-2 more times, ideally after
a genuine fresh boot, to confirm the accumulation mechanism is actually gone
and not just not-yet-triggered; (2) a beta T10 PART B (control, no get_cpu()
call) run to decompose the ~18% per-iteration speedup found in T10; (3) only
then return to the original priority list's remaining items if the T11
recheck holds up.

CLOSING NOTE (2026-09-14): the investigation was stopped here, at the verdict
above - NONE of the three recommended next steps were run before closing. That
means the T11 finding (accumulation mechanism at n=17 not reproduced) is
CORROBORATED-BUT-NOT-CONFIRMED: real and reproducible within this session's
data, but resting on a single 50-repeat run rather than the 3-attempt pattern
main branch required before trusting it. If this folder is picked up again
later, do not treat "beta fixed the variance mystery" as settled - start from
the recommended-next-steps list above rather than treating T01-T11 as the
final word. This closure does NOT by itself say anything about whether
menu.bd development should resume; that is a separate decision layered on top
of this data, not a consequence automatically implied by it - see
`external/CLAUDE.md`'s Resume status / Current focus sections for whatever was
separately decided about that, since this file's job is only to record what
beta showed, not what to build in response to it.
