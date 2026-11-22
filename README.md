# herdr-cssh

ClusterSSH with [Herdr](https://herdr.dev) — one SSH session per host, each
labeled with its host name, with your keystrokes mirrored to every session at
once so typing reaches them all.

Think [tmux-cssh](https://gitlab.com/peikk0/tmux-cssh) /
[ClusterSSH](https://github.com/duncs/clusterssh) without a Tk GUI: one place,
many hosts, driven through Herdr's local socket API.

## Requirements

- [Herdr](https://herdr.dev) — a terminal workspace manager
- Python 3.8+ (stdlib only)
- `ssh` (or any program you pass with `-p`)

## Installation

Copy `herdr-cssh` somewhere in your `$PATH`:

```
cp herdr-cssh ~/.local/bin/
```

## Usage

```
% herdr-cssh -h
Usage: herdr-cssh [options] host [host ...]

Options:
  -h              Show help
  -p <program>    Run <program> <host> instead of ssh <host> (default: ssh)
  -c              herdr backend: open a tab in the current workspace instead
                  of a new one
  -n <name>       Name of the group / workspace / tab (default: cssh)
  -o <ssh args>   Additional arguments passed to the program before the host

% herdr-cssh web{1..4}

% herdr-cssh -o '-p 2222 -l my_user' web{1..4}

% herdr-cssh --list            # which Herdr runtime is available
```

`herdr-cssh` runs in the foreground: it opens the sessions, then blocks,
mirroring your keystrokes to all of them until you press `Ctrl-C`.

## Backends

`herdr-cssh` auto-detects the running Herdr runtime and speaks its socket
API:

| Backend    | Detected when                          | What it does                                            |
| ---------- | -------------------------------------- | ------------------------------------------------------- |
| herdr-core | `$HC_TERMINAL_SOCKET` is set (you're in a `hc`/`herdr-app` terminal) | one SSH **terminal** per host, labeled `cssh=<group>`, `host=<name>`; keystrokes mirrored from the calling terminal to the others |
| herdr      | `~/.config/herdr/herdr.sock` (or `$HERDR_SOCKET_PATH`) is live | a new **workspace** with a **tiled tab** — one labeled pane per host; keystrokes mirrored from the focused pane to the others |

`herdr-cssh --list` shows the active backend and every candidate socket with
its state.

## How it works

1. **Detect the runtime.** herdr-core first (when `$HC_TERMINAL_SOCKET` is
   live), then the herdr JSON API (default-session socket, then named-session
   sockets).
2. **Create the sessions.**
   - *herdr-core*: `terminal.run` once per host, each with a `cssh`/`host`
     label, no parent (so they survive this process).
   - *herdr*: `workspace.create` (or reuse the current workspace with `-c`),
     then `layout.apply` with a balanced BSP split tree — one pane per host,
     each running `ssh <host>` and labeled.
3. **Mirror keystrokes.**
   - *herdr-core*: observes the calling terminal's surface and replays input
     to the other terminals via `terminal.send`.
   - *herdr*: subscribes to `pane.focused` events, tracks the focused pane,
     and mirrors input to the others.

## Notes

- The headless `herdr` server exits when no TUI client is attached; for the
  herdr backend, run `herdr` first (or attach a client), then run
  `herdr-cssh` from a shell.
- Unreachable hosts: the SSH terminal exits and (in herdr-core) is auto-removed;
  the remaining sessions keep syncing.
- The balanced herdr layout splits panes evenly (4 hosts → 2×2 grid, 6 → two
  rows of three).
