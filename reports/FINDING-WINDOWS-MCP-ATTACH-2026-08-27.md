<!-- AUDIENCE: MACHINE -->
# FINDING · WINDOWS MCP ATTACHMENT · read-only investigation · 2026-08-27

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
