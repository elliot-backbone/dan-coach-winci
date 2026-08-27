<!-- AUDIENCE: MACHINE -->
# FINDING · WINDOWS MCP ATTACHMENT · read-only investigation · 2026-08-27

AMENDED-THROUGH: 2026-08-27 §8 (root cause found; §3-§4 hypotheses superseded, marked inline)

SCOPE: the operator asked one question. Are the five failed runs an instance of F-VMT-47?
METHOD: read-only. Existing run logs, the Bill package on disk, no new dispatch, no writes.
ANSWER: **NO, AND THE EVIDENCE POINTS THE OTHER WAY.** Detail below, with what is not known.

## 1 THE DECISIVE MEASUREMENT, and it disproves my own hypothesis first

Timestamps from run 33107002357, differenced:

| interval | elapsed |
|---|---|
| store open + `PRAGMA integrity_check` (bash, same node, same COACH_ROOT) | **11.3 s** |
| `claude mcp list` start -> `Failed to connect` | **1.1 s** |

**A 1.1 SECOND FAILURE IS NOT A TIMEOUT.** The server cannot even have reached its store: the
same store takes 11.3 s to open on that machine, in that run, seconds earlier. The child died
almost immediately.

MY MCP_TIMEOUT HYPOTHESIS IS THEREFORE WRONG AND IS WITHDRAWN. It was set (`MCP_TIMEOUT:
120000`, confirmed in the log) and the failure time did not move. I proposed it as "most likely
cause, measured rather than guessed"; the measurement that would have tested it was the one I
had not taken. Recording it because a discarded hypothesis with its disproof is worth more than
a quiet correction.

## 2 WHY THIS IS NOT F-VMT-47

F-VMT-47 states two things: the SHIPPED route (`--mcp-config` + `--strict-mcp-config`) attaches
ASYNCHRONOUSLY and races the first prompt, failing silently; and the USER-SCOPE route works
SYNCHRONOUSLY and is the one that attaches on Windows.

What I measured is the USER-SCOPE route FAILING, loudly, in 1.1 s, five runs running.

That is not the finding's failure mode. It is closer to its inverse: the route F-VMT-47 names as
the working one is the route that does not work here. And the failure is not silent, which is
the property that made F-VMT-47 dangerous.

**THE SHIPPED ROUTE WAS NEVER EXERCISED.** The F-VMT-47 measurement step exists in the workflow
but reports `skipped` in all four runs that reached it, because it sits after the failing step.
So this session has produced **NO EVIDENCE AT ALL** about F-VMT-47, in either direction. Anyone
reading these runs as support for it, or against it, would be reading something that did not
run.

## 3 THE CONTROLLED COMPARISON, and it is a good one

**SUPERSEDED 2026-08-27 BY §8.** Both residual variables below (path separators, store size)
were subsequently tested and BOTH ARE DEAD: run 33108029320 failed identically with a fully
forward-slash path, on client 2.1.247 and on the Bill pin 2.1.238. The section stands as the
reasoning that motivated the probe, not as a live hypothesis.

`bill-coach-winci` passes this same gate on the same runner class. Read from its workflow and
its shipped package rather than recalled:

| | Bill (PASSES 2026-08-25) | Dan (FAILS x5, 2026-08-27) |
|---|---|---|
| runner | windows-latest | windows-latest |
| scope | user (`~/.claude.json` mcpServers) | user (`add-json --scope user`) |
| `command` | `"node"` | `"node"` |
| assertion | `mcp list` matching `(Connected\|check)` | identical |
| args separators | forward slash throughout | **mixed**: `D:\a\...\dan-coach-winci/coach-plugin/...` |
| store bytes | 45,203,456 (44.6 MB + 585 KB) | **1,230,811,136** |

Route, scope, command form and assertion are IDENTICAL and Bill's connects. So the mechanism
F-VMT-47 describes is not what separates them. Two variables remain: the mixed path separators
in the args, and a store 27x larger.

## 4 WHAT THE 1.1 SECONDS ARGUES, stated as inference and labelled as such

**SUPERSEDED 2026-08-27 BY §8.** The path inference below was tested and disproven. What the
1.1 seconds actually argued — child dies before touching the store — was correct; the cause
was the entry guard, not the path.

Store size explains a slow open. It does not explain a death before the store is touched. That
makes the SIZE variable the weaker of the two, and the PATH variable the stronger, on this
evidence alone.

