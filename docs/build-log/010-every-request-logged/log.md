# Log: a log line for every request

Status: done (2026-11-17, last commit b83fb6a).

One entry per commit-sized unit, newest last, written when the unit is ready
to commit. The commit hash is filled in once the maintainer has committed.

## 2026-11-17 — Every request logged, `/health` at `INFO`

- **What changed:** `crates/gl-node/src/request_log.rs` — the throttle is
  gone: every request writes its line, at `INFO` below 400 (a passing
  `/health` included) and `WARN` for 4xx, 5xx and a request cut short; no
  `suppressed=`, no expiry summaries. `RequestLogState` keeps only the twin
  limiter that works out `client`; `is_health` and the `DEBUG` level went.
  `ledgers.rs`: `test_node::node_with` takes no log. `test_support.rs`: the
  capture helpers rebuild tracing's interest cache once their subscriber
  exists (below). README (*Request log*, *Log volume on a public bind*),
  CHANGELOG (the unreleased 009 bullets rewritten), OPERATIONS (*Requests*,
  the command table), the tracker (MA-1).
- **Tests:** `every_request_logs_a_line_of_its_own_strangers_included`,
  `strangers_behind_one_proxy_are_logged_with_their_forwarded_address`,
  `a_passing_health_check_logs_at_info` replace the three throttle and
  `DEBUG` tests; `a_request_cut_short_still_logs_a_line` renamed; the rest
  as in 009.
- **Gate:** `scripts/gate.sh` — 1772 passed, 0 failed, 0 ignored (last run,
  after the second review's fixes; three more runs of the 242 log-capturing
  tests, all passed).
- **Revert checks:** `scripts/revert_specs/node_logs.json` — NL-R5, NL-R6
  and NL-R12 retired (what they pinned is gone); NL-R17 (a request without
  a note left out) and NL-R18 (a passing `/health` left out) new; NL-R10's
  anchor follows the code; NL-R3 also runs the forwarded-strangers test.
  18 of 18 bite.
- **Departures from the plan:** the capture helpers keep a second, idle
  dispatcher alive for the whole test process (`test_support.rs`,
  `keep_two_dispatchers`). One run captured nothing in
  `a_control_character_in_a_path_or_operation_name_cannot_start_a_line`.
  The cause (found by the review, in tracing-core 0.1.36's `callsite.rs`):
  while exactly one dispatcher is alive, tracing computes a log statement's
  interest from the *current thread's* default, so a test thread with no
  subscriber that reaches a statement first caches it as wanted by none,
  and a capture running elsewhere records nothing. A first fix, rebuilding
  the interest cache once a capture's subscriber existed, made it worse in
  `capture_warnings_async` (rebuilt before its subscriber was the thread's
  default: the L-158 test failed serially, and in 3 of 5 parallel runs of
  `ledgers::tests`); it is gone. With two dispatchers always alive, interest
  is computed over every live one, under tracing's lock: the serial
  reproduction passes, and so did five runs of `ledgers::tests` and six of
  the request-log, ledger and guard tests together.
- **Review:** one important finding (the first fix above, replaced);
  taken: comments that still said "throttled" or "a line of its own", the
  tracker's ID (MA-1, not A1, which `RESIDUALS.md` uses), the health test
  captured under the node's default filter.
- **Added in the same unit — strangers' lines under their own target:**
  `request_log.rs` writes a line without a note under `http::stranger`, so
  `RUST_LOG=…,http::stranger=off` leaves strangers out and keeps key
  holders' lines; `http=off` still covers both. `openledger logs` matches
  both targets for `--http` and gains `--no-strangers`;
  `run_local_node.sh` gains `--no-strangers`. Test
  `stranger_lines_can_be_turned_off_alone`; NL-R15 re-anchored, NL-R19 (a
  stranger's refusal at `INFO`) and NL-R20 (every line under `http`) new:
  `node_logs.json` 20 of 20 bite. Gate: 1772 passed, 0 failed, 0 ignored.
  The installer's smoke test (offline, 0.4.0 image, fake `docker`): 38 of
  38. Live, from source: by default four lines (two `/health`, the operator's
  query, a wrong key); with `--no-strangers` the operator's query only. A
  rehearsal check for `--no-strangers` is added; the rehearsal was not
  re-run.
- **Second review:** nothing important. Taken: two idle dispatchers (so
  tracing counts more than one before the first capture registers), and the
  helper called from the crate's other log captures too (`config.rs`,
  `p2p_service.rs` twice, `anti_entropy_alpn.rs`, `graphql/plaid.rs`); the
  rehearsal's `--no-strangers` check also wants a key holder's line; the
  docs say every `/health` is a stranger's line, a keyed one too (it checks
  the key itself, with no note); a `Bug-catch:` naming NL-R19; a comment
  tying `run_local_node.sh`'s fallback to `DEFAULT_LOG_FILTER`.
