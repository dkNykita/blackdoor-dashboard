POSSIBLE BUG REPORTS - index
=============================
Compiled from field_notes.txt and live testing across the whole
refactor session. Each file contains: Current behavior, Expected
behavior, Process description, Steps to reproduce - ready to submit/
report, or to use as a basis for discussion with other players/devs.

01_sqli_raises_instead_of_returning.txt
  sqli() can throw a runtime error instead of the documented graceful
  {success, data, error} return.

02_object_literals_cannot_span_multiple_lines.txt
  An object literal can't be written across multiple lines (parse
  error), even though every other multi-line construct in the
  language works fine.

03_download_denies_var_log_despite_ownership.txt
  download() denies files under /var/log even for a confirmed owning,
  world-readable account - works normally outside /var/log.

04_connect_success_not_documented_as_unauthenticated.txt
  connect().success doesn't mean successful login - documented only
  in documentation_from_editor.txt, missing from api_documentation.txt.

05_module_top_level_restriction_undocumented.txt
  Unlike a top-level script, a module can't have a top-level `var`
  initialized by a function call - undocumented anywhere, only
  surfaces as a parse error.

06_prompt_ignores_whitespace_only_input.txt
  A space + Enter never reaches prompt() (the terminal discards it) -
  breaks the common "space for default" convention.

07_get_cpu_aggregate_inconsistent_across_identical_runs.txt
  get_cpu()'s aggregate behaves inconsistently across identical script
  runs - the largest and least closed item, still a partially open
  question even after extensive elimination testing.
