CPU LOAD RESEARCH PROGRAMME
===========================
A first-principles investigation of Blackdoor's CPU-load model: how
get_cpu() actually computes its number, what one tick costs, exactly
what triggers the "[SYSTEM] CPU overloaded" kill, and - the real prize
- why the SAME script on the SAME data crashes at wildly different
points across identical back-to-back runs (observed: +40.7 / +111.0 /
+40.3 / +65.5 / +64.3 seconds).

This programme exists because menu.bd development was FROZEN (2026-09-14)
after every symptom-level fix and every blind pacing tweak failed to
resolve that crash. We stopped tuning the script and turned to
understanding the machine underneath it. The goal is a MODEL of the CPU
system solid enough that menu.bd work can resume on top of it - not
another guess.

Scope note (standing rule): this is a SIMULATED in-game subsystem.
Never explain a result by the player's real computer, OS, or window
manager. Every hypothesis here must be expressible and testable purely
inside the game.

HOW THIS DOCUMENT WAS BUILT
---------------------------
28 experiments (T01-T28) were designed by a multi-agent process: four
readers digested field_notes.txt, the official docs, every prior test,
and the community notes; six independent "lenses" each proposed
experiments (quantitative model / tick-cost taxonomy / kill trigger /
variance mystery / observer effect / scope & multi-process); overlaps
were merged into 28 canonical experiments; each was authored into a
complete runnable script. An adversarial review pass (two critics per
test: isolation+interpretation, and feasibility+redundancy) completed
for T01-T10 before the run hit a rate limit. So:
  - T01-T10 carry ADVERSARIAL-REVIEW FLAGS below - fold those in before
    running.
  - T11-T28 are authored but NOT yet critic-reviewed - re-read each
    script critically before spending real in-game time on it. Their
    predictions are still stated in advance in the script headers, so a
    crash-before-measurement is still recognizable as such.

THE MODEL WE ARE TRYING TO BUILD
--------------------------------
Every observation so far is consistent with, but has not pinned down,
this working model. The programme is designed to confirm each clause or
break it:
  1. Time is ticks. In-game elapsed "seconds" may be a tick counter
     (base tier 10 ticks/sec) presented as seconds, not a sampled wall
     clock. Evidence: three runs reproduced 98.6000 / 98.6020 / 98.6030s
     - a 3ms spread a real clock cannot produce. (T01 settles this; if
     true, every "seconds" figure in field_notes.txt is really a LINE
     COUNT and must be re-read in those units.)
  2. One executed line = one tick. Unconfirmed whether the unit is a
     physical line, a parsed statement, or an evaluated expression. (T14)
  3. get_cpu() = ticks charged in a recent window, as a percentage of
     the tier's per-second budget. The window's shape (hard edge vs
     exponential decay) and length are unknown. Dyadic readings (93.75 =
     15/16; 99.9984... = 1-2^-16) hint at a power-of-two EMA. Idle sits
     rock-steady at 10.0 (= exactly one tick in a one-second window at
     base tier). (T04, T07, T08)
  4. The kill fires after load stays at the ceiling for a sustained
     duration (~5s per the security guide). Whether it is duration-at-
     ceiling, accumulated tick-debt, or a queue rule - and whether a
     sub-second sleep resets it - is unknown. (T06, T10-T13)
  5. The budget's SCOPE is unknown: per-script or shared across all the
     player's running processes; whether traces/IDS/connections/daemons
     feed it. (T20-T22, T26, T27)
  If clauses 1-4 hold and clause 5 turns up a shared, time-varying,
  cross-run-persistent contributor, THAT is the shape of the variance
  mystery. If no such contributor exists and T03 shows a large noise
  floor even in a bare deterministic script, the variance is irreducible
  engine nondeterminism and menu.bd must simply be built to tolerate a
  worst-case budget.

STANDING METHODOLOGY (read before every session)
------------------------------------------------
  * A KILLED SCRIPT IS DATA, NOT FAILURE. Many tests are supposed to
    die; the kill (and where it happened) is the measurement. Every
    script is built so its LAST printed line is still informative if it
    dies mid-run.
  * ONE RUN PROVES NOTHING here. The whole subject is run-to-run
    variance. Believe a number only after the stated repeat count. For
    variance tests, minimum 6 runs, ideally 8; for deterministic
    calibration tests, 2-3 is enough to catch a fluke.
  * RUN FROM A KNOWN STATE. Before each run, record whether it is: a
    fresh game boot, a run immediately after a previous script was
    killed, or a long-idle session. This is itself a variable (T20, T25).
    Check `ps` / `jobs` and confirm nothing else is running unless the
    test specifically wants a second process.
  * PASTE THE FULL RAW OUTPUT. Never summarize a run by hand before it
    is recorded - the exact numbers (including float tails like
    10.0000001798557) are the signal.
  * CROSS-CHECK THE INSTRUMENT. get_cpu() and log() may perturb what
    they measure. Every meter-based finding must be cross-checked
    against a meter-free channel (a kill/no-kill outcome, or a wall-clock
    interval) before it is trusted. T05/T19 measure the perturbation
    directly; T03/T11 are meter-free by construction.
  * WHEN A RESULT CONTRADICTS AN EARLIER FINDING, do not overwrite -
    record both and design a tie-breaker. field_notes.txt has been wrong
    and self-corrected before; assume it can again.
  * NEVER re-suspect an already-ruled-out cause (see field_notes.txt's
    STARTUP CPU OVERLOAD INVESTIGATION) without new evidence.

RECORDING PROTOCOL (so the work resumes cold, weeks later)
----------------------------------------------------------
Keep a running results log (results.txt in this folder, appended never
overwritten) plus the one-line outcome mirrored into tests/README.txt's
"Result:" field for that script. One record PER RUN, with these fields:
  RUN-ID      : test id + run number, e.g. T04 run 3/3
  WHEN        : real date/time, and session marker (boot #, or "same
                session as previous run")
  START-STATE : fresh-boot | after-kill | long-idle ; ps/jobs clean? y/n
  TIER        : CPU tier as T02 measured it (or "base 10t/s assumed")
  RAW         : the complete pasted terminal output, verbatim
  SCALAR      : the one number/outcome this run contributes (e.g.
                interval=27.1s ; or "killed at burst iter 47" ; or
                "SURVIVED")
  KILLED?     : no | yes-at:<last line printed>
  VS PREDICT  : confirms | refutes | ambiguous, against the script
                header's stated CONFIRMS-IF / REFUTES-IF
  NOTES       : anomalies, meter values, anything unexpected
After a test's repeats are all in, write a one-paragraph VERDICT into
tests/README.txt (Result:) AND, if it changes the model, into
field_notes.txt. That is the durable record; this programme file is the
plan, tests/README.txt is the per-script status, field_notes.txt is the
theory.

THE ADAPTIVE SESSION PLAN
-------------------------
Sessions are sitting-sized (~15-25 min incl. repeats). Run them in
order: the early ones calibrate the instrument that the later ones use.
Each session ends at a DECISION POINT that can redirect the rest.

SESSION 1 - CALIBRATE THE INSTRUMENT   [T01, T02, T03, T04]   ~20-25 min
  Establish the units everything else is measured in: is the clock real
  (T01), how many seconds is one executed statement / what is the real
  tier (T02), what is the run-to-run NOISE FLOOR of a bare deterministic
  script (T03), and how many meter-points does one line add / is the
  meter a linear counter or an EMA (T04).
  DECISION POINT: If T01 shows time is tick-quantised, re-read every
  "seconds" figure in field_notes.txt as line counts before continuing.
  If T03's bare-script noise floor is already large (>15% scatter with
  NOTHING else running), the variance is engine-level - jump the
  variance session (5) forward, because the taxonomy sessions can't
  explain a floor that exists without them.

