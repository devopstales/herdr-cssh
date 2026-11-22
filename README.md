# herdr-cssh

ClusterSSH with [Herdr](https://herdr.dev) — one SSH session per host in a
tiled tab, each pane labeled with its host, with keystrokes synced from the
pane you're looking at to the others.

Think [tmux-cssh](https://gitlab.com/peikk0/tmux-cssh) /
[ClusterSSH](https://github.com/duncs/clusterssh) without a Tk GUI: one tab,
many panes, driven through Herdr's local socket API.

## Requirements

- [Herdr](https://herdr.dev) — a terminal workspace manager. Install with:
  ```
  curl -fsSL https://herdr.dev/install.sh | sh
  ```
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
  -c              Use the current Herdr workspace and open a new tab in it
  -n <name>       Name of the workspace or tab (default: cssh)
  -o <ssh args>   Additional arguments passed to the program before the host

% herdr-cssh web{1..4}

% herdr-cssh -o '-p 2222 -l my_user' web{1..4}

% herdr-cssh -c app{1..3}        # new tab in the current workspace
```

`herdr-cssh --list` shows the local Herdr sockets available to this user and
whether each is reachable.

## How it works

`herdr-cssh` talks to a running Herdr server over its local socket API — the
same JSON protocol the `herdr` CLI uses:

1. **Resolve the socket.** It prefers `$HERDR_SOCKET_PATH` (set when you run
   inside a Herdr pane), then the default-session socket
   (`~/.config/herdr/herdr.sock`), then any named-session socket under
   `~/.config/herdr/sessions/`.
2. **Create the layout.** Without `-c` it creates a new Herdr workspace; with
   `-c` it reuses the workspace of the currently focused pane. It then calls
   `layout.apply` with a balanced BSP split tree — one pane per host, each
   running `ssh <host>` (or your `-p` program) and labeled with the host name.
3. **Stay alive and track focus.** It subscribes to Herdr's `pane.focused`
   events to know which pane you're looking at, and mirrors your keystrokes
   from that pane to the others so typing reaches every session at once.
   Press `Ctrl-C` to stop.

The target tab is focused, so the Herdr TUI (or `herdr session attach
<session>`) shows it immediately.

## Layout mapping

| tmux-cssh              | herdr-cssh                                   |
| ---------------------- | -------------------------------------------- |
| new tmux session       | new Herdr workspace                          |
| panes + tiled layout   | tab with a balanced BSP pane layout          |
| pane border label      | pane label (shown in the Herdr TUI)          |
| synchronize-panes      | focus tracking + input fan-out               |
| switch-client / attach | target tab focused                           |

## Notes

- The headless Herdr server (`herdr server`) exits when no TUI client is
  attached. For interactive use, run `herdr` first (or attach a client), then
  run `herdr-cssh` from a shell.
- Pane labels show the host name; Herdr renders them in the TUI.
- The balanced layout splits panes evenly (a 4-host run gives a 2×2 grid, a
  6-host run gives two rows of three).