- **The same uncommitted diff also holds L-159 / T105** (the images no
  longer set `RUST_LOG`; `docs/audits/OUTSTANDING_TASKS_11_16.md`), at the
  maintainer's word: one commit for both.
- **Added in the same unit — the switches:** `openledger request-log
  [on|off|key-holders]` in the installer (above, *plan.md*) and
  `run_local_node.sh --no-request-log`. Offline smoke test, the 0.4.0 image
  with a fake `docker`: 22 of 22 for `request-log` (each value's line, the
  key kept, one `RUST_LOG` line, mode 0600, the container recreated, the
  same value twice restarting nothing, a hand-written `RUST_LOG` refused
  and untouched, bad values refused) and 38 of 38 for `logs`, as before.
  Three rehearsal checks added; the rehearsal was not re-run.
  The L-159 test (`l159_no_shipped_image_sets_rust_log`) now refuses only a
  line that sets `RUST_LOG` for the node — an image's `ENV …RUST_LOG…`, a
  compose service's `- RUST_LOG=…` or `RUST_LOG: …` — since the
  `request-log` arm names it to write the operator's env file (the gate
  caught the coarse grep); L159-R1..R3 still bite. Live: `run_local_node.sh
  --no-request-log`, no request line, the rest of the log as before. Gate:
  1772 passed, 0 failed, 0 ignored.
- **Third review** (the switches): two important findings, both taken.
  (1) A restart that failed left the env file saying the new value; the
  same command again then said "already" and started nothing. It now always
  runs `docker compose … up -d node` and the health check (compose changes
  nothing when both agree), and `openledger request-log` alone says when
  the running node was not started with what the file says, or is not
  running. (2) The rehearsal's stranger probe went to a path Caddy answers
  itself; the checks now send a stranger's 401 to the node, read the
  container's own `RUST_LOG` (`docker inspect`), exercise `off`, and assert
  no stranger line at all after `key-holders` (every health probe is one).
  Also taken: refused during an unfinished upgrade (`previous/pending`); a
  `RUST_LOG` in any form compose reads (`export`, spaces, `:`, empty), more
  than one, or one in `secrets.env` counts as hand-written; asking which
  takes no lock; the L-159 test reads the installer's two generated files
  (its image's Dockerfile and `compose.yml`) and refuses any line naming
  `RUST_LOG` there, as in the `Dockerfile` and the demo stack; a new test,
  `the_request_log_switches_build_on_the_nodes_default_filter`, ties both
  scripts' copies to `DEFAULT_LOG_FILTER` (NL-R21: 21 checks). Offline
  smoke test: 44 of 44 for `request-log` (a failed restart, then the same
  command again; five hand-written forms, two lines, `secrets.env`, an
  unfinished upgrade), 38 of 38 for `logs`. The harness empties its files
  inside the container: deleted on the Mac first, Docker Desktop's shared
  folder refused the next append (a harness fault, six false failures).
- **Rehearsal:** `scripts/rehearse_install.sh` — passed on ubuntu-24.04 and
  debian-12 (88 checks, 44 each), after the third review's fixes: on each,
  a stranger's request logged under `http::stranger` and left out by
  `--no-strangers` while a key holder's stayed; `request-log` on by
  default; `key-holders` and `off` each started the recreated container
  with the expected `RUST_LOG` (`docker inspect`); `key-holders` logged the
  key holder's request and no stranger line; `off` logged the node's own
  lines and no request line; `on` removed the line and the variable, served,
  and logged a stranger's `/health` again. Gate: 1773 passed, 0 failed, 0
  ignored.
- **Commit:** b83fb6a

## 2026-11-24 — `kind=` and `fields=`: what ran (follow-up)

- **What changed:** the request line names what ran, between `operation=`
  and `user=`: `kind=` (`query`, `mutation` or `subscription`, the
  maintainer's ask on 2026-11-24: "what operation was made, be it a query or
  a mutation, without seeing the request data") and `fields=`, the root fields of the operation the node parsed
  (`fields=syncBank`, `fields=bankInbox,connectedBanks`), whatever the
  client named its request, or if it named none — `operation` is only the
  client's label (the maintainer asked for this after the first Plaid
  sandbox run, where every line said `operation=-`). A schema extension,
  `request_log::LogFields`, reads them from the node's own parse (no second
  parse) into a slot the GraphQL handlers give the request, on the ledgers'
  schema after `AccessControl` — so a refused request still says what it
  asked — and on the node endpoint's. The field, never its alias, arguments
  or values; each once, in the query's order; fragments at the root
  followed, each once; the operation the schema runs, picked as
  `Schema::execute` picks it (by `operationName`, or the query's only one,
  named or not; none, `-`, where it runs none); a name the schema lacks shown (syntax-checked, at most 64 characters, `?`
  otherwise); at most eight, then `+N`, and fewer when the line has no room
  under 512 bytes (the rest of the line is measured first); `-` when no
  query was parsed. README *Request log*, CHANGELOG *Added* (a field
  inserted: something reading by position must adapt);
  `scripts/rehearse_install.sh`'s checks take it (not run: it needs the
  Lima VMs).
