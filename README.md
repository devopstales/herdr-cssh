# herdr-cssh

ClusterSSH with [Herdr](https://herdr.dev) — one SSH session per host, each
labeled with its host name, with your keystrokes mirrored to every session at
once so typing in any pane reaches them all.

Think [tmux-cssh](https://gitlab.com/peikk0/tmux-cssh) /
[ClusterSSH](https://github.com/duncs/clusterssh) without a Tk GUI: one place,
many hosts, driven through Herdr's local socket API.

## Requirements

- [Herdr](https://herdr.dev) — a terminal workspace manager
- Python 3.9+ (stdlib only)
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
  -c              open the group in the current workspace/space instead of a
                  new one
  -n <name>       Name of the group / workspace / tab (default: cssh)
  -o <ssh args>   Additional arguments passed to the program before the host

% herdr-cssh web{1..4}

% herdr-cssh -o '-p 2222 -l my_user' web{1..4}

% herdr-cssh --list            # which Herdr runtime is available
```

`herdr-cssh` opens the sessions and exits. The group then lives on inside
Herdr — focus any pane and type; keystrokes reach every pane of the group.

## Backends

`herdr-cssh` auto-detects the running Herdr runtime and speaks its socket
API:

| Backend    | Detected when                          | What it does                                            |
| ---------- | -------------------------------------- | ------------------------------------------------------- |
| herdr-app  | `$HC_TERMINAL_SOCKET` is set (you're in a `hc`/`herdr-app` terminal) | a **tiled space** in the running Herdr window: `space.new` (or a tab with `-c`) for the first host, then `pane.split` right/down for the rest, so N hosts fill a grid — the tmux-cssh split view. |
| herdr      | `~/.config/herdr/herdr.sock` (or `$HERDR_SOCKET_PATH`) is live | a new **workspace** with a single **tiled tab** — one labeled pane per host (or a tab in the current workspace with `-c`) |

`herdr-cssh --list` shows the active backend and every candidate socket with
its state.

## Keys

Type these in any pane of the group. The prefix is `Ctrl-]`.

| Keys | Action |
| --- | --- |
| `Ctrl-]` `t` | Toggle broadcast for all panes. Labels show `[solo]` while it is off, so you can type in one pane. |
| `Ctrl-]` `q` | Close all the host panes |
| `Ctrl-]` `]` | Send a literal `Ctrl-]` to the program |

## How it works

1. **Detect the runtime.** herdr-app/herdr-core first (when `$HC_TERMINAL_SOCKET`
   is live), then the herdr JSON API (default-session socket, then named-session
   sockets).
2. **Build the tiled layout.**
   - *herdr-app*: `space.new --name <group> --folder <cwd> --argv [ … ]` for the
     first host (or `tab.new` in the current space with `-c`), then
     `pane.split --direction right|down` on the first pane for each remaining
     host, tiling them into a grid. Each terminal is labeled `cssh=<group>`,
     `host=<name>`.
   - *herdr*: `workspace.create`, then `layout.apply` with a balanced BSP split
     tree — one pane per host, each labeled with its host. With `-c`, the
     layout is applied as a new tab of the current workspace instead.
3. **Mirror keystrokes.** Herdr cannot report what you type in a pane, so every
   pane runs its program through a small pty wrapper (`herdr-cssh --pane`, an
   internal mode) that:
   1. runs the program on a pty,
   2. passes your keystrokes to it, and
   3. forwards them to the other wrappers over unix datagram sockets in a
      temporary directory under `/tmp`.

   Received keystrokes are written only to the local program, so nothing loops.
   The launcher writes the group state and exits; the directory is removed when
   the last pane exits.

## Notes

- A host whose SSH session exits (unreachable, auth failure, `exit`) keeps its
  pane open with `exited with status N. Press Enter to close.` so the error can
  be read; press Enter to close it. The rest of the grid stays and keeps
  syncing. Use `-o` to add ssh args (`-o ConnectTimeout=5`,
  `-o BatchMode=yes`, `-J` jump hosts, etc.).
- Keystrokes typed before every pane has started may not reach all panes.
- The headless `herdr` server exits when no TUI client is attached; for the
  herdr backend, run `herdr` first (or attach a client), then run
  `herdr-cssh` from a shell.
- The herdr-app backend needs the Herdr window open (it drives the running
  window). The herdr backend works headless.
