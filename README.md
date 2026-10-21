# herdr-muse-restore

Relaunch Muse Code (`muse`) sessions whose Herdr panes came back as bare
shells after a server restart — the same treatment Herdr gives natively to
Claude/Codex panes.

## The problem

Herdr restores workspaces, tabs, panes and cwds after a restart, and it
auto-resumes the agents it knows how to launch. Muse is *detected* but has
no native resume, so restored Muse panes sit at an empty shell prompt while
everything else picks up where it left off.

## What this does

A Herdr startup hook (runs once per server start, after the snapshot is
restored) that:

- reads the pane→session bindings kept fresh by the Muse lifecycle reporter
  ([akshat12/herdr-muse](https://github.com/akshat12/herdr-muse), required —
  see below) at `~/.local/share/herdr-muse/bindings.json`,
- drops stale bindings (pane closed, session store gone),
- skips panes where `muse` already runs (e.g. server came back via
  `--handoff` and never died),
- waits for each remaining pane's shell to become ready (restored shells can
  take a while to spawn; firing `agent start` too early fails with
  `agent_pane_busy` and never retries),
- relaunches each session with
  `herdr agent start <name> --kind muse --pane <id> -- resume <session-id>`,
  mirroring what Herdr's native resume planner does for Claude.

Up to 4 panes relaunch in parallel. Failures are logged, never fatal: the
pane keeps its shell, and the next Muse lifecycle event re-reports state
anyway.

## Requirements

- Herdr ≥ 0.6.10 (uses `agent start --kind muse`, `pane process-info`,
  startup hooks).
- **The bindings file.** This plugin does not discover sessions itself — it
  trusts the exact pane→session map written by the reporter hook from
  [akshat12/herdr-muse](https://github.com/akshat12/herdr-muse). Install that
  first and start one new `muse` session so `bindings.json` exists. Panes
  with no binding are skipped (by design: no guessing, no wrong-session
  resume).

## Install

```bash
herdr plugin install jpcarranza94/herdr-muse-restore
```

or manually:

```bash
git clone https://github.com/jpcarranza94/herdr-muse-restore
herdr plugin link ./herdr-muse-restore
herdr server reload-config
```

Takes effect on the next Herdr server start. No runtime dependencies beyond
`python3` and the `herdr` CLI.

## Knobs (env)

| Variable | Default | What |
|---|---|---|
| `MUSE_RESTORE_PANE_TIMEOUT_S` | `600` | Max wait per pane for its shell to become ready |
| `MUSE_RESTORE_POLL_S` | `3` | Poll interval while waiting |
| `MUSE_RESTORE_BINDINGS` | `~/.local/share/herdr-muse/bindings.json` | Override bindings path (tests) |
| `HERDR_MUSE_RESTORE_DRY_RUN=1` | off | Classify every binding and print the plan without launching |

## Limitations

- Only restores sessions the reporter hook saw (no hook history → no
  resume; unlike newest-per-cwd heuristics this never attaches the wrong
  session to a pane).
- Session restore after restart is per-pane relaunch, not in-place TUI
  reattach — same as Herdr's native resume.

## License

MIT. Companion to the MIT-licensed
[akshat12/herdr-muse](https://github.com/akshat12/herdr-muse) reporter.
