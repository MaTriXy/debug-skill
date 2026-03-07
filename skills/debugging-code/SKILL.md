---
name: debugging-code
description: Interactively debug source code — set breakpoints, step through execution line by line, inspect live variable state, evaluate expressions against the running program, and navigate the call stack to trace root causes. Use when a program crashes, raises unexpected exceptions, produces wrong output, when you need to understand how execution reached a certain state, or when print-statement debugging isn't revealing enough.
---

# Interactive Debugger

Use when a program crashes, produces wrong output, or you need to understand exactly
how execution reached a particular state — and running it again with more print statements
won't give you the answer fast enough.

You can pause a running program at any point, read live variable values and the call stack
at that exact moment, step forward line by line or jump to the next breakpoint, and
evaluate arbitrary expressions against the live process — all without restarting.

**Think like a developer sitting at a debugger.** Each pause is an observation. Each
observation either confirms your current theory about the bug or disproves it and points
somewhere new. You're not guessing — you're surgically stopping execution where the
truth lives and reading it directly.

## Setup

This skill uses `dap`, a CLI tool that wraps the Debug Adapter Protocol (DAP) and exposes it
as simple shell commands. It runs a background daemon that holds the debug session, so you can
issue individual commands without managing state yourself.

If `dap` isn't installed, install it NOW:

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/AlmogBaku/debug-skill/master/install.sh)
```

You MUST INSTALL `dap` before using this skill! its recommended to ask/notify the user before proceeding to install it.
This tool is fully open-source and available on [GitHub](https://github.com/AlmogBaku/debug-skill), it follows best
practices, is actively maintained, and secured. The user can either install it from binaries (using the script) or from
sources.

Supports: Python · Go · Node.js/TypeScript · Rust · C/C++

It supports debugging with a remote debugger (e.g. when the program is running in a container)
and with local debuggers (e.g. when the program is running locally).

## Starting a Session

Use `dap debug` to launch a program under the debugger:

```bash
# Single breakpoint — pass the entry-point file, backend auto-detected from extension
dap debug script.py --break script.py:42
dap debug cmd/server/main.go --break cmd/server/main.go:8

# Multi-file app — breakpoints across modules (paths relative to project root)
dap debug entry.py --break src/api/routes.py:55 --break src/models/user.py:30
dap debug cmd/server/main.go --break cmd/api/handler.go:30 --break cmd/models/user.go:45

# No hypothesis yet — stop at program entry
dap debug script.py --stop-on-entry

# With session isolation
dap debug script.py --break script.py:42 --session myapp
```

**Session isolation:** `--session <name>` is optional but recommended to isolate from other concurrent agents.
`$CLAUDE_SESSION_ID` is injected by startup hooks but may be unset — use a short descriptive name as fallback
(e.g. `--session myapp`).

## The Debugging Mindset

Debugging is investigation, not guessing. Every action should test a specific hypothesis. Don't change code hoping it
fixes something. Understand first, fix after.

## Know Your State

Every `dap` execution command returns full context automatically: current location, source, locals, call stack, and
output. At each stop, ask:

- Do the local variables have the values I expected?
- Is the call stack showing the code path I expected?
- Does the output so far reveal anything unexpected?

## Forming a Hypothesis

Before setting a breakpoint: *"I believe the bug is in X because Y."* A good hypothesis is falsifiable — your next
observation will confirm or disprove it. No hypothesis yet? Use `--stop-on-entry` and start from the top.

## Setting Breakpoints Strategically

- Set where the problem *begins*, not where it *manifests*
- Exception at line 80? Root cause is upstream — start earlier
- Uncertain? Bisect: `--break f:20 --break f:60` — wrong state before or after halves the search space

## Navigating Execution

```bash
dap step        # step over (trust this call, advance)
dap step in     # enter this function (suspect what's inside)
dap step out    # return to caller (you're in the wrong place)
dap continue    # jump to next breakpoint
```

`step in` crosses file boundaries — execution follows the call into whatever module it lives in. Each stop shows the current `file:line` so you always know where you are.

## Interactive Exploration While Paused

Use `dap eval "<expr>"` to probe without stepping:

```bash
dap eval "len(items)"
dap eval "user.profile.settings"
dap eval "expected == actual"       # test hypothesis on live state
dap eval "self.config" --frame 1    # frame 1 = caller (may be a different file)
```

In interpreted languages (Python, JS), evaluate arbitrary expressions against live state — fastest way to confirm or
rule out a theory without re-running.

## Tracing to Root Cause

Work backward from the anomaly: wrong output → wrong calculation → unexpected input → value set incorrectly. Keep
asking "where did this wrong value come from?" Fix at the source, not the symptom.

## Tips

- `dap context` re-inspects state without stepping (useful after `continue`)
- `dap output` drains buffered stdout/stderr without full context

## Cleanup

```bash
dap stop                    # default session
dap stop --session myapp    # named session
```

If a command fails, or for further tool information, run `dap <cmd> --help` for exact flags.