I have not proven it. The measurement that would is a single run that registers the same server
with a fully forward-slash args path and reads `mcp list`. It is one step, it is cheap, and I
have not run it because the operator paused the lane.

NOT RULED OUT, and each needs its own probe: the environment Claude Code hands a spawned server
on Windows (a stripped env kills a Windows child instantly, which fits 1.1 s); node 24.19 versus
the 26.7 this was proven on locally; and a fast crash inside the server's own startup before any
store access.

## 5 WHAT I GOT WRONG IN THE INSTRUMENT

The handshake probe added specifically to answer this question **failed for its own reason**:
`SPAWN ERROR: spawn /c/hostedtoolcache/windows/node/24.19.0/x64/node ENOENT`. I passed
`command -v node`, which in Git Bash returns a POSIX path that a Windows spawn cannot resolve.
The probe never reached Coach and therefore says nothing about it. That is the second diagnostic
of mine in this session to return no information; the first was the `/dev/null` stdin fallback
that could not fail.

## 6 FINDINGS, ranked

1. **The MCP_TIMEOUT explanation is disproven** by a 1.1 s failure against an 11.3 s store open.
2. **These runs are not evidence about F-VMT-47**, for or against, because the route it concerns
   never executed. The step is present and `skipped` in four runs.
3. **They may be a DIFFERENT and unrecorded Windows attachment defect**, and if the path-separator
   probe clears it, they are instead my own plumbing. Both remain open.
4. **Bill is the control that makes this tractable**: identical route and command form, passing,
   on the same runner class, five days ago.

## 7 WHAT I DID NOT DO

No runs dispatched. No files changed outside this report. No writes to any store. The pause
holds.


## 8 ROOT CAUSE · found in run 33108029320, closing §3, §4 and §6.3

**`server.mjs:573` at a18f283a:**

```js
if (import.meta.url === `file://${process.argv[1]}`) {
```

POSIX: `argv[1]` is `/Users/...`, the concatenation equals `import.meta.url`, the server
starts. WINDOWS: `argv[1]` is `D:\a\...\server.mjs` while `import.meta.url` is
`file:///D:/a/...` — STRUCTURALLY UNEQUAL IN EVERY SPELLING. The module loads, the guard is
false, nothing holds the event loop, node EXITS 0, the client reports `Connection closed`.

THE DECISIVE MEASUREMENT, from the repaired handshake probe on the runner:
`SERVER EXITED code=0 signal=null`, no stderr. A clean voluntary exit without answering
initialize. Consistent with every prior number: 1.1 s vs the 11.3 s store open (the store is
never reached), both client versions, both path forms.

BOTH §3 RESIDUAL VARIABLES TESTED AND DEAD in the same run: forward-slash path fails
identically; 2.1.238 fails identically to 2.1.247.

THE SHIPPED PRODUCT ALREADY KNEW: `windows/runtime/coach-mcp-entry.mjs`'s header states the
canonical direct-entry comparison is POSIX-shaped and exists precisely to own the stdio
lifecycle instead. So the packaged activation route is correct by design on this point, and
**RUNBOOK 7.6's FALLBACK COMMAND (`claude mcp add ... -- node ...\server.mjs`) IS A
STRUCTURAL NO-OP ON WINDOWS.** Its 12-tool EXPECT was only ever validated on macOS,
VM-TESTER's pair-run included.

F-VMT-47 REFRAMED: with `server.mjs` as the target, BOTH routes register a server that exits
instantly on Windows; route choice is unreachable until the entry is Windows-safe. The shipped
product's exposure reduces to the async-attach question only, which remains open and unmeasured.

PROPOSED REMEDIATION, carried with the finding per the standing rule:
1. RUNBOOK s7.6: the fallback registers `windows\runtime\coach-mcp-entry.mjs`, never
   `server.mjs`.
2. PRODUCT, one line at `server.mjs:573`:
   `import.meta.url === pathToFileURL(process.argv[1]).href` — correct on both platforms.
Landing spots: RUNBOOK-PROMOTION-SMOKE-VM-2026-08.md s7.6 and the product defect register.
Routing is the coordinator's.

THIS LANE'S FIX: a wrapper lifting the server.mjs entry block verbatim minus the guard,
proven on the host (HANDSHAKE OK server=coach, TOOLS=12; heredoc body diffed identical to the
proven file). Dispatched as run 33108478227.
