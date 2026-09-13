CODEBASE AUDIT - index
======================
Systematic audit of ALL 84 functions across all 12 .bd files in this
repo (menu.bd + 11 utils/*.bd modules), done in two independent passes
plus a synthesis - not just the functions that happened to be on the
CPU-overload investigation's crash path (see ../field_notes.txt for
that narrative).

METHODOLOGY (why two passes)
- Independent/ - each function read and evaluated fresh: evidence
  (real caller count via grep), nested calls, purpose/reason for
  existing as its own function, relevant doc notes, and alternatives
  with an honest evaluation - written as if the CPU investigation never
  happened, to avoid confirmation bias toward things already suspected.
- Integrated/ - same structure, same functions, but explicitly cross-
  referenced against field_notes.txt/tests/README.txt: confirms where
  the fresh read agrees with what live testing already found, and flags
  where the fresh read reached the SAME conclusion independently
  (stronger signal) vs. where it's the only source for something.
- Synthesis.txt - compares the two, lists what's confirmed by both,
  what the Independent pass surfaced that the investigation never had
  a chance to notice (since it was busy chasing the crash), and a
  PRIORITIZED list of concrete next-step candidates. Read this first if
  you just want the actionable conclusion - the numbered files behind
  it are the evidence trail.

FILES (one per module, same numbering in both Independent/ and
Integrated/)
  01_menu.txt        - menu.bd (0 functions - top-level script, audited
                        by structural section instead)
  02_net.txt          - utils/net.bd (21 functions)
  03_hostdata.txt     - utils/hostdata.bd (31 functions)
  04_creds.txt        - utils/creds.bd (3 functions)
  05_crack.txt        - utils/crack.bd (7 functions)
  06_grab.txt         - utils/grab.bd (2 functions)
  07_cleanup.txt      - utils/cleanup.bd (4 functions)
  08_firewall.txt     - utils/firewall.bd (1 function)
  09_probe.txt        - utils/probe.bd (4 functions)
  10_sqli.txt         - utils/sqli.bd (6 functions)
  11_sync.txt         - utils/sync.bd (1 function)
  12_dashboard.txt    - utils/dashboard.bd (4 functions)
  (84 functions total - matches `grep -c "^func " <file>` per module)

STATUS: complete as of this write-up (both passes, all 12 modules,
Synthesis written). Not yet acted on - Synthesis.txt's P1/P2 candidates
are recommendations, no code has been changed as a direct result of
this audit yet.
