TESTS - index
=============
One entry per script in this folder: tested goal, current progress,
result. Kept in sync whenever a test is added, re-run, or its result
changes - see field_notes.txt for the full narrative/reasoning behind
each one; this file is just the compact status view.

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
  Progress: PART A run once, manually interrupted after 20 rounds (~608s).
  PART B (debug off) NOT YET RUN.
  Result (PART A): did NOT crash - survived ~5x longer (608s vs ~122s)
  and ~10x more token-dispatches (100 vs ~10) than the real crash needed.
  CONCLUSION so far: neither raw token-dispatch volume nor (as far as
  PART A alone shows) the always-on debug logging fully explains the
  real crash - something else is still missing. Superseded in
  Active/tests/ by test_dispatch_after_imports.bd; kept here in git.

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