SESSION 2 - METER MEMORY                [T07, T08, T09]        ~15-20 min
  How get_cpu() aggregates over time: the sub-second window edge vs a
  decay (T07), the decay curve from saturation and whether the famous
  ~3.7s throttle constant falls out of it (T08), and whether work above
  100% vanishes or accrues as hidden tick-debt (T09).
  DECISION POINT: window-vs-decay and ceiling-vs-debt together decide
  whether "100% for 5s" is a duration rule or an accumulated-debt rule -
  which reframes the entire kill session (3).

SESSION 3 - THE KILL TRIGGER            [T06, T10, T11, T12, T13]  ~20 min
  Is the kill catchable by try/catch (T06 - if yes, every later kill
  test can self-report, making them cheaper); what does the meter read
  at the instant of death (T10); the exact lethal unpaced-line boundary
  per construct (T11); does a sleep reset the kill timer and how much
  (T12); is the threshold exactly 100% or lower (T13).
  DECISION POINT: run T06 FIRST. If the kill is catchable, re-tool
  T10-T13 to catch-and-report before running them.

SESSION 4 - TICK-COST TAXONOMY          [T05, T14, T15, T16, T17, T18, T19]  ~25 min
  What actually costs a tick: call surcharge (T05), the charge unit -
  line vs statement vs expression (T14), loop-iteration overhead (T15),
  if/elif chain cost (T16), never-called declaration cost in a script vs
  a module (T17), log() pricing by length/concat/colour (T18), and
  whether polling the meter holds it up (T19).
  DECISION POINT: this produces a per-construct cost table. With it, any
  script's tick budget is predictable from its source - the tool needed
  to eventually explain (and rebuild) menu.bd. If T19 shows polling
  materially perturbs the meter, retro-flag every meter-based number in
  the corpus.

SESSION 5 - THE VARIANCE MYSTERY        [T20, T21, T22, T23, T24, T25, T26, T27]  ~25-30 min
  The actual goal. Does the meter reset between runs (T20); do two
  processes share one budget (T21); is the kill session-wide or
  per-process (T22); does a full terminal scrollback cost more (T23); is
  there a periodic engine pulse (T24); does session age slow a fixed
  workload (T25); does an open connection / live trace / COOLING trace
  feed the meter (T26 - the cooling trace is the prime suspect: it is
  the one known game state that persists across runs and decays on wall-
  clock time, exactly the shape of the crash spread); does remote_run()
  bill local or remote budget (T27).
  DECISION POINT: whichever factor moves the scalar across repeats is
  the variance cause. If NONE does, the variance is irreducible - record
  it and design menu.bd for the worst case.

SESSION 6 - TIER CARD (deferred)        [T28]                  ~10 min now
  Lay the baseline tier card now; re-run UNCHANGED after any future CPU
  upgrade to learn whether the meter is tier-normalized and whether the
  kill is wall-clock or fixed-tick. Low priority until an upgrade is
  actually bought.

EXECUTION ORDER (flattened, priority-front-loaded):
  T01 T02 T03 T04  ->  T07 T08 T09  ->  T06 T10 T11 T12 T13
  ->  T05 T14 T15 T16 T17 T18 T19  ->  T20 T21 T22 T23 T24 T25 T26 T27
  ->  T28

BIGGEST RISKS TO THIS PROGRAMME
-------------------------------
  1. The variance has no observable in-game cause (true engine
     nondeterminism). Contingency: T03 quantifies the floor; T24 hunts
     periodicity; T20/T25/T26 chase persistence. If all come back null,
     document the variance as irreducible and switch strategy from
     "explain it" to "tolerate it" (build menu.bd against a worst-case
     budget with wide margins).
  2. The observer effect swamps the signal. Contingency: T05/T19
     quantify it; the kill-based channel (T03/T11/T22) sidesteps the
     meter entirely and cross-checks every meter finding.
  3. The un-reviewed tests (T11-T28) hide a constraint violation that
     crashes the script before it measures anything. Contingency:
     re-read each script before its session; its header states the
     prediction, so a crash-before-measure is distinguishable from a
     real kill result.
  4. Runs are too few to beat the variance and a difference is called
     that is really noise. Contingency: the stated repeat counts and
     T03's measured error bar R - never call a difference smaller than R.

===============================================================================
PER-EXPERIMENT REFERENCE (T01-T28)
Priorities: P1 = highest information gain per minute, run first;
P4 = only if earlier results demand it. Full runnable scripts live in
the repo's tests/ folder (../../../../tests/ from here) under the
filename(s) shown. Read the script header before running - it repeats
the prediction and the run instructions.
===============================================================================
------------------------------------------------------------------------------
T01  [P1]  Clock and external-instrument validation
  file(s): T01.bd
  NEEDS ON DISK:
    Just T01.bd in ~/scripts/. Self-contained; also drives the terminal
    commands `time`, `ps`, `jobs` from outside the script (no disk deps).
  depends on (other tests first): None.
  Q:  Is get_unix_time() a true wall clock (agreeing with the terminal's
    `time` command and a physical stopwatch outside the game), at what
    granularity does it advance, does `time` measure a script from outside
    it including one that gets killed, and do ps/jobs expose any per-process
    CPU accounting?
  CONFIRMS IF:
    REAL WALL CLOCK: on both STEP 1/2 repeats the physical stopwatch and the
    printed `interval=` agree within 0.5s on a ~27s interval (e.g.
    interval=27.1s, stopwatch 27.0s and 27.3s), AND raw_clock_t0 prints a
    plausible current Unix epoch (~1789xxxxxx in Sept 2026).
  REFUTES IF:
    SIMULATED OR SCALED TIME: the stopwatch differs from the printed
    `interval=` by more than 1.5s in the SAME direction on both repeats
    (e.g. interval=27.1s but stopwatch 19s twice).
  runtime: Per run: clock mode at blocks=10 is ~36s total (~2s preamble, ~6s granularity probe, ~27s timed interval at ~2.7s per round — 1s of sleep plus ~17 exe  | repeats: STEP 1 (stopwatch) TWICE — mandatory.
  BLIND SPOTS:
    WHAT IT CANNOT TELL US `time` is an in-game command, so it may well read
    the same simulated clock as get_unix_time(). If both are simulated, they
    agree perfectly and prove only internal consistency.
------------------------------------------------------------------------------
T02  [P1]  Tick ruler: seconds per executed statement, and the player's real tier
  file(s): T02.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): T01 (external clock validation) should run first, since get_unix_time() is this test's only instrument.
  Q:  Does the engine actually block a script when its per-second tick budget
    is spent — i.e. is elapsed wall-clock time a linear readout of executed-
    statement count — and what is the real seconds-per-statement (hence
    ticks/sec tier)?
  CONFIRMS IF:
    All of the following hold, on both runs: (1) mean block= rises
    MONOTONICALLY across arms 0 -> 10 -> 20 -> 30; (2) the three steps are
    EQUAL — step(0->10), step(10->20) and step(20->30) agree with each other
    within ±25% or within the within-arm spread, whichever is larger;
  REFUTES IF:
    PRIMARY REFUTATION — block= is ~0.00–0.05 s at EVERY arm including 30,
    AND the five PHASE span= totals are equal within ~1 s (i.e. the spans
    are pure sleep time: ~19.6 / ~19.6 / ~19.6 / ~19.6 / ~9.8 s).
  runtime: ~115 s per run at base tier (PHASE-0 ~20 s, PHASE-10 ~24 s, PHASE-20 ~28 s, PHASE-30 ~32 s, PHASE-0B ~10 s, plus ~1 s of header).  | repeats: 2 full runs, back to back in the same session.
  BLIND SPOTS:
    WHAT IT CANNOT TELL US. (1) Per-LINE vs per-OPERATION charging.
