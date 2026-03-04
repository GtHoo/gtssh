# gtssh

Cluster SSH tool. Opens a separate xterm window per host plus a small control
window — anything you type in the control window is broadcast to all sessions
simultaneously.

```
┌─ gtssh ──────────────────────────────────────────────┐
│ gtssh  [● web1]  [● web2]  [○ db1]  [✕ db2]  [Quit] │
├──────────────────────────────────────────────────────┤
│ > systemctl restart nginx_                           │
│ broadcasting to 2/4 hosts  (click a button to toggle)│
└──────────────────────────────────────────────────────┘
```

## Requirements

- Python 3 (stdlib only — `tkinter`, `pty`, `socket`, …)
- `xterm` (default mode) or `tmux` (tmux mode)

```bash
sudo apt install xterm tmux   # Debian/Ubuntu
sudo dnf install xterm tmux   # Fedora/RHEL
```

## Usage

```bash
./gtssh host1 host2 host3
./gtssh user@host1 user@host2
./gtssh -l admin host1 host2
./gtssh -p 2222 -i ~/.ssh/mykey host1 host2
./gtssh host1 @webservers host2      # mix direct hosts and cluster groups

# tmux mode — tiled panes, no X display required
./gtssh --tmux @webservers
./gtssh --tmux host1 host2
```

### Options

| Flag | Description |
|------|-------------|
| `-l USER` | SSH login username |
| `-p PORT` | SSH port |
| `-i FILE` | SSH identity file |
| `-o OPT` | Extra SSH option (repeatable) |
| `--tmux` | Use tmux panes instead of xterm windows |
| `--list` | List configured cluster groups |
| `--init` | Create `~/.gtssh/clusters` config file |
| `--proxmox-sync [TARGET]` | Sync running VM IPs from Proxmox into a cluster group (default target: `@hypervisor`) |
| `--cluster-name NAME` | Cluster name to write when using `--proxmox-sync` (default: `proxmox`) |
| `--vmlist PATH` | Path to `vm-list.sh` (auto-detected if omitted) |

## Control window (xterm mode)

| Action | Effect |
|--------|--------|
| Type anything | Broadcast keystroke to all active hosts |
| `Enter` | Send command to all active hosts, clear input |
| Click a green `●` button | Exclude that host from broadcasts (turns grey `○`) |
| Click a grey `○` button | Re-include that host (turns green `●`) |
| Click a red `✕` button | Reconnect to that host (opens a new xterm) |
| `Quit` | Close all xterm windows and exit |

All special keys are forwarded: arrows, `Tab`, `Escape`, `Ctrl+C/D/Z`, F-keys, etc.

You can still type directly in any individual xterm window to send to just that host.

## Tmux mode

`--tmux` opens all hosts as tiled panes in a single tmux session with `synchronize-panes` enabled — keystrokes are broadcast to all panes simultaneously. No X display needed.

| Action | Effect |
|--------|--------|
| Type anything | Broadcast to all panes |
| `Ctrl-b :setw synchronize-panes off` | Disable broadcast |
| `Ctrl-b :setw synchronize-panes on` | Re-enable broadcast |
| `Ctrl-b q` | Show pane numbers |
| `Ctrl-b d` | Detach session |

## Cluster groups

Run `./gtssh --init` to create `~/.gtssh/clusters`, then define groups:

```
# ~/.gtssh/clusters
webservers  web1.example.com web2.example.com web3.example.com
dbservers   user@db1.example.com user@db2.example.com
```

Use groups on the command line:

```bash
./gtssh @webservers
./gtssh @webservers @dbservers
./gtssh @webservers extra-host.example.com
```

## Proxmox integration

`--proxmox-sync` queries a Proxmox host via `vm-list.sh`, extracts IPs of all
running VMs and containers, and saves them as a named cluster in `~/.gtssh/clusters`.

```bash
# Sync @hypervisor group → saves as @proxmox cluster
./gtssh --proxmox-sync

# Sync a specific host or group
./gtssh --proxmox-sync proxmox-node1
./gtssh --proxmox-sync @hypervisor --cluster-name myvms

# Specify vm-list.sh location explicitly
./gtssh --proxmox-sync --vmlist /path/to/vm-list.sh

# Then connect to all synced VMs
./gtssh @proxmox
./gtssh --tmux @proxmox
```

`vm-list.sh` connects to Proxmox over SSH. It tries users in this order:

1. `root` (default)
2. Username from `~/.servers/config` (`SSH_USER=myuser`)
3. Current `$USER`

Proxmox hosts are defined in `~/.servers/hosts`:

```
# name|ip|uri|desc|groups
proxmox|192.168.10.101|||hypervisor
```

## How it works

Each xterm runs a small Python relay script that:

- Creates a pty pair and runs `ssh` on it
- Listens on a Unix socket (`/tmp/gtssh-<pid>-<host>.sock`) for broadcast data
- Bridges xterm keyboard input and broadcast input into SSH's stdin

The main process holds a socket connection to each relay and writes bytes to
broadcast. No X11 event injection (`xdotool`, etc.) is needed.

## Install

Copy `gtssh` anywhere on your `$PATH`:

```bash
sudo cp gtssh /usr/local/bin/gtssh
```
