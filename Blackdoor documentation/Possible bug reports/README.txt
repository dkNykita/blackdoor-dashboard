POSSIBLE BUG REPORTS - index
=============================
Sestaveno z field_notes.txt a živého testování napříč celou refactor
session. Každý soubor obsahuje: Aktuální stav/chování, Očekávaný stav,
Popis procesu, Způsob replikace - připraveno k odeslání/nahlášení,
případně jako podklad pro diskuzi s dalšími hráči/vývojáři.

01_sqli_raises_instead_of_returning.txt
  sqli() může vyhodit runtime chybu místo dokumentovaného graceful
  {success, data, error} návratu.

02_object_literals_cannot_span_multiple_lines.txt
  Object literál nejde zapsat na víc řádků (parse chyba), i když
  každá jiná víceřádková konstrukce v jazyce funguje bez problémů.

03_download_denies_var_log_despite_ownership.txt
  download() odmítne soubory pod /var/log i pro potvrzeně vlastnící,
  world-readable účet - funguje normálně mimo /var/log.

04_connect_success_not_documented_as_unauthenticated.txt
  connect().success neznamená úspěšné přihlášení - zdokumentováno jen
  v documentation_from_editor.txt, chybí v api_documentation.txt.

05_module_top_level_restriction_undocumented.txt
  Modul (na rozdíl od top-level scriptu) nesmí mít na top-level `var`
  inicializovanou voláním funkce - nezdokumentováno nikde, jen parse
  chyba při pokusu.

06_prompt_ignores_whitespace_only_input.txt
  Mezera + Enter se do prompt() nikdy nedostane (terminál to zahodí) -
  rozbíjí běžnou "space for default" konvenci.

07_get_cpu_aggregate_inconsistent_across_identical_runs.txt
  get_cpu()'s agregát se chová nekonzistentně mezi identickými běhy
  skriptu - nejrozsáhlejší a nejméně uzavřená položka, stále částečně
  otevřená otázka i po rozsáhlém vylučovacím testování.