------------------------------------------------------------------------------
T03  [P1]  Baseline noise floor: repeated runs of one fixed deterministic workload
  file(s): T03.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): T02 (the tick-rate ruler), for interpretation and safety-sizing only — T02 supplies the ticks/sec figure that 
  Q:  Does a minimal, imports-free, call-free, deterministic script doing
    exactly the same fixed arithmetic work take the same wall-clock time
    every run, or does run-to-run variance exist even in the simplest
    possible system? Concretely: what is the numeric noise floor R (percent
    spread of a fixed workload's duration) that every later A/B and variance
    test must be…
  CONFIRMS IF:
    AMBIENT VARIANCE CONFIRMED — any one of these, with run order showing no
    monotonic warm-up/degradation trend that explains it: (a) RUN-LEVEL:
    (max DONE total − min DONE total) / median DONE total >= 0.15 across the
    8 runs. (b) BETWEEN-RUN (per-run medians): take each run's median burst=
    value;
  REFUTES IF:
    AMBIENT VARIANCE REFUTED — all three hold simultaneously: (a) all 8 DONE
    total values within 5% of their median (e.g. every run 29.1-31.3s), (b)
    the 8 per-run median burst= values within 5% of the grand median, (c)
    all 40 individual burst= samples within 10% of the grand median.
  runtime: ~30 s per run at base tier (10 lines of prelude, then 5 rounds of sleep(1) + ~2.5 s burst + sleep(1) + log), proportionally less on any higher tier —   | repeats: 8 runs back to back in one uninterrupted session, giving 40 burst samples and 8 DONE totals.
  BLIND SPOTS:
    WHAT IT CANNOT TELL US 1. SCOPE.
------------------------------------------------------------------------------
T04  [P1]  Meter response function: points per executed line, and linear counter vs EMA
  file(s): T04.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): T02 (wall-clock tick-rate ruler) — soft dependency, not blocking.
  Q:  How many meter points does one executed line add, is get_cpu()'s own
    line charged to the reading it returns, and is the reading a linear
    function of lines just executed or a geometric saturation curve?
  CONFIRMS IF:
    Model L is confirmed if ALL of the following hold: 1. PART A returns
    five values separated by a constant step, within float noise — the
    canonical expectation being 10/20/30/40/50, but ANY constant step s
    confirms the model and fixes the unit at 1 line = s points.
  REFUTES IF:
    Model L is refuted, and each refutation mode names its own replacement:
    - GEOMETRIC SATURATION: PART B's steps shrink with N — N=1 already reads
    above 70 and N=3 above 90 — and the residual (100 - cpu) forms a
    geometric sequence. That refutes the tick counter and confirms an EMA;
  runtime: ~95-105 seconds per run.  | repeats: 2 runs, back to back, in the same fresh session.
  BLIND SPOTS:
    SUBSTANTIVE DISAGREEMENT BETWEEN THE SOURCE DRAFTS, RESOLVED HERE. Draft
    L1T2 stated that a slope S gives "tier = 1000/S ticks/sec".
------------------------------------------------------------------------------
T05  [P1]  Call surcharge and the observer effect: what get_cpu(), get_unix_time(), str() and a user call really cost
  file(s): T05.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): T01 (tick-rate calibration: seconds per executed statement) — needed ONLY to convert the measured seconds-per-
  Q:  Does calling get_cpu() add measurable execution cost beyond the one tick
    its source line already costs — and what is the marginal per-call cost
    of get_unix_time(), str() and a user-defined function? Secondarily: is
    the charge levied per CALL or per LINE?
  CONFIRMS IF:
    All comparisons are against the SAME run's CTRL arm, never across runs.
    d_X = mean(burst, arm X) - mean(burst, CTRL), in seconds;
  REFUTES IF:
    REFUTES "get_cpu() adds load" as anything beyond the cost of its own
    line: d_gcpu < 0.15s across both runs WHILE the UFNC arm shows a clear
    positive delta. The positive control is required — a null result with no
    arm showing any delta means the method is blind, not that calls are
    free.
  runtime: ~2.5 minutes per run at base tier if all surcharges are zero: 7 arms x 5 rounds x ~3.2s per round (sleep(1) + ~0.8s burst + sleep(1) + log) = ~112s, p  | repeats: 2 runs, back to back in the same session, nothing else running.
  BLIND SPOTS:
    1. TIME AND TICKS ARE CONFLATED.
------------------------------------------------------------------------------
T06  [P1]  Is the overload kill catchable by try/catch
  file(s): T06.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): none — no imports, no file I/O, no state, no dependency on any other experiment's result.
  Q:  Does try/catch intercept the CPU-overload kill, or is it an
    unconditional engine-level termination outside the language? And if it
    is catchable: does execution continue afterwards, does it work one call
    deep, and what does the engine call the error?
  CONFIRMS IF:
    CONFIRMS the hypothesis (kill is NOT catchable, external to the
    language) if, in all 3 runs: "T06 P1 CONTROL caught=1" prints (proving
    try/catch demonstrably works in this exact script and session), then
    "T06 P2 flat burst starting NOW" prints, then the overload banner
    appears and NOTHING else does — no P2 CAUGHT line, no P3 resumed line,
    no end line. The P1…
  REFUTES IF:
    Two distinct refutations, which must not be confused. (1) CATCHABLE —
    "T06 P2 FLAT CAUGHT=1" prints.
  runtime: Expected branch (not catchable): ~18-20s per run — 2s settle, ~6s for the control phase, 3s pre-burst, then death 5-7s into the burst.  | repeats: 3 runs, subject to two stop rules.
  BLIND SPOTS:
    (1) The central irreducible confound: a null result cannot fully
    separate "the kill is not catchable" from "the handler ran but could not
    emit". sleep(3)-first and the P1 control bound this, but the control
    proves the construct works at LOW load, not at ceiling load, and no in-
    language observation of a handler that cannot speak is possible.
