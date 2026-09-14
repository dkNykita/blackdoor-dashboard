POSSIBLE BUG REPORTS - index
=============================
Compiled from field_notes.txt and live testing across the whole
refactor session. Each file contains: Current behavior, Expected
behavior, Process description, Steps to reproduce - ready to submit/
report, or to use as a basis for discussion with other players/devs.

Three states now, one folder each:
  Open/         - reported, not yet actively worked
  Investigation/ - actively being investigated (may take a while,
                  may never fully resolve - not the same as Closed)
  Closed/       - resolved, with a "STATUS: RESOLVED" note per file

OPEN - see Open/ for full text:

Open/01_param_default_type_and_quoting_leak.txt [script]
  get_param() returns a @param-declared default typed/quoted exactly as
  written in the comment, ignoring the inline fallback argument: a
  quoted default (`default="10"`) leaks literal quote characters into
  the value, and an unquoted numeric default (`default=10`) returns an
  actual NUMBER even when the call site expects a string - either one
  crashes downstream code that assumes a clean string. Found internally
  2026-09-14 via two successive T01.bd crashes; submitted to Discord
  same day (see Report link:). Fix already applied to this project's
  own tests/T01-T28 (see field_notes.txt); this report is about the
  engine behavior itself, not our code. (Renamed from
  01_param_default_quotes_embedded_literally.txt when the second, type-
  coercion half of the bug was found and folded in.)

INVESTIGATION - see Investigation/ for full text:

Investigation/01_cpu_load_instability_investigation.txt [script]
  (Renamed from 01_get_cpu_run_to_run_inconsistency.txt, original
  report kept at the bottom of the file for history.) Consolidates the
  original get_cpu() run-to-run inconsistency symptom (5 runs, same
  code+data: +40.7s/+111.0s/+40.3s/+65.5s/+64.3s) with 6 concrete,
  independently-reproduced findings from the CPU Research programme's
  controlled testing: (1) a non-deterministic kill boundary under
  repeated near-lethal bursts - the leading root-cause candidate; (2)
  get_cpu() quantizing to multiples of 10 and disagreeing under rapid
  polling; (3) its decay window not matching a real throttle-loop's
  cost; (4) a persistent elevated floor after any kill; (5) log() with
  string concatenation costing ~7-9x a plain line; (6) get_unix_time()
  returning a float against its documented int. Dev (2026-09-14) has
  acknowledged a likely cause and is preparing a beta-branch fix - this
  consolidated report was posted to Discord the same day (see Report
  link:) to compare against it once the fix lands. menu.bd/utils/*.bd
  development remains paused; see CLAUDE.md.
  Active research programme for this report: Investigation/CPU Research/
  (README.txt = methodology + 28-experiment plan; results.txt = per-run
  log) - the T01-T28 scripts it defines live in ../../tests/.

CLOSED (v1.1.1) - see Closed/ for full text + resolution note per file:

Closed/01_sqli_raises_instead_of_returning.txt [script]
  sqli() can throw a runtime error instead of the documented graceful
  {success, data, error} return.
  -> Fixed: sqli() now returns a normal result instead of crashing.

Closed/02_object_literals_cannot_span_multiple_lines.txt [script]
  An object literal can't be written across multiple lines (parse
  error), even though every other multi-line construct in the
  language works fine.
  -> Fixed: object literals, lists, and function calls can now span
     multiple lines.

Closed/03_download_denies_var_log_despite_ownership.txt [gameplay, script]
  download() denies files under /var/log even for a confirmed owning,
  world-readable account - works normally outside /var/log.
  -> Fixed (reframed): ls now shows real permissions/ownership,
     matching download()/upload() - confirms the denial itself was
     legitimate, ls was the one showing wrong data.

Closed/04_connect_success_not_documented_as_unauthenticated.txt [script]
  connect().success doesn't mean successful login - documented in the
  in-editor `help api` reference, but not flagged in the web API
  reference's own connect() example.
  -> Fixed: web API docs for connect() now carry the same warning the
     in-editor one has.

Closed/05_module_top_level_restriction_undocumented.txt [script]
  Unlike a top-level script, a module can't have a top-level `var`
  initialized by a function call - undocumented, only surfaces as a
  parse error (import itself IS documented, just not this asymmetry).
  -> Fixed: in-editor scripting reference now explains what a
     module's top level can hold.

Closed/06_prompt_ignores_whitespace_only_input.txt [ui, script]
  A space + Enter never reaches prompt() (the terminal discards it) -
  breaks the common "space for default" convention.
  -> Fixed: prompt() no longer discards whitespace-only input.

Note: "crash" (available tag) means a GAME crash (the app itself),
not a script halting on an error - none of the reports above qualify,
so none carry that tag.
