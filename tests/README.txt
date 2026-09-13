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
  Progress: NOT YET RUN.
  Result: pending.