- **Tests:** `fields_say_what_ran_whatever_the_client_named_it` (ten
  requests: unnamed, aliases, two operations, fragments, an unknown field,
  eleven fields, a query that does not parse, an operation named in the
  query alone, an `operationName` the query lacks, two operations and no
  name),
  `a_refused_request_still_says_what_it_asked` (a read key's
  `openAccount`), `the_node_endpoints_line_says_what_ran`;
  `every_line_is_under_512_bytes` adds a 64-character operation with twenty
  63-character fields; the four lines' tests and the shipped binary's take
  the field.
- **Gate:** `scripts/gate.sh` — 1778 passed, 0 failed, 0 ignored.
- **Revert checks:** NL-R23 to NL-R33 in `node_logs.json` (the extension
  off either schema, the alias logged, no cap, no room limit, the fields
  read after the role check, `operationName` ignored, no fragment followed,
  a lone named operation taken for none, a mutation logged as a query, no
  kind on the line) — 11 of 11 bite. NL-R2 and
  `users.json`'s U-R1 anchored on code this changed: re-anchored, both
  bite; every anchor in the 18 files occurs once (458 checks).
- **Review of `kind=`:** nothing important. Taken: README says `kind` is
  `-` only when no operation is picked (a refused role, a field the node
  lacks still show their kind) and lists the last `-` for `fields` (a
  spread of a fragment the query lacks); the CHANGELOG's request-log field
  list names `kind` and `fields`, and its by-position warning is gone (the
  request log is unreleased); the module doc names `kind`.
- **Review:** one important finding, fixed: a query whose one operation is
  named, sent without `operationName`, runs (async-graphql files a named
  operation under `Multiple` even alone, and runs it when it is the only
  one) but logged `fields=-` — any key holder could hide what it ran; the
  operation is now picked exactly as `Schema::execute` picks it, with three
  more cases and NL-R29 to NL-R31 (the operation choice and fragments had
  no revert check). Taken: README says what was asked shows even when it
  did not run (`@skip`, a missing field, a refused role) and lists every `-`;
  NL-R27's line is about 750 bytes, not 1,500; two stale doc comments.
  Reported, not this change: async-graphql's own recursive-depth check
  follows fragment spreads without remembering them, so a small query of
  fragments that each spread the next twice may cost about 2^31 steps
  before the depth limit (not run; needs a key).
- **Also:** `007`'s last entry gets its commit, `62fd6be`.
- **Departures from the plan:** none (a follow-up).
- **Commit:** —
