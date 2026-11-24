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
| herdr-app  | `$HC_TERMINAL_SOCKET` is set (you're in a `hc`/`herdr-app` terminal) | a **host grid in the tab you ran `herdr-cssh` from**: that pane **becomes the first host's session** (ssh runs in it) and the remaining hosts get one pane each (`pane.split` halving right/down) — exactly N host panes, no new space or tab, no leftover source pane. Keystrokes typed in the first pane are mirrored to the others; `Ctrl-C` stops the sync, the created host panes close, and that pane is returned to its shell. |
| herdr      | `~/.config/herdr/herdr.sock` (or `$HERDR_SOCKET_PATH`) is live | a new **workspace** with a **tiled tab** — one labeled pane per host; keystrokes mirrored from the focused pane to the others |

`herdr-cssh --list` shows the active backend and every candidate socket with
its state.

## How it works

1. **Detect the runtime.** herdr-app/herdr-core first (when `$HC_TERMINAL_SOCKET`
   is live), then the herdr JSON API (default-session socket, then named-session
   sockets).
2. **Build the tiled layout.**
   - *herdr-app*: the pane you ran `herdr-cssh` from becomes the first host's
     session (ssh is spawned in it as a child process); the remaining hosts
     get one pane each via `pane.split --direction right|down` halving
     splits. Each host terminal is labeled `cssh=<group>`, `host=<name>`.
   - *herdr*: `workspace.create` (or reuse the current workspace with `-c`),
     then `layout.apply` with a balanced BSP split tree — one pane per host,
     each running `ssh <host>` and labeled.
3. **Mirror keystrokes.**
   - *herdr-app*: the source pane's tty is put in raw mode; every keystroke
     is replayed on the remaining host panes (interactive attach,
     herdr-core framed protocol) and fed to the first host. For ssh the
     first host runs with `SSH_ASKPASS` forced to a small helper that
     prints a password file `herdr-cssh` writes from your first typed line
     — so you type the password once and it reaches all hosts — and
     command input goes over a stdin pipe (with `-tt`, so the remote shell
     gets a pty and echoes like the other panes). `Ctrl-C` (0x03) is
     forwarded to all hosts, then the sync stops.
   - *herdr*: subscribes to `pane.focused` events, tracks the focused pane,
     and mirrors input to the others.

## Notes

- Password auth: type the password once at the first prompt — it is
  delivered to every host (see above). The password file is removed on
  stop.
- A host (pane 2..N) whose SSH session exits (unreachable, auth failure,
  `exit`) closes its pane; the rest of the grid stays and keeps syncing. If
  the first host's session ends, its pane stays and typing from it reaches
  the other panes only. Use `-o` to add ssh args
  (`-o ConnectTimeout=5`, `-o BatchMode=yes`, `-J` jump hosts, etc.).
- The headless `herdr` server exits when no TUI client is attached; for the
  herdr backend, run `herdr` first (or attach a client), then run
  `herdr-cssh` from a shell.
- The herdr-app backend needs the Herdr window open (it drives the running
  window). The herdr backend works headless.
