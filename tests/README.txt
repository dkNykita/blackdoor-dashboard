TESTS - index
=============
One entry per script in this folder: tested goal, current progress,
result. Kept in sync whenever a test is added, re-run, or its result
changes - see field_notes.txt for the full narrative/reasoning behind
each one; this file is just the compact status view.

CPU LOAD RESEARCH PROGRAMME (T01-T28): the authoritative plan, ordering,
methodology, recording protocol and per-experiment reference lives in
../Blackdoor documentation/CPU Research/README.txt - read that first.
The T01-T28 scripts in this folder implement it. This file tracks their
per-script Result: as they are run; the programme file is the campaign.
Run results get appended to ../Blackdoor documentation/CPU Research/
results.txt (one record per run). The four test_cpu_decay/decode_pacing/
token_dispatch/dispatch_after_imports scripts predate the programme (they
came out of the frozen menu.bd investigation) and are NOT part of T01-T28.

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

T09.bd  (T09, CPU-load research programme - NOT a menu.bd test)
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
  Progress: authored, not yet run. Depends on T04/T08 reference values.
  Result: pending.

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
