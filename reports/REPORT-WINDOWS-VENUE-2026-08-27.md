<!-- AUDIENCE: MACHINE -->
# REPORT · WINDOWS LIVED-USE VENUE · 2026-08-27

AMENDED-THROUGH: original, 2026-08-27

## 0 IDENTITY AND SCOPE

Session: operator's own terminal, NOT a fleet seat. No inbox, no STATE line, no allocation.
Ask, in the operator's words: "i need to test in the conditions dan will be using coach in,
which is in claude code on windows, it can be in the CLI", then "the option of you getting out
the way / and i talk to coach directly / here in CLI / but that coach needs to be deployed on
the git windows VM".

DELIVERABLE: a Windows machine running Claude Code with Coach attached, and an SSH line the
operator connects to himself.

STATUS: **NOT DELIVERED.** Infrastructure PROVEN, attachment BLOCKED. Detail in section 4.

## 1 MEASURED · what is true

| claim | value | how measured |
|---|---|---|
| self-hosted runner for dan-coach-windows.yml | **0** | `gh api repos/elliot-backbone/dan-coach/actions/runners` -> total_count 0 |
| private-repo Windows run | REFUSED, 0 steps executed | run 33103254978, check-run annotation |
| refusal text | "the job was not started because recent account payments have failed or your spending limit needs to be increased" | same annotation |
| public-repo Windows run | ADMITTED, executes | runs 33104203194 onward |
| store transferred | 1,230,811,136 B | `gh release view`, byte-equal to local |
| store hash on runner | MATCHES the backup-API copy | workflow step, sha256 8071b8de…c8ee |
| store opens on Windows | user_version 3, integrity ok | gate step, 14 s elapsed |
| server handshake on host | HANDSHAKE OK: server=coach, exit 0 | hs.mjs against the promoted store |
| tools on host | 12 | tools-probe.mjs |
| registration stored on runner | `{"type":"stdio","command":"node","args":["D:\\a\\…\\server.mjs"],"env":{"COACH_ROOT":"C:/coach"}}` | printed from ~/.claude.json on the runner |
| client attachment on runner | **FAILS** `-32000: Connection closed` | `claude mcp list`, 5 consecutive runs |

## 2 PROVEN · what was built and works

- `elliot-backbone/dan-coach-winci`, PUBLIC, two files: one workflow, one README. Verified
  against the git tree, not asserted: `blob 15762 .github/workflows/…`, `blob 2187 README.md`.
  Nothing else, ever. Free Actions minutes; the private repo is billed and refused.
- Credential: fine-grained PAT, contents:read, `dan-coach` only, as Actions secret
  `DAN_COACH_READ_TOKEN`. Never printed, never echoed. Read from the operator's clipboard and
  piped to `gh secret set`. Proven four ways before use, the strongest being a ranged asset
  download returning the bytes `SQLite format 3`.
- Store path: promoted store -> backup-API copy (runbook 7.4a: checkpoint TRUNCATE, -wal read
  back at exactly 0, copy hashed, not the live path) -> private release asset -> runner ->
  hash re-verified there.
- Registration route: `claude mcp add-json --scope user`, runbook 7.6's route of record.
- Two probes, each with its wrong case RUN, not described:
  gate: good `user_version 3, integrity ok` / wrong `unified_store_missing`, exit 1.
  tools: good `TOOLS=12` / wrong `server-exited-1`, exit 1.

## 3 DEFECTS FOUND IN MY OWN WORK, each measured and each fixed

1. **PowerShell here-string vs YAML block scalar.** `'@` must sit at column 0; column 0 ends the
   YAML block. Direct conflict, file would not parse. Caught by READING before running. Both
   probes moved to bash heredocs.
2. **Checkout guard too broad.** Refused any `.sqlite`; failed run 33104203194 on
   `reading-corpus.pilot.sqlite`. The invariant is that a SERVING store must not ride in with
   the code; a disposable corpus is not that. Narrowed to the serving name, and the others are
   now LISTED with sizes rather than blocked or hidden.
3. **PowerShell consumed the `--` separator.** `claude mcp add … -- node $SERVER` reached the
   CLI as `error: missing required argument 'commandOrUrl'`. Moved to bash.
4. **Git Bash delivered one backslash instead of two.** `"C:\coach"` is invalid JSON (`\c` is
   not an escape) so the document failed to parse before Claude saw it:
   `Invalid configuration: : Invalid input`. REPRODUCED EXACTLY on the host in an isolated
   CLAUDE_CONFIG_DIR: two backslashes register, one returns that exact string. Fixed by removing
   the character rather than escaping it harder: Windows accepts `C:/coach`.