------------------------------------------------------------------------------
T07  [P1]  Meter memory I: sub-second window edge and partial credit for aged lines
  file(s): T07.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): T04 (the instrument unit) — NOMINALLY, and deliberately made non-blocking.
  Q:  Does get_cpu() forget old ticks at a hard age cutoff (a sliding window
    with a definite edge) or discount them continuously (a decay) — and
    where is that edge? Measured two ways in one run: by varying the delay
    AFTER a fixed burst (how ticks age out), and by varying a gap INSIDE the
    same burst (whether credit for old ticks is partial). A window length
    recover…
  CONFIRMS IF:
    HARD WINDOW CONFIRMED if, in ARM A, the successive differences down the
    ladder (d = 0.05, 0.20, 0.40, 0.60, 0.80, 1.00, 1.40, 2.20) are roughly
    CONSTANT at about -U per 0.1s of extra d while the reading is falling,
    and then become exactly ZERO — with the last readings equal to each
    other to within |delta| < 0.01 (float noise is ~1e-7; one unit is ~10,
    so 0.0…
  REFUTES IF:
    WINDOW REFUTED, DECAY INSTEAD, if ARM A's successive differences SHRINK
    geometrically instead of staying constant, and the reading never stops
    falling: A8 (d=2.20) strictly below A7 (d=1.40) strictly below A6
    (d=1.00), each by more than 0.01. Corroborated from the other side if B4
    (g=3.00) is strictly below B3 (g=1.50).
  runtime: ~2.0 to 2.2 minutes per run.  | repeats: 2 runs.
  BLIND SPOTS:
    WHAT THIS CANNOT TELL US 1. A FAST DECAY IS INDISTINGUISHABLE FROM AN
    EDGE.
------------------------------------------------------------------------------
T08  [P1]  Meter memory II: decay from saturation over seconds, and the ~3.7s throttle constant
  file(s): T08.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): T04 (the unit: how many meter points one tick is worth, hence where the floor actually is — the (reading − flo
  Q:  Starting from a saturated meter, what does get_cpu() read after exactly
    k whole seconds of sleeping (k = 0,1,2,3,5,8) — a step down to the
    floor, or a graded decay? And does the ~3.7s throttle constant fall out
    of that curve, or not?
  CONFIRMS IF:
    DECAY CONFIRMED (graded) if all four hold: 1. Sorting the eight printed
    post= values BY k gives a monotonically falling series — and sorting
    them by PHASE NUMBER does not (the phase order is deliberately
    scrambled: 0,3,1,8,2,5,0,2).
  REFUTES IF:
    Any one of these refutes the decay hypothesis, and each is separately
    decisive: STEP (Model A confirmed): k=0 reads ~90-100 and k=1,2,3,5,8
    are all equal at the floor, with their spread no larger than the
    P1-vs-P7 replicate spread. This refutes decay-over-seconds entirely,
    confirms a sub-second window, and promotes the ~3.7s throttle constant
    to a genuine un…
  runtime: ~100-115 seconds per run.  | repeats: 2 runs, back to back, in the same session — with a 3rd immediately if the two disagree on SHAPE (step vs grade
  BLIND SPOTS:
    WHAT IT CANNOT TELL US 1. Sub-second structure is invisible.
------------------------------------------------------------------------------
T09  [P2]  The ceiling: does work above 100% vanish, or is there hidden tick debt
  file(s): T09.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): T04 (decay from saturation — one clean sample per delay) and T08 (the k=2 fixed-delay reference value).
  Q:  Does the meter clamp at 100 and discard everything above it, or does an
    accumulator keep counting past the ceiling so a bigger burst leaves more
    residue behind?
  CONFIRMS IF:
    CLAMP CONFIRMED — all four cb readings equal within float noise (phases
    started identically, so the comparison is valid), all four c0 readings
    equal within float noise, and all four c2 readings equal within float
    noise. Observed get_cpu() noise is ~1e-7, so any difference above ~0.001
    is real and anything below it is not.
  REFUTES IF:
    DEBT CONFIRMED (clamp refuted) — cb stays flat across all four phases
    but c2 rises monotonically with B (e.g. 10 / 25 / 55 / 70).
  runtime: ~45 seconds per run.  | repeats: 3 runs back to back in one fresh session, nothing changed between them.
  BLIND SPOTS:
    SUBSTANTIVE DISAGREEMENT WITH THE SOURCE DRAFT, and why it was resolved
    this way. L1T5 specified one capture per phase at a fixed 2-second
    delay.
------------------------------------------------------------------------------
T10  [P1]  Death trace: what the meter reads at the instant of the kill
  file(s): T10.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): No hard dependency: the script is fully standalone (no imports, no file I/O, no state) and can be run at any p
  Q:  What is get_cpu() reading at and immediately before the moment the
    overload kill fires, and for how many consecutive real seconds was it at
    the ceiling first? Secondary, from the same trace at zero extra cost:
    what is the player's actual tick rate / CPU tier, and does a killed
    script leave residual load for the next one?
  CONFIRMS IF:
    ALL of the following hold on the 3 cold PART A runs: 1. No SURVIVED line
    appears;
  REFUTES IF:
    ANY one of these refutes some part of the hypothesis, and each is
    separately informative: (a) The last printed cpu is below 99 -> the kill
    does not require a ceiling reading; get_cpu()'s printed value is not the
    trigger quantity.
  runtime: Per run: ~3s settle + a burst that either ends in death at a predicted ~6s or runs the full 20s wall-clock bound + a 5s tail = 23-28 seconds of script  | repeats: 8 runs: 3 cold PART A, 3 back-to-back (hot) PART A, 2 PART B.
  BLIND SPOTS:
    SUBSTANTIVE CHANGES I MADE TO THE SOURCE DRAFT (L3T1), recorded as
    required: 1. Count-bounded loop replaced with a time-bounded one.
------------------------------------------------------------------------------
T11  [P1]  Lethal burst boundary, per construct, with and without get_cpu() probing
  file(s): T11.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): No hard dependency — the script self-calibrates (the CALIB line derives this machine's seconds-per-iteration a
  Q:  What is the smallest unpaced burst that kills — for pure arithmetic, for
    the cheapest log line, and for a log line with one extra concatenation —
    is that boundary sharp or stochastic, and does inserting get_cpu() calls
    into the burst move it?
  CONFIRMS IF:
    SHARP BOUNDARY (P1): some n prints an OK line on every trial (8 in-run
    holds at Stage 2 plus the ladder rung) and n+step prints no OK line on
    all 3 fresh runs at Stage 3. The kill is then a deterministic count of
    consecutive unpaced ticks.
  REFUTES IF:
    NO SHARP BOUNDARY — THE HEADLINE REFUTATION: the same n both survives
    and dies. Concretely, either (a) Stage 2 prints OK for rungs 0,1,2 and
    then no OK line at rung 3 or later at a fixed n, or (b) any of the three
    Stage 3 fresh runs at the smallest lethal n prints an OK line while the
    others do not.
  runtime: Per run: ~15s for a single shot (settle + burst + settle + calibration), ~20s for a log arm, ~70s for an 8-rung hold, and 45-130s for a full 10-rung l  | repeats: Bisection needs no repeats — a kill is a binary signal and a surviving run is not ambiguous.
  BLIND SPOTS:
    WHAT IT STILL CANNOT TELL US 1. WHAT the killer counts.
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T12  [P1]  Yield semantics: does a sleep reset the kill timer, fully or partially, and how short can it be
  file(s): T12.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): T11 (burst bisection: the exact lethal straight-line count n*) for the burst size — shipped at 30 lines from t
  Q:  Does a single sleep between sub-lethal bursts reset the kill timer
    completely, does debt survive the sleep and accumulate across bursts
    until a stack of individually-safe bursts kills, or does the sleep drain
    only part of it — and is a short yield (0.05-0.5s) as protective as
    sleep(1)? One harness, two sweeps: the rep index sweeps repetition count
    at a fixed…
  CONFIRMS IF:
    Read the floor= column first, the survival bit second. FULL RESET
    confirmed if every rung prints all 20 reps and "T12 PHASE-B SURVIVED",
    AND floor= is flat across reps (no trend beyond get_cpu()'s ~1e-7 float
    noise), AND PHASE-C dies by r=2-3 in those same runs (the control
    proving the burst really was lethal when concatenated).
  REFUTES IF:
    (1) SCATTERED DEATH REPS across identical runs of the same rung (e.g.
    r=3, r=14, r=6 at ms=1000) refutes full reset, accumulator and partial
    drain simultaneously, and localises the investigation's central run-to-
    run variance inside the kill trigger itself — the direct explanation for
    the +40.7/+111.0/+40.3/+65.5/+64.3 spread, and by far the most valuable
    pos…
  runtime: ~115s per run at ms=1000 (20 reps x ~4.5s, plus ~4s of yieldcheck, ~11s of settles, ~10s of control) and ~95s at ms=50;  | repeats: 2 runs per rung, and bisection instead of walking the ladder.
  BLIND SPOTS:
    (1) "Survives" is the weakest possible outcome: a drain rate only
    slightly faster than the burst rate is indistinguishable from full reset
    by the survival bit alone. This is why floor= is the headline channel
    rather than survival — a non-zero plateau proves incomplete reset even
    in a surviving run — but if the meter turns out to be clamped (T09's
    CLAMP resul…
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T13  [P2]  Sustained plateau: is the kill threshold exactly 100% or some lower level
  file(s): T13.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): T02 (the tick ruler) — HARD dependency, two ways.
  Q:  Will a load held steadily below the ceiling (95 / 90 / 82 / 70 / 60 / 50
    percent) also kill if it is sustained long enough, or is the kill
    threshold literally 100%? Equivalently: is the killer watching the LEVEL
    of load, or only whether the tick window is completely full?
  CONFIRMS IF:
    THRESHOLD IS EXACTLY THE CEILING if (a) the lvl=100 positive control
    dies within roughly 5 seconds (inside blk 1 or 2), AND (b) every paced
    rung including lvl=95 prints its SURVIVED line after the full hold, AND
    (c) each surviving run's SLEEPCAL line shows per == S and its blk t=
    spacing matches the header's pred_blk=, proving the constructed duty is
    the dut…
  REFUTES IF:
    METHOD INVALID (report and stop): the SLEEPCAL line shows per != S (the
    engine floors sub-second sleeps to a frame or to 0.1s, so the rung ran
    at a lower duty than its name), or the measured blk t= spacing is far
    BELOW the header's pred_blk= (the engine is not blocking the script to
    its tick rate, so wall-clock time is not a readout of tick consumption).
    The…
  runtime: About 70-95 seconds per run at base tier: ~8s preamble, the sleep calibration (8 x S, which is 11s at lvl=50 down to under 1s at lvl=95), a 60-second   | repeats: Positive control (lvl=100): 1 run, first, always, before any rung.
  BLIND SPOTS:
    1. GRAIN IS CONFOUNDED WITH LEVEL — the biggest one.
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T14  [P2]  Charging granularity: physical line vs statement vs evaluated operation, and the price of dead lines
  file(s): T14.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): T02 (T02.bd) — HARD dependency, twice over.
  Q:  Is tick cost charged per physical source line, per statement, or per
    evaluated operation — and do comment lines, blank lines and `pass`
    consume budget when they sit inside an executed block?
  CONFIRMS IF:
    All deltas are per-round block= against the SAME run's BASE; U =
    sec_per_unit from T02 (~0.10 s/unit at base tier).
  REFUTES IF:
    "Each line costs one tick" (the official doc line) is refuted outright
    if R >= 0.4 — 24 additions on 2 lines then cost roughly what 24
    additions on 24 lines cost, which no per-line model permits. It is
    refuted in the other direction, more strongly, if R > 1.5: operators
    would cost MORE than the lines holding them.
  runtime: ~2.0 to 2.5 minutes per run at base tier.  | repeats: 2 runs, back to back in the same session.
  BLIND SPOTS:
    1. THE IMPORT CLAIM IS CONSTRAINED, NOT SETTLED.
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T15  [P1]  Loop iteration cost: what one pass of a for/while costs beyond its body
  file(s): T15.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): Nothing hard-blocks running this: all three headline quantities are differences and ratios that are self-inter
  Q:  How many ticks does a loop header cost per iteration, over and above the
    statements in its body, and does `for i in range(n)` differ from `for x
    in <list>` differ from an explicit `while`?
  CONFIRMS IF:
    PER-PASS CHARGING CONFIRMED if the two independent instruments agree on
    a small integer number of ticks. Meter block: (M2 - M1) is a clean
    multiple of the meter's step size, ~30 points (3 ticks per while-pass);
  REFUTES IF:
    PER-PASS CHARGING REFUTED (header charged once per loop, not per
    iteration) if spanB - spanA < 2s and (M2 - M1) is ~10 points, with the
    while arm's whole excess accounted for by its explicit increment alone.
    PER-LINE CHARGING REFUTED ENTIRELY if (M2 - M1) is not a clean multiple
    of the meter's step size — e.g.
  runtime: ~2.4 minutes per run under the 1-tick-per-line model (meter block ~27s including six 3s settles;  | repeats: 2 runs.
  BLIND SPOTS:
    WHAT IT CANNOT TELL US. (1) It measures the loop PASS as an indivisible
    unit — it cannot decompose it into condition evaluation vs.
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T16  [P2]  if/elif chain: charged per condition tested, per branch taken, or per whole construct
  file(s): T16.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): T02 (tests/T02.bd) — supplies sec_per_unit, needed to convert spans into absolute ticks_per_chain,
  Q:  In an 8-branch if/elif chain, is tick cost charged per condition
    evaluated, only for the branch actually taken, or for every physical
    line of the construct regardless? Secondary: what does one
    `starts_with()` condition cost on top of a bare comparison, i.e. what
    does read_entries()'s real 7-deep token dispatch chain actually cost per
    token?
  CONFIRMS IF:
    PER CONDITION EVALUATED (the hypothesis) is confirmed when, in BOTH
    runs: (1) spread = sum_LAST − sum_FIRST >= 2.5 s — at base tier the
    prediction is 7 extra conditions x 5 rounds x 0.1 s = 3.5 s; (2)
    sum_NOMATCH sits between the two, roughly one tick (~0.5 s over 5
    rounds) BELOW sum_LAST, never above it;
  REFUTES IF:
    Per-condition charging is refuted whenever spread = sum_LAST − sum_FIRST
    < 1.5 s in both runs. Which rival then holds is read off the same three
    numbers: NO SHORT-CIRCUIT if all three of d_first/d_last/d_nomatch are
    >= 3.0 s but < 6.0 s and within 1.5 s of each other (every condition
    runs even after a match — branch order is worthless and the chain costs
    its…
  runtime: ~145 s per run at base tier (10 ticks/sec): CTRL ~19 s, LAST ~23 s, NOMATCH ~22 s, FIRST ~19 s, CTRL-B ~11 s, STRLAST ~26 s, plus ~13 s of arm-boundar  | repeats: 2 runs, back to back in the same session.
  BLIND SPOTS:
    WHAT IT CANNOT TELL US 1. One depth only.
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T17  [P2]  Declaration cost: never-called funcs in a script vs in an imported module
  file(s): T17.bd + T17_mod_a.bd + T17_mod_b.bd + T17_mod_c.bd
  NEEDS ON DISK:
    ALL FOUR files flat in ~/scripts/: T17.bd (driver) + T17_mod_a.bd +
    T17_mod_b.bd + T17_mod_c.bd. The driver imports the three modules by
    absolute path. Module line counts (8/32/32) and 6-line headers are load-
    bearing - do NOT edit them.
  depends on (other tests first): STRICTLY: nothing.
  Q:  Does a never-called `func` definition consume tick budget, and does it
    cost the same at a script's top level as inside an imported module?
    Specifically: is a declaration billed per physical source line it
    occupies, per declaration statement, at parse/startup, or not at all —
    and is that rule different in a module than in a script?
  CONFIRMS IF:
    All thresholds are relative to R = the INL-CTRL span measured in the
    SAME run (24 executed `a = a + 1` statements), so the verdict is CPU-
    tier-independent. The script prints R, R/2 and R/6 on the SUM-PRED line
    for direct side-by-side comparison with SUM-RAW.
  REFUTES IF:
    THE COMMUNITY CLAIM IS REFUTED if (B−C) ≥ 0.3·R in the median of three
    runs — module-level declarations are billed, and Krok 12's move of four
    functions into utils/dashboard.bd was justified by folklore that does
    not hold. This is the headline refutation and it is the reason the test
    exists.
  runtime: ~65-75 seconds per run at base tier.  | repeats: 3 runs, back to back, in the same session.
  BLIND SPOTS:
    SUBSTANTIVE DISAGREEMENTS WITH THE SOURCE DRAFT (L2T6), recorded as
    required: 1. The source draft read (decl_big − decl_small) as "the cost
    of 12 module-level declarations".
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T18  [P2]  log() pricing: string length, concatenation depth, and the colour argument
  file(s): T18.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): HARD PREREQUISITES (this test is uninterpretable without them): - T02 (T02) — supplies seconds-per
  Q:  Does log() cost exactly one tick regardless of its argument, or does
    cost scale with string length, with the number of concatenation
    operations, or with a second Color argument?
  CONFIRMS IF:
    Thresholds are per round (4 rounds per arm, 6 measured lines per round);
    at base tier 0.1s per line = exactly 1 extra tick.
  REFUTES IF:
    STRICT PER-LINE CHARGING (the full null, and the outcome that closes
    this question permanently): all seven arm means — NOLOG, SHORT, LONG,
    COLOR, CONCAT, SHORTB and DBUG — agree within T03's noise floor (about
    +/-0.10s per round, so |d| < 0.15s for every pair), in both runs.
    Critically, d_short = mSHORT - mNOLOG is ALSO ~0, because a bare `b = 1`
    is itself o…
  runtime: ~3 minutes per run under the per-line (null) model: 7 arms x 4 rounds x ~4.5s per round (sleep(1) + burst + sleep(2) + round line) is ~125s, plus ~18s  | repeats: TWO full runs, back to back, in the same session.
  BLIND SPOTS:
    SUBSTANTIVE DISAGREEMENTS WITH THE SOURCE DRAFT (L5T3), and why this
    version departs: 1. The draft ran five SEPARATE parameterised executions
    (`T18 none|short|long|concat|color`) and compared arms across
    runs.
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T19  [P2]  Throttle poll cost: does watching the meter hold it up
  file(s): T19.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): T11 (meter-free kill boundary n*) - needed to size loadn safely;
  Q:  How many times does throttle()'s `while get_cpu() > threshold: sleep(1)`
    loop actually iterate; does shortening the poll interval lengthen the
    wait (the instrument sustaining what it measures);
  CONFIRMS IF:
    POLLING PERTURBS (any one of these): mean wait= at iv=0.25/thr=80
    exceeds mean wait= at iv=1/thr=80 by more than 2.0s; or the iv=0.25 arm
    reaches polls=48/48 in 2 of 3 cycles;
  REFUTES IF:
    POLLING IS NEUTRAL: mean wait= agrees within +/-0.5s across iv=1, 0.5
    and 0.25 while polls= scales roughly 1:2:4, and PHASE P's unpolled clear
    time matches the iv=1 wait= within 1.0s — the wait is pure elapsed-time
    decay, the instrument does not sustain the reading, and
    field_notes.txt's "polling more often does NOT help and can make things
    WORSE" is wrong a…
  runtime: ~2.5 minutes typical, ~3.0 minutes worst case, at defaults (cycles=3, loadn=5, base tier).  | repeats: One run at defaults yields 5 PHASE P points and 3 within-run replicates of wait=/polls= per PHASE T condition 
  BLIND SPOTS:
    SUBSTANTIVE DISAGREEMENTS WITH THE SOURCE DRAFT, recorded as required.
    (a) The draft ran six separate executions, one per arm;
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T20  [P1]  Cross-run carryover: the meter on line 1 after four different preludes
  file(s): T20.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): Nominally T03 (the fresh-script baseline reference block and the run-to-run range R its post-kill medians woul
  Q:  Does the CPU meter reset between script runs? Specifically: does load
    left by a script that was just killed — or one that just finished
    normally — still read on the very first executable line of the next
    script, and does a script started seconds after a kill run degraded even
    when the meter reads clean?
  CONFIRMS IF:
    CARRYOVER CONFIRMED — any one of these three signatures is sufficient,
    and they are ranked by how decisive they are. (1) METER RESIDUE (M2):
    LINE1_cpu >= 20 in at least 2 of the 2 sub-second post-kill runs (B1,
    B2), while every prelude-D run reads <= 10 (the known idle floor).
  REFUTES IF:
    NO CARRYOVER: LINE1_cpu reads 0 (or within a couple of points of the
    prelude-D value) in BOTH sub-second post-kill runs and in prelude C, the
    post-kill total= medians sit inside the spread of the two prelude-D
    runs, and no run shows an R0-vs-R3 burst gradient outside what D shows.
    That converts a twice-repeated player assertion into a measurement,
    removes "t…
  runtime: Probe ~23s per run (4 rounds x ~4.5s, plus ~1s preamble and a ~2s exit burst);  | repeats: Nine runs in the full block: 4 post-kill (B1, B2 at <1s;
  BLIND SPOTS:
    WHAT IT CANNOT TELL US 1. A clean LINE1_cpu does NOT prove the KILL
    TRIGGER was reset.
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T21  [P1]  Two processes: does the meter sum, and is the tick budget shared or per-process
  file(s): T21.bd + T21_ghost.bd
  NEEDS ON DISK:
    T21.bd + T21_ghost.bd both in ~/scripts/. Launch T21_ghost with & FIRST,
    then run T21 in the foreground.
  depends on (other tests first): T04 (meter response function — what one executed line is worth in meter points) sharpens this test by saying w
  Q:  When two of the player's scripts run at once, does get_cpu() return the
    sum of both processes' load, and do they share one ticks/sec budget or
    each get a full one?
  CONFIRMS IF:
    METER AGGREGATES: arm B's PHASE M readings are >= 1.8x arm A's —
    expected ~20.0 against arm A's ~10.0 — in at least 7 of the 8 polls in
    BOTH B runs, while arm A itself reads 10.0 +/- 0.001 (re-validating the
    historical anchor before B is interpreted). PHASE M2 must agree with
    PHASE M within the same run, proving the reading is a steady property of
    "two proce…
  REFUTES IF:
    METER IS PER-SCRIPT: arm B's PHASE M readings are within +/-0.5 of arm
    A's across all 8 polls in both B runs (i.e. still 10.0,
    indistinguishable), and cpu0 still reads ~0, with the ghost confirmed
    alive in `ps` before and after.
  runtime: ~60s per watcher run solo (PHASE M ~16s, PHASE R ~20s, PHASE M2 ~8s, setup/settle ~12s, closing ~2s).  | repeats: 2 runs of arm A and 2 of arm B, strictly alternated A, B, A, B (the second A doubling as the recovery arm).
  BLIND SPOTS:
    1. CANNOT SEPARATE "METER COUNTS PROCESSES" FROM "METER COUNTS WORK".
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T22  [P1]  Kill scope: does an overload take down a second, perfectly-paced process, and does the victim see it coming
  file(s): T22.bd + T22_bomb.bd
  NEEDS ON DISK:
    T22.bd + T22_bomb.bd both in ~/scripts/, and /home/<you>/ must be
    writable (both save() a status file there, since a backgrounded script`s
    log is not visible). Launch order per the script header.
  depends on (other tests first): NOMINALLY T21 (two-process meter summation / budget sharing), T10 (solo death trace) and T06 (is the kill catc
  Q:  When one process overloads, is the kill session-wide (it terminates a
    second, well-paced script) or per-process — and in the session-wide
    case, what does the victim's own last get_cpu() reading show?
  CONFIRMS IF:
    SESSION-WIDE KILL (confirms the documented claim) — all five of these
    agree, in at least 2 of 3 ARM 1 runs: 1. the victim's terminal output
    stops and its "T22 VICTIM DONE" line never prints;
  REFUTES IF:
    PER-PROCESS KILL (refutes the documented claim) — the victim prints all
    40 polls and its "T22 VICTIM DONE" line, while /home/Nykita/t22_bomb.txt
    is frozen at state=BURNING and `ps` no longer lists the bomb. The
    offender died alone.
  runtime: ARM 0: ~65 seconds of script time (40 polls at ~1.5s each), plus ~10s of ps/setup.  | repeats: ARM 0: ONCE.
  BLIND SPOTS:
    1. ROLE ASYMMETRY IS NOT CONTROLLED IN THE MAIN ARM.
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T23  [P3]  Terminal scrollback: does printing into a full terminal cost more than into a cleared one
  file(s): T23.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): T03 (baseline noise floor R, from 8 back-to-back runs of one fixed deterministic workload) — hard dependency: 
  Q:  Does the volume of text already printed this session change how fast a
    script executes? Holding the printed payload exactly constant (the same
    six log() calls of the same two-character string every time), does the
    cost of emitting it depend on how much is already on screen — and if
    something does ramp, is it the buffer depth or simply time-in-run?
  CONFIRMS IF:
    SCROLLBACK COSTS. With mA/mB/mC/mD the four phase means for the fixed
    six-line log block and R = T03's per-round noise floor (~±0.10s): mC −
    mA > 2R, with strict ordering mA &lt;
  REFUTES IF:
    SCROLLBACK IS FREE. All four phase means agree within R of each other,
    the four PH-DONE span= values agree within 4R, and ARITH is likewise
    flat — no dose-response, no reversal to reverse.
  runtime: ~2 min 45 s per run (16 measured rounds at ~5.6s, two fill blocks at ~20s, plus ~35s of header, phase boundaries, snapshot and summary).  | repeats: Two runs minimum, back to back in one session: RUN1 after a terminal `clear` (launches into an empty terminal)
  BLIND SPOTS:
    SUBSTANTIVE DISAGREEMENT BETWEEN THE SOURCE DRAFTS, and how it was
    resolved. L4T3 wanted the terminal filled by hand with non-script
    commands (`help`/`cat` x40) before each of ten separate runs, so the
    fill costs the probe zero ticks;
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T24  [P2]  Pulse hunt: an unaliased, controller-free time series looking for periodic engine-side events
  file(s): T24.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): T03 (baseline noise floor — 8 back-to-back runs of one fixed deterministic workload) defines what a "slow samp
  Q:  Is there a periodic engine-side event (autosave, world simulation, a
    trace or IDS tick) that intermittently steals tick throughput from an
    already-running script — and if so, what is its period?
  CONFIRMS IF:
    S1 PERIODIC ENGINE EVENT CONFIRMED — all four of these together: (a) at
    least 2 rounds per run have burst= >= 1.5x that run's median burst=
    (median taken over R1..Rn, excluding the R0 warm-up); (b) those slow
    rounds appear at DIFFERENT round indices in different runs — this is the
    decisive clause, see refutes_if for why;
  REFUTES IF:
    S5 NO PERIODIC EVENT — every burst= in every run falls within 25% of
    that run's median with no round exceeding the 1.5x threshold, OR slow
    rounds occur but no single period P in the 8-120s band fits them across
    the pooled absolute timestamps (irregular, non-repeating spacing). That
    removes autosave/world-simulation/IDS pulses from the candidate list and
    push…
  runtime: ~4.8s per round (sleep(2) + ~2.0s burst + ~0.8s round overhead at base tier) x 22 rounds, plus a ~1.6s prelude and a 1s settle = ~1 min 50s per run.  | repeats: 3 runs minimum, started at deliberately UNEQUAL intervals (~20s then ~7s).
  BLIND SPOTS:
    (1) COARSE CLOCK is the single biggest threat. The official reference
    declares get_unix_time() -> int;
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T25  [P2]  Session age: fresh boot vs long-open session vs immediate restart
  file(s): T25.bd
  NEEDS ON DISK:
    Just this script alone in ~/scripts/. No imports, no file I/O, no
    connection - fully self-contained (get_param has defaults).
  depends on (other tests first): T03 (baseline noise floor — 8 back-to-back runs of one fixed deterministic workload) supplies R, the threshold
  Q:  Does something accumulate across a play session such that one fixed,
    byte-identical workload runs measurably slower the longer the game has
    been open — and does quitting and relaunching the game reverse it?
    Secondarily: on the very first script of a fresh boot, does get_cpu() on
    executable line 1 read 0, separating "the load meter is clean at game
    load" from…
  CONFIRMS IF:
    S2 CONFIRMED — REVERSIBLE SESSION ACCUMULATION: median work= across
    block B (aged session, 4 runs) exceeds median work= across block A
    (fresh boot, 4 runs) by MORE than R, AND median work= across block C
    (immediately after relaunch) falls back to within R/3 of block A's
    median. Both halves are required;
  REFUTES IF:
    S1 — NO SESSION ACCUMULATION: all three block medians of work= fall
    within R/3 of each other, no block's four runs trend monotonically among
    themselves, and every LINE1_cpu reads ~0 including the aged block's.
    That converts the player's "no carryover" assertion into a measured fact
    at session scale, means the 5-run variance block's samples were
    genuinely ind…
  runtime: ~30s per run: an 11-line prologue plus sleep(2), then six rounds of ~2.3s of work plus a log plus sleep(2).  | repeats: 4 runs per block, 12 total.
  BLIND SPOTS:
    BLOCK ORDER IS FIXED AND UNCOUNTERBALANCED. A always precedes B precedes
    C within one evening.
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T26  [P2]  Connected state: does an open connection, a live trace or a cooling trace feed the local meter
  file(s): T26.bd
  NEEDS ON DISK:
    T26.bd in ~/scripts/. The disconnected/null part needs nothing; the
    CONNECTED phases need a REACHABLE target host passed as the `host`
    param.
  depends on (other tests first): T04 (T04.bd) for the meter-response unit — what one tick is worth in meter points, which t
  Q:  With the script doing the identical trivial work every second, does
    get_cpu()'s local idle floor read higher (a) while a connection is open
    and its trace is counting down, and (b) while a trace is merely COOLING
    after disconnect, than it does with nothing connected at all?
  CONFIRMS IF:
    Any of three positive signatures, all read within a single run so the
    instrument cancels: STANDING CONNECTION COST — PHASE B's modal value is
    at least one full step (10.0 points) above PHASE A's modal value,
    sustained across 7 or more of B's 10 samples, and PHASE C falls back to
    PHASE A's level. An open socket is then a standing per-second cost and
    every con…
  REFUTES IF:
    ENVIRONMENT IS INERT — all three phases share one modal value (expected
    10.0, matching test_cpu_decay.bd's solo floor) and the elevated counts
    are zero or equal across phases, in both connected runs. That closes "an
    open connection is intrinsically more expensive per line" and "a live or
    cooling trace eats the player's CPU budget" as explanations for the exi…
  runtime: ~70 seconds per run (about 12 of them connected), and three runs back to back with no waiting between them: ~3.5 minutes of script time total, well un  | repeats: Three runs, in this order and with no waiting between them: one PART N null control first (while trace state i
  BLIND SPOTS:
    RESOLUTION FLOOR. The meter is quantised in what appears to be 10-point
    steps (one tick at base tier).
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T27  [P3]  remote_run() billing: does a remote payload burn local or remote budget
  file(s): T27.bd
  NEEDS ON DISK:
    T27.bd in ~/scripts/, PLUS: the Pivot Source tool (remote_run needs it),
    a reachable+cracked remote host (`host` param), and a runnable .bd
    already uploaded on that host at the `rpath` param.
  depends on (other tests first): T26 (does an open connection, a live trace or IDS activity feed the local meter) and T21 (what a second LOCAL 
  Q:  When a script launched with remote_run() saturates the CPU it runs on,
    does the player's local get_cpu() rise and do local scripts get killed,
    or is the cost charged entirely to the remote box?
  CONFIRMS IF:
    REMOTE BILLING (docs correct), and only if the validity gate passes
    first: "T27 LAUNCH rr=true" AND "T27 MID ram_used=" strictly greater
    than the pre-launch ram_used in the "T27 SPECS" line AND "T27 LIVE
    relaunch=false" (the payload provably started and was still running mid-
    window). With the gate passed: all 12 PHASE B readings within 15 points
    of Abar (the…
  REFUTES IF:
    LOCAL BILLING: any of B0/B1/B2 (the first three PHASE B readings) at or
    above Abar + 40, OR the script is killed during PHASE B — last line
    printed is "T27 LAUNCH" or a "T27 B n=" line, followed by the overload
    banner. Either contradicts three official statements at once and
    instantly promotes any remote payload, daemon or background job into a
    first-class e…
  runtime: ~80 seconds per run on base-tier hardware (~63s of sleep() plus ~130 executed lines at 10 ticks/sec, plus ~14 ticks of API calls).  | repeats: 2 runs.
  BLIND SPOTS:
    WHAT IT STILL CANNOT TELL US. There is no remote get_cpu() — the remote
    box's own meter is unobservable, so this measures only whether the LOCAL
    meter moves, never how hard the remote box is actually working.
  (!) not yet adversarially reviewed - re-read the script before running.
------------------------------------------------------------------------------
T28  [P4]  Tier scaling card: is the meter tier-normalised and is the kill wall-clock or fixed-tick
  file(s): T28.bd
  NEEDS ON DISK:
    T28.bd in ~/scripts/. Optionally reads ~/.config/blackdoor.conf for the
    tick rate; falls back to base tier (10 t/s) if that file is absent.
  depends on (other tests first): Baseline reference values it cross-checks rather than re-derives: T02 (tick rate / seconds-per-unit — CALIB bu
  Q:  Is get_cpu() a tier-normalised percentage or an absolute tick count, and
    is the overload kill 5 wall-clock seconds or a fixed number of ticks —
    i.e. what exactly does buying a faster CPU change?
  CONFIRMS IF:
    POST-UPGRADE (the informative reading, and the reason this card exists):
    - TIER-NORMALISED METER (U1): floorA AND slope both divide by the tier
    multiplier — floorA 10.0 -> 5.0 and slope 30.0 -> 15.0 on Mk.II; /4 on
    Mk.III;
  REFUTES IF:
    POST-UPGRADE: - ABSOLUTE-TICK METER (U2): floorA and slope are both
    unchanged after an upgrade. Then every percentage in field_notes is a
    statement about one tier only, the REFRAMING block's tier-independent
    ceiling is the correct account, and the normalisation guidance in
    field_notes CPU LOAD and utils/net.bd lines 70-78 must be struck.
  runtime: Base tier, expected death at rung 2 or 3: ~75-85 seconds.  | repeats: 2 runs now to lay down the baseline card, back to back in the same session, with ps/jobs checked before each.
  BLIND SPOTS:
    WHAT IT CANNOT TELL US AT ALL WITHOUT A PURCHASE. The headline question
    is comparative by construction.
  (!) not yet adversarially reviewed - re-read the script before running.
