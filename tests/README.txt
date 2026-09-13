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