5. **A DIAGNOSTIC THAT COULD NOT FAIL, and this is the one worth carrying.** The fallback ran
   the server with stdin at `/dev/null`; the server read EOF and exited cleanly with no output,
   so a healthy server and a broken one were indistinguishable. It cost one full run that
   returned no information. Replaced with a real handshake probe that sends `initialize`, waits,
   and captures stderr, reporting SPAWN ERROR / SERVER EXITED+code+stderr / TIMEOUT / OK.
   CLAUDE.md already carries this rule ("a validator that cannot fail is decoration"); I wrote
   one anyway.

## 4 CONTRADICTS · the open blocker

**The registration is correct and the client still will not attach.**

Read back FROM THE RUNNER, not inferred: `COACH_ROOT` is `C:/coach`, args resolve to the
checked-out server, `add-json` reports success, the store at that root opens with
`integrity ok`. `claude mcp list` then returns `✘ Failed to connect — -32000: Connection
closed` on five consecutive runs.

`Connection closed` means the child process ended and never says why.

HYPOTHESIS UNDER TEST when the operator paused, stated as hypothesis: Claude Code allows an MCP
server 30 s to handshake; the gate step measured 14 s to open this store and run
integrity_check on a cold 1.15 GB file, and the first open is the slowest. A server that misses
the deadline is reported IDENTICALLY to a crash. `MCP_TIMEOUT` is now 120000 at job level.
Run 33107002357 carries both that change and the real handshake probe. Its probe output was not
read before the pause, so THIS IS NOT YET DIAGNOSED and is not claimed as one.

**RELEVANCE TO F-VMT-47 (VM-TESTER).** That finding says Windows MCP attachment fails silently
and that nobody has measured it on current Claude Code. These five runs may be that finding
reproducing on the route the runbook calls the working one. If so they are evidence about a
PRODUCT defect and not only about my plumbing. Stated as a possibility with its reasoning, not
as a conclusion: I have not isolated the cause.

## 5 A CORRECTION I OWE, raised by CODEX-2 and accepted

I reported that `reading-corpus.pilot.sqlite` was "build artifacts and NOT Dan's material",
hedged as "on that evidence". The hedge was warranted and the characterisation was WRONG.
CODEX-2's independent read (`ANALYSIS-COMMITTED-SQLITE.md`, immutable read-only): it is "a
disposable loaded copy of the Dan-coach reading corpus", and `met-def-store.sqlite` holds 82
metric rows including "Dan's exact metric wording".

My inference was from `verbatim_anchor` values pointing at repository paths. That was one field
read shallowly, and it did not support the conclusion I drew from it. THE STRIP WAS MORE
IMPORTANT THAN I FRAMED IT: two of the seven held Dan's material.

CODEX-2 reached the same disposition independently (untrack + gitignore, all 13) and confirms
the applied commit. Same result, better evidence, and the correction stands against me.

## 6 DECIDE · for the master and the operator

1. **Continue or stop.** Five runs, roughly 90 minutes, no session delivered. The infrastructure
   is proven to the last gate. My recommendation: ONE more run to read the handshake probe,
   because it either clears the blocker or names it, and it is already dispatched and paid for.
2. **Git history.** The 13 database files are untracked but their bytes remain in every commit
   that carried them, `a18f283a` included. Excising them rewrites that SHA, and `a18f283a` is
   the serving pin named in the promotion receipt and checked out by this workflow. Reclaiming
   28.7 MB is not worth breaking the commit the product serves from. NOT DONE, operator's call.
3. **F-VMT-47.** If the handshake probe shows the server healthy and the client still refusing,
   that is a measured Windows attachment defect on the route of record, and it belongs to the
   product lane rather than to this venue. I raise it; I do not own it.

## 7 PATHS

- workflow: `elliot-backbone/dan-coach-winci` `.github/workflows/dan-coach-interactive.yml`
- this report: `~/dan-coach-winci/reports/REPORT-WINDOWS-VENUE-2026-08-27.md`, copy in ~/Downloads
- strip commit: `dan-coach` 33b2f09f (13 paths untracked) plus the .gitignore commit before it
- runs: 33103254978 (billing refusal), 33104203194, 33104390786, 33104655356, 33105154467,
  33105519392, 33107002357 (unread)
