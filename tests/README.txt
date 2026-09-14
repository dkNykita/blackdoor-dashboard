TESTS - index
=============
One entry per script in this folder: tested goal, current progress,
result. Kept in sync whenever a test is added, re-run, or its result
changes - see field_notes.txt for the full narrative/reasoning behind
each one; this file is just the compact status view.

CPU LOAD RESEARCH PROGRAMME (T01-T28): the authoritative plan, ordering,
methodology, recording protocol and per-experiment reference lives in
../Blackdoor documentation/Bug Reports/Investigation/CPU Research/README.txt
- read that first (it lives under the bug report it serves,
Investigation/01_get_cpu_run_to_run_inconsistency.txt). The T01-T28
scripts in this folder implement it. This file tracks their per-script
Result: as they are run; the programme file is the campaign. Run results
get appended to that same folder's results.txt (one record per run).
The four test_cpu_decay/decode_pacing/token_dispatch/dispatch_after_imports
scripts predate the programme (they came out of the frozen menu.bd
investigation) and are NOT part of T01-T28.

2026-09-14 CROSS-CUTTING FIX (before any test but T01 was run in-game):
T01's first real run crashed immediately - "Line 163: Cannot convert
'"10"' to int." Root cause: `@param ... default="10"` (quoted) embeds
literal quote characters into what get_param() returns; the workflow
subagents that authored T01-T28 didn't know this project-specific quirk
(menu.bd's utils/net.bd and utils/sqli.bd already defend against it).
Fixed across the whole set in one pass: 23 quoted @param defaults ->
unquoted (11 files: T01,T03,T10,T11,T12,T13,T19,T20,T24,T25,T26), and
32 get_param() call sites (15 files, including T22/T22_bomb/T27/T28
which had no quoted @param but were guarded anyway) now chain
`.replace("\"", "").trim()`. Full writeup: field_notes.txt's VERIFIED
VS DOCS. None of the per-script entries below have been re-audited
individually for this - treat every T01-T28 script's param handling as
fixed as of this date regardless of what its own entry says.

ROUND 2 (same day, re-run after the above): unquoting `default=10`
exposed that get_param() then returns an actual NUMBER (not a string)
for that default - the plain `.replace(...)` from round 1 crashed with
`'number' has no method 'replace'`. Re-fixed all 32 call sites to
`str(get_param(...)).replace("\"", "").trim()` (str() first, matching
utils/sqli.bd's normalize_path() - stronger than utils/net.bd's
resolve_raw_target(), which skips the str()). Lesson: get_param()'s
return type is NOT guaranteed to match the inline fallback argument's
type - always str() it before any string method.

T01.bd  (T01, CPU-load research programme - NOT a menu.bd test)  [DONE]
  Goal: validate the ONLY instrument the whole programme depends on -
  get_unix_time() - against two independent clocks (a physical
  stopwatch, the `time` command) before trusting any interval this
  investigation has ever recorded or will record. Checks: is
  get_unix_time() really the Unix epoch or an in-game clock wearing
  that name; does it return an int (per web_api.txt) or a float; is
  line pacing really ~100ms/line at base tier or tick-quantised; does
  in-game elapsed time match real wall-clock time.
  Progress: ALL 8 STEPS COMPLETE. T01 is the first fully-finished test
  in the programme.
  Result for STEP 1 (2/2 repeats in): raw_clock_t0 was a real Sept-2026
  Unix epoch value WITH a fractional part on both runs (1789377462.328,
  1789377693.794) - CONFIRMS get_unix_time() is genuinely epoch-based,
  and directly confirms the long-suspected doc mismatch (web_api.txt
  says get_unix_time() -> int; it returns a float). Granularity probe
  landed in the "~0.10 with jitter (0.09-0.11)" bucket on BOTH runs
  (0.101/0.099/0.101/0.099, then 0.100/0.100/0.099/0.101) - CONFIRMS
  ~100ms/line pacing at base tier twice over, REFUTES tick-quantisation
  (never exactly 0.1 to full precision) and REFUTES the "far below 0.1"
  full-speed-burst theory. Interval vs stopwatch: run 1 diff=0.339s
  (within the test's 0.5s threshold), run 2 diff=0.501s (1ms OVER it -
  essentially a tie). Both intervals are shorter than their matching
  stopwatch reading, same direction both times - consistent with a
  small, fairly stable human reaction-lag bias (~0.4s/press budgeted by
  the test itself), NOT with the alternate "differ by >1.5s" scaled-time
  signature, which neither run comes remotely close to. VERDICT: no
  evidence in-game time is scaled/simulated, but the margin is tight
  enough that STEP 3-4 data (below) is the tighter check.
  Result for STEP 3: bare `time T01 10` fails ("Unknown command: t01",
  lowercased) regardless of spacing/parens/comma syntax - `time` only
  wraps real builtin verbs, not a bare script name. `time run T01 10`
  (the exact fallback the script's own header suggests) worked first
  try - CONFIRMS `time` is a usable external instrument (the "no
  instrument exists" branch is refuted), and confirms `run <script>
  <args>` as the general syntax for composing a script invocation with
  another terminal command.
  Result for STEP 3+4 together: diff (time minus total_from_first_line)
  was +1.700s at blocks=10 and +1.701s at blocks=20 - 1 MILLISECOND
  apart despite the in-game interval nearly doubling (27.4s -> 54.4s).
  CONFIRMS the overhead is FIXED (process start/`run`-dispatch cost
  paid before the script's own t0 line ever runs), and DECISIVELY
  REFUTES "time and get_unix_time() run on different clocks" - a
  proportional relationship would have shown roughly double the diff
  at blocks=20. `time`'s ~1.70s constant is now a known correction, not
  an open question.
  Result for STEPS 5-6 (2/2 repeats): `time` printed NOTHING for either
  killed run - not a duration, not a zero, no line at all where runs
  5-6's "(time: X.XXXs)" was. CONFIRMS (twice) the "no external
  instrument for time-of-death" branch, REFUTES the hoped-for "a
  duration is printed for a killed script" branch - the documented "5
  straight seconds" kill-delay claim is STILL unverified and needs a
  different method (see T06/T11). Side finding: burst_start (pure fixed
  bookkeeping before the lethal burst, no field data or loops over host
  records involved) was +4.301s and +4.300s on the two repeats - 1ms
  apart, essentially identical under idle conditions. Cross-reference
  for the still-open menu.bd variance mystery: a data-free fixed tick
  sequence reproduces to ~1ms here, so that variance is NOT a general
  noise floor in the tick/clock system - it's specific to menu.bd's own
  code path.
  Result for STEPS 7-8 (bg/fg concurrency): ps/jobs show a STATUS column
  beyond PID+NAME ("running (bg)"), but it's a state flag, not a cpu/
  load/tick number - no per-process CPU accounting exists, aggregate
  get_cpu() is still all we get. Concurrency itself: FG's interval while
  a BG job (blocks=25) was alive (27.407s) matched the solo baseline
  (27.394-27.401s) within 6-13ms; BG's own interval (67.906s) matched a
  linear model fit from two EARLIER solo runs (predicted 67.9135s at
  blocks=25) within 7.5ms, despite FG running concurrently with it for
  part of its duration. Both directions DECISIVELY REFUTE "concurrent
  processes share one tick budget" for this load shape - rules out a
  stray simple background script as an explanation for the menu.bd
  variance mystery (does not rule out heavier concurrent I/O).
  T01 OVERALL VERDICT: the instrument is sound - get_unix_time() is
  real-epoch, float-returning, ~100ms/line-accurate at base tier, not
  scaled/simulated, and unaffected by ordinary concurrent processes.
  `time` is usable for completed runs (constant +1.70s overhead to
  subtract) but blind to killed ones. Every interval this investigation
  has ever recorded stands as originally understood - the run-to-run
  variance mystery is NOT an instrument artifact. See CPU
  Research/results.txt (runs 1-10, full history) and field_notes.txt
  for all promoted findings.

test_cpu_decay.bd
  Goal: characterize get_cpu()'s actual behavior over time in isolation
  (does it decay to near-zero at rest, or plateau at a nonzero floor?),
  and whether a small unpaced burst with NO function/API calls (plain
  arithmetic only) is enough to CPU-overload on its own.
  Progress: run once, completed both phases.
  Result: idle phase (sleep(1)+log() pairs) held a rock-steady cpu=10.0
  indefinitely, no crash risk. Burst phase (50 plain arithmetic lines,
  zero calls, zero pacing) CRASHED the script before it could even log
  the post-burst reading. CONCLUSION: CPU overload is universal per-line,
  not tied to function/API calls - refuted the "+10t needs a function
  call" hypothesis.

T02.bd  (T02, CPU-load research programme - NOT a menu.bd test)  [DONE]
  Goal: THE TICK RULER. One question the whole investigation had always
  assumed but never checked: does the engine actually BLOCK a script
  once a second's tick budget is spent (is elapsed wall-clock time a
  linear readout of executed-statement count), and if so what's this
  machine's real tick rate? Five phases (0/10/20/30/0B unit lines of
  `a = a + 1`, bracketed by get_unix_time() only) with ZERO get_cpu()
  calls anywhere, so the measurement can't be contaminated by get_cpu()
  itself adding load.
  Progress: COMPLETE, 2/2 runs (well within the 15% agreement threshold
  - no 3rd run needed).
  Result: block= rose monotonically 0.10/1.10/2.10/3.10s across arms
  0/10/20/30 in BOTH runs, matching the base-tier prediction almost
  exactly - CONFIRMS the engine blocks on tick exhaustion (wall-clock
  time IS a linear work meter) and DECISIVELY REFUTES "engine doesn't
  block" and "not linear." sec_per_unit = 0.09993s (run 1), 0.10003s
  (run 2), 0.09998s combined -> tick_rate = 10.00-10.01 ticks/sec both
  ways of computing it - this machine is BASE TIER, now measured
  directly rather than assumed. Drift control (PHASE-0B vs PHASE-0)
  matched within 2.5ms both runs; integrity check (`a`'s value after
  every phase) passed with zero deviation both runs; RULER-CLOCK raw
  carried decimals both times (float clock, third independent
  confirmation after T01). This directly confirms, for the first time
  with zero get_cpu() involvement, the "1 line = 1 tick = 100ms at base
  tier" model this whole investigation has used since day one. See CPU
  Research/results.txt (T02 runs 1-2) and field_notes.txt's CPU LOAD
  section for the promoted finding.

T03.bd  (T03, CPU-load research programme - NOT a menu.bd test)  [DONE]
  Goal: BASELINE NOISE FLOOR. The control this investigation never ran:
  the simplest possible workload (no imports, no I/O, no throttle(), no
  data-dependent branching, exactly one get_cpu() call taken BEFORE
  timing starts), run 8 times back to back, to give every later
  experiment an error bar and test whether ambient engine noise alone
  could explain the central mystery's 2.7x crash-point variance.
  Progress: COMPLETE, all 8 required repeats done in one sitting.
  Result: DETERMINISTIC. All 40 burst= samples (8 runs x 5 bursts each)
  spread only 0.78% around a 3.19995s mean; per-run means spread only
  0.144%; DONE totals across all 8 independent runs (~4.6 min of
  wall-clock) spread only 0.023%. cpu0=0 on every single run - no
  creeping residual load across 8 consecutive launches. This lands
  squarely in the "tight within runs AND across runs -> deterministic
  tick delivery" bucket, an order of magnitude inside even that
  bucket's own threshold, and DECISIVELY REFUTES both "ambient per-tick
  jitter" and "per-run state sets a speed for that whole run" (the
  shape that would have matched the central mystery). THE MOST
  IMPORTANT RESULT SO FAR: whatever causes menu.bd's real +40.3s ->
  +111.0s variance is NOT generic engine/tick noise - it has to come
  from something this probe deliberately excluded (repeated get_cpu()
  polling, throttle() looping, a live connection, file I/O, imports, or
  string/dict building over real data). T03 is now the reference
  dataset T14-T19 and T20-T25 get judged against. Side finding: this
  session's `daemons` command returned "requires Daemon Source -
  purchase it from the shop first" - "no planted daemons" has never
  actually been independently verifiable in this investigation, only
  assumed (the tool to check isn't owned). See CPU Research/results.txt
  and field_notes.txt for the full writeup.

T04.bd  (T04, CPU-load research programme - NOT a menu.bd test)  [DONE]
  Goal: METER RESPONSE FUNCTION. Is get_cpu() a linear tick counter
  (constant points per executed line) or a geometric-saturation EMA,
  how many meter points does one line add, and does get_cpu()'s own
  line get charged to the reading it returns? PART A/A2 (rapid vs
  filler-spaced back-to-back reads), PART C (ambient floor, 3s-spaced),
  PART B (a paced 0/1/3/5/8/12/20/26-line staircase, settled with
  sleep(4) before each burst).
  Progress: COMPLETE, 2/2 runs (protocol specifies no conditional 3rd
  run for this test; audit f=80 valid on both).
  Result: EVERY reading in both runs is an exact multiple of 10 (the
  odd-looking "39.9999999999989" etc. are plain float rounding error
  for 40) - strong reinforcement of the standing "meter quantizes to
  10s" observation. Neither named model fits: Model L (constant step
  per line) is REFUTED (N=1, N=3 both read 0, not a rising 10/30); the
  named geometric-saturation alternative is also REFUTED (no shrinking-
  residual pattern, N=1/N=3 don't start high). HEADLINE SPLIT: PART A
  (rapid back-to-back reads, no settle) is NOT run-to-run reproducible -
  its one nonzero value read 40 in run 1 and 30 in run 2, a full
  quantum-of-10 apart. PART B (paced, sleep(4) settle before every
  burst) is PERFECTLY reproducible at all 9 checkpoints in both runs,
  right down to a genuinely surprising non-monotonic dip: N=8 (cpu=10)
  reads LOWER than N=5 (cpu=40) despite doing MORE work, identically in
  both runs. get_cpu() is deterministic when called after a proper
  settle period and unstable/order-sensitive when polled rapidly in
  succession - itself a real, reproducible property of the instrument,
  and a plausible thread for why this codebase's own get_cpu()-polling
  code (throttle(), refresh_status()) might behave inconsistently even
  on identical workloads. The N=8 dip is a new, unexplained open
  question - a future paced test isolating N=6/7/9/10 one at a time
  could localize where it starts and ends. See CPU Research/results.txt
  (T04 runs 1-2) and field_notes.txt for the full writeup.

T08.bd  (T08, CPU-load research programme - NOT a menu.bd test)  [DONE]
  Goal: METER MEMORY II - does get_cpu() decay from saturation as a
  short step, a geometric/EMA decay, or a slow linear drain, and does
  that decay explain the real crash traces' ~3.7s throttle() cost? One
  clean (unpolled) post-burst sample per phase at k=0,3,1,8,2,5,0,2
  seconds delay (scrambled order + 2 replicates as drift controls),
  plus a P9 throttle-probe that inlines nu.throttle()'s exact shape to
  measure the wait directly.
  Progress: COMPLETE, 2/2 runs (shapes agreed exactly, no 3rd run
  triggered). Run BEFORE T07 per T07.bd's own precondition #3, which
  flags the CPU Research README's listed session order (T07, T08, T09)
  as inverted.
  Result: NONE of the three named models fits cleanly. Sorted by k:
  100 (k=0) -> 80 (k=1) -> 10/floor (k=2 through k=8), reproduced to
  10+ decimal digits across both runs. This REFUTES the slow linear-
  drain model (predicted ~9s to clear; actual clears by ~3.5s since
  burst start) and REFUTES a constant-ratio geometric decay (residual-
  above-floor goes 90->70->0, not a fixed ratio). BEST FIT: a
  rectangular window roughly 2.0-2.5s long, with partial credit where
  the burst's own 1.5s duration straddles the window edge (explaining
  the intermediate 80 at k=1 before a full drop to floor at k=2).
  HEADLINE: the P9 throttle-probe (an isolated burst + one inlined
  throttle()-shaped poll loop) measured wait=1.90s, NOT the ~3.7s real
  crash traces show - roughly HALF. The isolated decay-then-single-poll
  pattern does NOT explain the real throttle() cost on its own; T04's
  finding (get_cpu() is unstable when polled rapidly without a settle,
  exactly what throttle()'s own repeated polling does) is the leading
  candidate for the missing ~1.8s. Side finding: a 15-line unrolled
  burst is confirmed SUB-LETHAL (survived cleanly both runs) - new data
  point for future burst sizing in this programme. See CPU
  Research/results.txt (T08 runs 1-2) and field_notes.txt for the full
  writeup.

T07.bd  (T07, CPU-load research programme - NOT a menu.bd test)  [DONE]
  Goal: METER MEMORY I - window edge vs decay, sub-second resolution.
  ARM A (gap AFTER a fixed 6-line burst, scrambled d=0.05..4.00 plus a
  late A-REP replicate) and ARM B (gap INSIDE the burst, g=0.05..3.00)
  cross-check each other and a built-in A1-vs-B1 comparison tests
  whether tick order or within-run drift affects the reading - a check
  the test's own author explicitly ties to the central variance
  mystery. Run AFTER T08 (T07.bd's own precondition #3 corrects the
  session plan's listed order). Tier confirmed via neofetch/sysinfo:
  Mk.I, 10 tps, matching T02 exactly - no ladder rescaling needed.
  Progress: COMPLETE, 2/2 runs (shapes agreed exactly, no 3rd run
  triggered).
  Result: CALSLEEP validity check passes (sub-second clock confirmed,
  matches T01). ARM A sorted by d is NOT monotonic in either direction
  (0,60,80,80,40,80,40,~0,0 for d=0.05..4.00) - fits none of the three
  named ARM-A shapes cleanly. ARM B shows a genuine edge: flat at 40
  through g=0.50, floor by g=1.50, staying flat (B3=B4, a hard edge),
  roughly consistent with T08's ~2.0-2.5s window estimate. B1-B3=40=
  exactly 4x T04's unit - per the test's own rule this points to a
  sleep line's tick being charged ON ENTRY, not exit.
  HEADLINE: the test's built-in A1-vs-B1 check fires. A1 (first d=0.05
  reading, early, t~20s) reads 0; B1 (same 0.05s gap, different
  arrangement, t~94s) reads 40; A-REP (SAME arrangement as A1, but
  late, t~127s) ALSO reads 40, matching B1 rather than its own earlier
  twin. This rules out pure arrangement/order as the explanation and
  points at real within-run drift tied to absolute time/tick-history,
  not local d/g value. CRITICALLY: every one of these "erratic" values
  reproduced EXACTLY across both fully independent runs - this is NOT
  noise, it is a fully deterministic function of something more complex
  than d or phase order alone (most likely absolute elapsed time or
  cumulative tick history, both themselves fully deterministic per
  T02/T03/T08). REFRAMES the leading theory for the central mystery:
  engine-level randomness in get_cpu() looks increasingly unlikely
  (refuted again here, as in T04/T08, whenever the exact same sequence
  repeats); it is far more plausible that different real host data
  shifts the exact timing/tick-count at which menu.bd's own get_cpu()
  calls land, and if get_cpu()'s true update rule is this sensitive to
  exact history (as demonstrated here), a small data difference could
  produce a wildly different but individually reproducible reading -
  explaining the "same code, different run, different crash point"
  puzzle without requiring true randomness. See CPU Research/
  results.txt (T07 runs 1-2) and field_notes.txt for the full writeup.

T09.bd  (T09, CPU-load research programme - NOT a menu.bd test)  [DONE]
  Goal: does get_cpu() clamp at 100 and DISCARD everything above it, or
  does a hidden accumulator keep counting past the ceiling so a bigger
  burst leaves more residue behind? Four unpaced straight-line bursts
  (10/25/40/48 `x = 1` statements, no calls, no pacing), each with an
  identical instrument: a pre-burst baseline capture (cb), a delay-0 peak
  capture (c0), a delay-2s residue capture (c2), and burst_s = elapsed
  seconds for the burst itself. Secondary outputs: the first real
  bisection of the lethal UNROLLED burst size (the famous "fatal 50" was
  50 loop ITERATIONS ~ 150 executed lines, never 50 lines), an empirical
  ticks/sec calibration from four (lines, seconds) points, and a
  launch-time cpu reading on line 1 as a file-length control (334-line
  file vs the 47-line file that previously read cpu=0 at launch).
  THE UNPACED BURST IS THE EXPERIMENT - a kill is a result, not a
  failure; phases ascend so the last "T09 begin B=K" line names the
  killer exactly.
  Progress: COMPLETE, 3/3 runs (2 for the ceiling comparison, the 3rd
  free variance data - protocol calls for 2 more if a kill ever
  happens, but none did).
  Result: TEXTBOOK, UNAMBIGUOUS CLAMP CONFIRMATION - the cleanest result
  the whole programme has produced. base=10, peak=100, res=10 for EVERY
  burst size (10/25/40/48 lines) in EVERY run, matching to 10+ decimal
  digits (B=10's residue even reproduces the exact same tiny float-noise
  offset, 10.0000000003128, in all 3 runs). Peak is pinned at exactly
  100 even for the SMALLEST burst (B=10) - refutes "not saturating."
  Residue is IDENTICAL regardless of burst size - decisively refutes
  the DEBT model (a 48-line burst leaves no more residue than a 10-line
  one). cb never climbs across phases - refutes carryover/contamination.
  None of the 3 runs killed on ANY burst, including B=48 (the sharpest
  possible probe of the literal "50 executed ticks = 5s@100%" rule) -
  the central mystery did NOT reproduce in this simplest possible
  system; the ceiling behavior itself is rock-solid. Every in-script
  CPU guard (throttle(), cpu_guard_fast(), every "while get_cpu() >
  threshold" loop) can be trusted not to be blind to "how far over" a
  burst went. Side findings: launch_cpu=0 on this 334-line top-level
  SCRIPT (by far the longest test script run so far) DISMISSES the
  concern that a top-level script's own file length costs launch
  ticks the way an imported module's does; burst_s gives a 5th
  independent tick-rate calibration (10.007 ticks/sec, agreeing with
  T02 within 0.05%) and confirms burst-length-to-time is cleanly
  LINEAR from 10-48 lines with a small (~0.10s) fixed overhead, no
  superlinear cost. The user's informal live-meter observation during
  run 3 showed far more intermediate fluctuation than the script's
  3-point sampling captures - qualitatively consistent with T07/T08's
  finding that the true signal evolves continuously between samples.
  See CPU Research/results.txt (T09 runs 1-3) and field_notes.txt for
  the full writeup.

T06.bd  (T06, CPU-load research programme - NOT a menu.bd test)  [DONE]
  Goal: KILL CATCHABILITY - is the "[SYSTEM] CPU overloaded" kill
  interceptable via try/catch? A methodology question: if catchable,
  every future kill-based test could log its own exact death point and
  keep running instead of relying on inference from the last printed
  checkpoint. P1 (raise() inside try/catch, proves the language feature
  itself works here) -> P2 (a 200-iteration unpaced burst at top level,
  inside try - the actual answer) -> P3/P4 (nested + error-object
  fields, only reachable if P2 was caught).
  Progress: COMPLETE, 3/3 runs, all identical.
  Result: NOT CATCHABLE, confirmed unambiguously and identically in all
  3 runs - exactly the test's own predicted/expected branch. P1 prints
  cleanly (caught=1, try/catch genuinely works in this script/session).
  Output then stops immediately after "P2 flat burst starting NOW" in
  every run, followed by the kill banner - no "P2 FLAT CAUGHT=1", no
  P3, no P4, ever. The pre-burst cpu reading (59.9999999999983) is
  IDENTICAL down to the float-noise digits in all 3 runs. This CONFIRMS
  the kill is a genuine external/engine-level termination, not a
  language-level error - no in-script handler can ever intercept it or
  keep the script alive past it. METHODOLOGY-CONFIRMING: every crash
  location this whole investigation has ever recorded, inferred from
  the last-printed checkpoint, was the only method available and is
  validated by this result, not merely assumed. Side observation
  (informal): the user watched the live CPU meter hold near 0 for ~1s
  after the burst started before jumping 80->100->crash, while the
  script's OWN pre-burst get_cpu() read at roughly the same moment
  already showed 60 - reinforces the open question (also raised in
  T09) of whether the live UI meter and the get_cpu() API value are
  the same quantity or sample on different cadences. See CPU
  Research/results.txt (T06 runs 1-3) and field_notes.txt for the full
  writeup.

T10.bd  (T10, CPU-load research programme - NOT a menu.bd test)  [DONE]
  Goal: DEATH TRACE - what does get_cpu() read at (or one line before)
  the kill, and for how many real seconds does it sit at the ceiling
  first? First test of the KILL-TRIGGER block; logs one numbered
  get_cpu() sample per unpaced line until the engine kills it, instead
  of inferring the death point from whatever last happened to print.
  PART A instruments get_cpu(); PART B (control) substitutes a copy of
  the loop counter to isolate get_cpu()'s own polling cost.
  Progress: COMPLETE, 8/8 runs (3 cold PART A >5s apart, 3 hot PART A
  <=0.5s apart back to back, 2 PART B) - run in a CONTINUED session,
  not a fresh boot, which turned out to matter (see below).
  Result: THREE major findings.
  (1) RUN-TO-RUN VARIANCE REPRODUCED DIRECTLY: 5 of 6 PART-A runs died
  at exactly n=5 (t~5.4s), cpu=100 every sample; the 6th survived to
  n=9 (t~9.8s, nearly DOUBLE), with the only difference in its own
  trace being a single dip to cpu=90 (not 100) at n=5 - a below-ceiling
  "blip" that appears to have reset or delayed the kill's internal
  countdown. This is exactly T04's already-demonstrated rapid-polling
  instability (identical rapid-fire get_cpu() reads gave different
  values run to run there too) - but T10 now shows that instability,
  when it happens to land below the ceiling mid-burst, can measurably
  extend a script's survival time. This is the leading mechanistic
  candidate for the central menu.bd variance mystery: chance below-
  ceiling blips in an inherently rapid-poll-unstable meter, not any
  real randomness in the underlying engine.
  (2) THE "5 STRAIGHT SECONDS" KILL RULE: sustained-ceiling duration
  before death brackets to (4.3s, 5.5s] for normal runs and (3.3s,
  4.4s] for run 6's second (post-dip) streak - broadly consistent with
  ~5s but not pinned down more precisely than that by this instrument's
  ~1.1s sampling granularity. Neither confirms nor refutes an exact
  5.0s constant.
  (3) UNPLANNED, MAJOR SIDE FINDING ON log() COST: both PART A (~1.10s/
  iteration) and PART B (~1.00s/iteration, no get_cpu() at all) ran
  3-5x SLOWER than the test's own predicted 3.3-5.0 iterations/sec for
  a 2-3 statement loop body. The ~0.1s gap between A and B matches
  get_cpu()'s expected single extra tick exactly - but the remaining
  ~1.0s (10 ticks) for a loop body of just n=n+1 + one log() line (3
  str() calls, 2 concatenations) implies THAT LOG() LINE COSTS ROUGHLY
  7-9 TICKS BY ITSELF, not the 1 tick budgeted for this exact debug-
  line style everywhere in field_notes.txt. Not yet isolated/confirmed
  - T18 (log() pricing, not yet run) is the properly controlled
  instrument for this; T10 only surfaced it as a side effect.
  ALSO: entry_cpu reads ~0 (matching T01/T03/T09's "fresh script's
  first read is 0" pattern) but floor_cpu, read only 3 SLEEP-ONLY
  seconds later, reads 30-40 - backwards from simple decay, and it does
  NOT reset even after >5s idle between kills (climbs 30->30->40 then
  plateaus at 40 across the session regardless of cold/hot spacing).
  Strong evidence that repeated kills within one continued session
  leave a PERSISTENT elevated idle floor that doesn't fully clear the
  way the clean ~10 floor did in T01/T03/T04/T08/T09 - none of which
  had just experienced a violent kill. Methodological caution for any
  test run in a session where a kill already happened. See CPU
  Research/results.txt (T10 runs 1-8) and field_notes.txt for the full
  writeup.

T18.bd  (T18, CPU-load research programme - NOT a menu.bd test)
  Goal: price log() itself. Does it cost exactly one tick regardless of its
  argument, or does cost scale with STRING LENGTH, with the number of
  CONCATENATION OPERATIONS building the argument, or with the optional
  second Color argument? Applied stake: this project's ubiquitous debug
  pattern log("..." + str(get_unix_time() - t0) + "s") is budgeted at ONE
  tick everywhere in field_notes, but the v1.1.0 changelog (Updates/v1.1.0
  - Persistence.txt line 60, "the same function call, wrapped in identical
  arithmetic, could cost wildly different amounts of tick budget depending
  only on which side of the expression it sat on") describes an accountant
  that walks the expression tree - under which that line is 7-12 ticks and
  every instrumented trace in the corpus is inflated in the same direction.
  Seven arms in ONE execution, 4 rounds x 6 measured lines each, wall clock
  only, ZERO get_cpu() calls: NOLOG (b = 1) / SHORT (log 2-char) / LONG
  (log 241-char) / COLOR (log + Color.amber) / CONCAT (3 ops, 0 calls) /
  SHORTB (SHORT repeated as drift+scrollback control) / DBUG (the literal
  production debug line). LONG-vs-SHORT and COLOR-vs-SHORT are exact
  matched pairs - identical expression trees, one variable changed.
  THE SIX UNPACED log() LINES ARE THE EXPERIMENT - a kill is a result;
  arms ascend in predicted cost and the FULL summary for arms 1-6 prints
  BEFORE the riskiest arm (DBUG) runs, so a DBUG death still delivers six
  arms fully analysed. ~3 min/run, 2 runs.
  Progress: authored, not yet run. Depends on T02 (seconds-per-tick, and
  its verdict that the engine blocks on tick exhaustion) and T03 (noise
  floor); confirms/refutes T14's charging-unit verdict in the applied case.
  Result: pending.

test_decode_pacing.bd
  Goal: A/B - does utils/hostdata.bd's decode_tuple_list() (real,
  unmodified) eventually CPU-overload if hammered repeatedly with
  representative 3-item blobs? Does adding a second throttle() mid-item
  (PART B, local copy) measurably change that?
  Progress: PART A and PART B each run once, both manually interrupted
  (not crashed) after 31 and 37 rounds respectively.
  Result: NEITHER crashed - PART A held steady at cpu=100 for ~840s,
  PART B at cpu=80 for ~1442s (both far longer than the real ~122s
  crash). CONCLUSION (negative result): decode_tuple_list() alone, even
  unmodified, is NOT the direct cause of the real menu.bd crash - the
  extra throttle() measurably lowered the steady-state reading (100->80)
  and slowed throughput, a real effect, just not one this test could
  credit with preventing a crash (since the original didn't crash either).

test_token_dispatch.bd
  Goal: does read_entries()'s FULL per-token dispatch block (starts_with()
  elif chain + both real nu.debug_time() calls, debug forced on to match
  production - not just decode_tuple_list() alone) crash when hammered
  with the real crash trace's exact 5 tokens?
  Progress: PART A run twice (20 rounds/~608s, then 19 rounds/~587s),
  PART B run once (18 rounds/~549s). Restored to Active/tests/ (swapped
  back in over test_dispatch_after_imports.bd, which was abandoned) for
  this second round of testing.
  Result: neither PART ever crashed, and PART A vs PART B show
  ESSENTIALLY IDENTICAL per-round pacing (never more than ~1-2s apart
  across 18-20 rounds) - CONCLUSIVELY rules out the always-on debug
  logging as a meaningful cost factor. Both survived ~550-608s, ~5x
  longer than the real ~100-150s crash, the same negative result as
  every other isolated-hammering test this investigation has tried
  (test_decode_pacing.bd, test_dispatch_after_imports.bd). CONCLUSION:
  every synthetic-isolation angle tried so far has failed to reproduce
  the real crash - it likely depends on something in the FULL real
  menu.bd context, not on raw work volume, debug logging, or import
  debt individually.

test_dispatch_after_imports.bd
  Goal: same token-dispatch hammering as test_token_dispatch.bd, but run
  behind the SAME 11-module import chain menu.bd itself goes through
  first (sleep(1) after each) - tests whether the real run's ~19s of
  pre-get_entry() setup debt is the missing ingredient no prior test
  reproduced.
  Progress: PART A started, manually interrupted early (round 2, ~65s
  into dispatch) - too little data yet for a real comparison against
  test_token_dispatch.bd's 20-round/608s survival. Needs a comparably
  long run to mean anything. PART B not run.
  Result: inconclusive so far. Raw CPU during the IMPORT phase itself
  (player capture): 0-20-0-20-0-20-10-20-10-100-70-20-0-60-100-40-60-
  100-80-10-0-100 - confirms imports alone visibly spike cpu to 100
  multiple times (matches "importing costs ticks proportional to line
  count" - already known, now directly visible). BUT by the time
  dispatch actually started (logged "imports done" line), cpu had
  settled back down to 20 - same low baseline as test_token_dispatch.bd
  started from. So far this does NOT support "import debt carries over
  into dispatch" - the import-phase spikes appear to fully decay via the
  sleep(1) after the last import, same as everywhere else pacing has
  been shown to work. Dispatch's own round-to-round pace (round0->1:
  ~33s, round1->2: ~31s) also matches test_token_dispatch.bd's un-prefixed
  pace closely - no obvious slowdown from the import prefix either.
  Genuinely undetermined until it either crashes or survives a
  comparable round count.
  Progress update: abandoned - a comparable-length run costs 10+ minutes
  of real time for an inconclusive result; investigation pivoted to a
  faster real-world test instead (see below).

decode_tuple_list() mid-item throttle fix (applied directly to
utils/hostdata.bd, not a tests/ script - real-world test instead of more
synthetic hammering)
  Goal: does adding a second nu.throttle() mid-item (after field-by-field
  build, before append) delay or prevent the REAL menu.bd crash against
  the real host_intel.txt (known baseline: crashes at +122.5s, on line 4
  token 5)?
  Progress: applied to utils/hostdata.bd + Active mirror, one real `menu`
  run, then reverted immediately after the result came in.
  Result: made it WORSE - crashed at +53s instead (on line 2's per-line
  throttle, a bare-IP line with no tokens, previously always safe). Every
  step after the fix was shifted later by a near-constant ~3.8s (one
  extra throttle() call), and that constant per-item tax compounded across
  the run to bring the crash ~70s EARLIER, not delay it. CONFIRMS: more
  throttle() calls is not automatically safer once get_cpu() itself is
  understood to add load (see field_notes.txt CORRECTION) - reverted back
  to the original single-throttle-per-item shape. See field_notes.txt for
  the full before/after per-token trace.

read_entries() per-token throttle removal (opposite direction, applied
directly to utils/hostdata.bd)
  Goal: does REMOVING read_entries()'s per-token throttle() (relying on
  the per-line throttle + decode_tuple_list()'s own per-item throttle
  instead) delay or prevent the real crash?
  Progress: applied, one real `menu` run, reverted immediately.
  Result: made it MUCH worse - crashed at +13.6s, on line 1's very first
  token (fw=0:0:0, the cheapest possible field). Worse than even the
  extra-throttle attempt (+53s). Reverted, confirmed back to original.
  CONCLUSION: both tested directions (add a throttle, remove a throttle)
  made the crash happen sooner than the +122.5s baseline - the current
  single-throttle-per-token shape looks like a local optimum for this
  lever. "How many throttle() calls" is very likely not the actual lever
  controlling the crash - see field_notes.txt's NEXT IDEAS for where this
  points next (reducing total work per run, not re-tuning pacing around
  the same amount of work).

find_entry_by_host() - targeted get_entry() lookup (applied directly to
utils/hostdata.bd, real fix not a tests/ script)
  Goal: does skipping decode_*() work for every non-matching host in
  host_intel.txt (cheap per-line string match first, decode only the
  matching line) reduce or prevent the real crash?
  Progress: applied, live-tested twice (missing target 45.155.67.201,
  existing target 94.140.22.83), same session.
  Result: worked exactly as designed - missing-target run did ZERO
  decode work at all (get_entry() done at +50.2s, pure scan cost);
  existing-target run skipped lines 1-3 entirely and decoded only line
  4's real data. BUT the script still crashed both times - just later,
  and in a different place (utils/dashboard.bd's refresh_status(),
  around cracked_accounts_for()) - which led directly to the
  cracked_accounts_for() fix below. On a SECOND live retest (after the
  creds.bd fix), the existing-target run crashed much earlier than its
  own established baseline (+40.3s vs +109.5s/+122.5s, inside get_entry()
  itself) even though hostdata.bd was untouched between runs - player
  confirmed no session-carryover effect, so this is the ORIGINAL "timing
  varies wildly across otherwise-identical runs" mystery re-appearing on
  this code path, not a regression from this fix. See field_notes.txt for
  the full writeup - the root cause of that variance is still unknown.

cracked_accounts_for()/is_cracked() double-pass fix (applied directly to
utils/creds.bd, real fix not a tests/ script)
  Goal: both functions called read_credentials(nu) (a full throttled
  pass building an in-memory list) then throttled AGAIN over that list
  to filter for one host - same redundant-work shape as get_entry() had.
  Does reading the file once and filtering inline reduce the ~106-108s
  cracked_accounts_for() was costing every dashboard refresh?
  Progress: applied, live-tested once (missing-target run).
  Result: real improvement - cracked_accounts_for() cost dropped from
  ~106.1s to ~72.5s (~32% faster), and the overall crash moved later
  (~180.9s -> ~148.8s absolute). Did NOT close the gap entirely - the
  script still crashed, past check_trace() this time. A prior attempt at
  this exact function (commit 874834f) tried batching the throttle
  instead of removing the redundant pass; reverted (400c735) when
  batching was found unsafe elsewhere, which brought the full cost back
  until this fix.

Full-codebase audit for the get_entry()-style redundant-pass bug
  Goal: check crack.bd, grab.bd, cleanup.bd, firewall.bd, probe.bd,
  sync.bd, and net.bd's walk_tree()/check_trace() family for the same
  "decode/process everything when only one thing is needed" pattern
  already found and fixed twice (get_entry(), cracked_accounts_for()).
  Progress: complete read-through of every file, no live testing needed
  (this was a static audit, not a runtime test).
  Result: none of them have the bug - every throttled loop matches real,
  necessary work (scan N ports because N were requested, download every
  file because that's what "grab" means, etc.). sync.bd's use of the
  "full" read_entries()/write_entries()/read_credentials() functions is
  legitimate (do_sync() genuinely needs the complete list to merge/dedup).
  FOUND one more real instance as a direct result: save_entry() (see its
  own entry below) - the write-side twin of get_entry()'s bug, never
  caught by a live crash because the crash always happened earlier in
  the function.

save_entry() fix - write-side twin of get_entry()'s bug (applied
directly to utils/hostdata.bd)
  Goal: save_entry() called read_entries() (full decode of every host)
  then write_entries() (full RE-ENCODE of every host, 7 throttle() calls
  per entry) just to change one host's data - called on every
  refresh_status() and mark_cracked(). Does a targeted single-line
  rewrite (encode only the changed entry, copy every other line through
  as untouched raw text) reduce this cost?
  Progress: applied, live-tested twice (missing-target, existing-target).
  Result: INCONCLUSIVE - neither test run actually reached save_entry()
  (it's called at the very end of refresh_status(); both runs crashed
  well before that point). Fix remains theoretically sound (same
  reasoning that worked for get_entry()) but structurally unexercised by
  every real test so far.

Throttle before get_security()/probe_vuln() in refresh_status() (applied
directly to utils/dashboard.bd, then reverted)
  Goal: missing-target run crashed extremely reproducibly (98.60s/98.60s
  across two runs) right after "1st check_trace() done" - get_security()/
  probe_vuln() run with no throttle() ahead of them (only after, further
  down). Does a throttle() there delay/prevent the crash?
  Progress: applied, one live test, reverted immediately after.
  Result: ZERO measurable effect - crashed at +98.6000s, essentially
  identical to +98.6020s/+98.6030s before this change. Unlike the two
  prior throttle-placement experiments (both of which measurably shifted
  the crash point, one worse and one much worse), this one changed
  nothing. Reverted to keep the code simple. Reframed the missing-target
  timing as NOT mysterious - it's a deterministic sum of fixed call costs
  (reflects the creds.bd fix's ~33.6s saving exactly), unlike the
  existing-target path which shows real bimodal variance (+40.3s vs
  +109.5s) tied to something about actually connecting to a real target.
  STATUS: 3 throttle-placement experiments tried total (2 harmful, 1
  no-effect) - this direction looks exhausted; total-work-reduction fixes
  remain the validated productive direction instead.

cracked_accounts_for() caching (Codebase Audit Synthesis.txt P1, applied
directly to menu.bd + utils/dashboard.bd)
  Goal: cracked_accounts_for() was STILL the single most expensive
  remaining call in refresh_status() (~72.5s every refresh) even after
  its own double-pass fix, because it re-scans the whole credentials
  file on every refresh regardless of whether anything changed. Does
  caching the result in state.accounts (invalidated only by a real
  crack or a target switch) eliminate most of that cost?
  Progress: applied (state.accounts starts none, refresh_status() only
  calls cracked_accounts_for() when still none, mark_cracked() updates
  the cache incrementally, target-switch resets it to none).
  Live-tested twice (missing-target, existing-target).
  Result: INCONCLUSIVE by construction, not by failure - both scenarios
  crash before ever reaching a SECOND refresh_status() call within the
  same script execution (missing-target crashes during its one and only
  refresh; existing-target crashes before refresh_status() is even
  entered), and caching only helps the 2nd+ call. The fix is real and
  will help long play sessions with repeated refreshes/actions, but does
  NOT address the startup crash this investigation is chasing - that
  happens on/before the first refresh, where the full ~72-90s cracked_
  accounts_for() cost is unavoidable regardless of caching. See
  field_notes.txt for the full reasoning and the next-idea it points to
  (a secondary per-host credentials index, to reduce the FIRST-call
  cost itself - not yet attempted).

T25.bd  (T25, CPU-load research programme - NOT a menu.bd test)
  Goal: does something accumulate across a play session such that one
  fixed workload runs measurably slower the longer the game has been
  open - and does restarting the game reverse it? The manipulation is
  entirely OUTSIDE the script: one byte-identical probe (6 rounds of the
  same 20-line unrolled burst T20 uses, each bracketed by sleep(2)) run
  four times in each of three session states - A fresh boot, B after 15+
  min of ordinary play including two or three menu.bd overload kills,
  C immediately after a second relaunch. Headline scalar is work= (the
  summed burst seconds, undiluted by the sleeps, carried as a running
  total on every round line so a killed run still yields a partial).
  Secondary: LINE1_cpu, get_cpu() on the very first executable line,
  which on the FIRST script of a fresh boot also separates "the meter is
  clean at game load" from "the meter looked clean because the player
  idled first" - an assumption no test has ever isolated.
  THE REVERSAL IS THE CLAIM: a rise in B without a fall in C is NOT
  confirmation, it is the separate and bigger finding that something
  survives a relaunch. Discriminates S1 no effect / S2 reversible
  ageing / S3 durable ageing / S4 per-run-not-per-session (block B's own
  four runs climbing among themselves). Matters because the 5-run
  variance block (+40.7/+111.0/+40.3/+65.5/+64.3s) was collected late in
  one long, heavily crashed session and has been treated throughout as
  five independent samples.
  Progress: authored, not yet run. Needs T03's noise-floor range R (or
  falls back to its own block-A range, whichever is larger) and T20's
  carryover verdict to attribute a block difference to session age
  rather than to the runs immediately preceding it. Cannot be done in
  one sitting - two game restarts and 15+ min of play between blocks -
  so schedule it around the rest of the programme.
  Result: pending.

T16.bd  (T16, CPU-load research programme - NOT a menu.bd test)
  Goal: in an 8-branch if/elif chain, is the tick cost charged PER
  CONDITION EVALUATED, only for the BRANCH TAKEN, or for EVERY PHYSICAL
  LINE of the construct regardless? read_entries()'s per-token dispatch
  is exactly this shape (7-deep starts_with() elif chain, no else), and
  the real crash trace shows late-branch tokens costing more than early
  ones - an observation never separated from the decode work each branch
  does. Six arms share ONE measuring loop; only the bracketed window
  differs: CTRL (empty - the floor), LAST (k=8), NOMATCH (k=99, the
  pivot: eight failed tests and NO body line), FIRST (k=1), CTRL-B
  (drift control), STRLAST (the real starts_with() shape, 8 deep, whose
  delta over LAST prices one starts_with() condition). The three headline
  arms are the SAME function called with a different argument, so the 16
  source lines are provably identical and k cannot be constant-folded.
  Arm order runs UPHILL on purpose (LAST before FIRST): drift can only
  hide the predicted spread, never manufacture it. Wall clock only;
  get_cpu() appears exactly twice per arm, outside every measured
  bracket. Four models, four distinct signatures - see the header table.
  NOMATCH is the verdict pivot: NOMATCH == CTRL means condition testing
  is free; NOMATCH near LAST means testing is where the cost is.
  Progress: authored, not yet run. Converts to absolute ticks via T02's
  sec_per_unit; its rider cross-checks T14's per-call verdict.
  Result: pending.

T26.bd  (T26, CPU-load research programme - NOT a menu.bd test)
  Goal: does anything OTHER than this script's own executed lines feed the
  CPU meter? Specifically: is get_cpu()'s idle floor higher while a
  connection is open and its trace is counting down, and higher again while
  a trace is merely COOLING after disconnect, than it is with nothing
  connected? No official document lists a trace, an IDS sweep or an open
  socket as a CPU contributor, and none lists them as non-contributors; no
  prior script in tests/ has opened a connection at all.
  Three phases measured by ONE poll() function called with a one-character
  label, so the 3-line instrument is provably identical across phases and
  the whole comparison lives inside one run: A 10 polls disconnected /
  B 10 polls connected, trace live / C 12 polls disconnected, trace cooling.
  poll()'s body is the same shape as test_cpu_decay.bd's idle loop, so
  PHASE A is directly comparable to that test's replicated 10.0 floor.
  get_trace_time()/traces()/ids_status()/server_specs() are called ONLY on
  the bracketing MARK lines, never inside a poll loop - the B-IN/B-OUT
  countdown readings prove the trace was live without touching the
  instrument. PART N is the null control: same phases, same sleeps, no
  connection ever opened, isolating meter drift over elapsed runtime.
  PRIMARY STATISTIC IS NOT THE MEDIAN: the meter is quantised in 10-point
  steps (1 tick at base tier), so a sub-tick environmental cost cannot move
  the modal value - report per phase both the modal value AND the count of
  samples reading above PHASE A's mode. Six discriminated outcomes: inert /
  standing connection cost / trace-not-socket (the jackpot) / sub-tick /
  connect() transient only / drift artefact - see the header table.
  Why the cooling phase is the sleeper: a cooling trace is the only known
  game state that persists ACROSS script runs and decays on real wall-clock
  time. If it costs even 1 tick/sec, five back-to-back runs against a host
  start at five different points on that curve - the exact shape of the
  +40.7/+111.0/+40.3/+65.5/+64.3s spread. Run order is N first (clean), then
  two connected runs back to back with NO wait between them, so run 3's
  PHASE A is deliberately a cooling-trace baseline and run 2 vs run 3 PHASE A
  is a free cross-run carryover measurement. Rider: the SPECS line records a
  real remote cpu_tier/cpu_ticks - the first anywhere in this investigation.
  ~70s per run, 3 runs, no waiting between them.
  Progress: authored, not yet run. Reads against T04's meter-response unit
  and T08's decay curve (to tell a standing cost from connect()'s own 5-tick
  transient decaying away); sibling of T21 (does the meter sum across
  PROCESSES) asking the same question about non-process engine state.
  Result: pending.
