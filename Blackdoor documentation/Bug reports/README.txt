POSSIBLE BUG REPORTS - index
=============================
Compiled from field_notes.txt and live testing across the whole
refactor session. Each file contains: Current behavior, Expected
behavior, Process description, Steps to reproduce - ready to submit/
report, or to use as a basis for discussion with other players/devs.

OPEN: none currently - all 6 reports below were closed by v1.1.1
(see Official documentation/Updates/v1.1.1 - Bug Fixes.txt).

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

WITHDRAWN FOR NOW:
  get_cpu() aggregate inconsistency across identical runs - pulled
  pending retesting under the corrected tick-cost model (network calls
  like connect()/download() turned out to cost real ticks, not 0 -
  see field_notes.txt). Prior testing for this report was done under
  the wrong cost assumption, so its conclusions aren't reliable yet.
