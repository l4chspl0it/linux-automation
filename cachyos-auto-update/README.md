# CachyOS Auto-Update

Unattended daily updates for CachyOS using `arch-update`, an `expect` script and a systemd **user** timer. No typing `y`, no password prompt, no manual run.

## How it works

```
Login -> systemd user timer -> daily-update.exp -> arch-update -> pacman (passwordless sudo rule)
```

| Piece | Job |
|---|---|
| `daily-update.exp` | Answers every `arch-update` prompt automatically |
| `/etc/sudoers.d/arch-update` | Lets the update run without a sudo password |
| `cachy-update.service` | Runs the script |
| `cachy-update.timer` | Triggers it 5 min after login, then every 24h |

## Requirements

- CachyOS (Arch-based, systemd)
- `arch-update` and `expect` installed
- `sudo` (also supports `sudo-rs`, `doas`, `run0` in arch-update itself, but this guide uses sudo)

## Setup

### 1. Find what arch-update elevates

`arch-update` is a bash script that calls privileged commands through a `su_cmd` variable, so a plain `grep sudo` finds nothing. Use this instead:

```bash
grep -rhoE '\$\{su_cmd\}"? +[^ ]+( +[^ ]+)?' /usr/bin/arch-update /usr/share/arch-update/lib/*.sh | sort | uniq -c
```

Commands found: `pacman`, `paccache`, `systemctl restart`, `checkservices -F`, `pacdiff`.

`pacdiff` is interactive (pacnew merges), so it is left out of the unattended rule.

Get the full paths:

```bash
command -v pacman paccache systemctl checkservices pacdiff
```

### 2. Passwordless sudo for those commands

CachyOS has no `vi`, so point visudo at nano:

```bash
sudo EDITOR=nano visudo -f /etc/sudoers.d/arch-update
```

Add one line (replace `youruser`):

```
youruser ALL=(root) NOPASSWD: /usr/bin/pacman, /usr/bin/paccache, /usr/bin/checkservices, /usr/bin/systemctl restart *
```

Test it:

```bash
sudo -k
sudo -n pacman --version
```

You should see the pacman banner with no password prompt.

### 3. The expect script

Save as `~/Documents/Scripts/daily-update.exp`:

```tcl
#!/usr/bin/expect -f

set timeout -1
log_user 1

puts ""
puts "=========================================="
puts "       CachyOS Daily Update"
puts "=========================================="
puts ""

spawn /usr/bin/arch-update

expect {
    -re {Proceed with update\? \[Y/n\]} {
        send -- "y\r"
        exp_continue
    }
    -re {Proceed with installation\? \[Y/n\]} {
        send -- "y\r"
        exp_continue
    }
    -re {Would you like to remove them from the cache now\?} {
        send -- "y\r"
        exp_continue
    }
    -re {Select the service\(s\) to restart.*:} {
        send -- "0\r"
        exp_continue
    }
    -re {Press "enter" to quit} {
        send -- "\r"
        exp_continue
    }
    eof {
        puts ""
        puts "CachyOS update process finished."
    }
}
```

Notes:
- `set timeout -1` stops long downloads timing out.
- Full path in `spawn` so it works without your shell PATH.
- The sudo password block is removed on purpose (step 2 replaces it).

Make it executable and test:

```bash
chmod +x ~/Documents/Scripts/daily-update.exp
~/Documents/Scripts/daily-update.exp
```

### 4. systemd user service

`~/.config/systemd/user/cachy-update.service`

```ini
[Unit]
Description=Daily CachyOS update

[Service]
Type=oneshot
ExecStart=/home/youruser/Documents/Scripts/daily-update.exp
```

### 5. systemd user timer

`~/.config/systemd/user/cachy-update.timer`

```ini
[Unit]
Description=Daily CachyOS update

[Timer]
OnStartupSec=5min
OnUnitActiveSec=24h

[Install]
WantedBy=timers.target
```

- `OnStartupSec=5min` runs 5 min after the user session starts (PC is not always on).
- `OnUnitActiveSec=24h` repeats daily if you stay logged in.
- Logging out and back in the same day can trigger another run.

### 6. Enable

```bash
mkdir -p ~/.config/systemd/user
systemctl --user daemon-reload
systemctl --user enable --now cachy-update.timer
systemctl --user list-timers
```

## Verify

Run the real service the same way the timer will:

```bash
systemctl --user start cachy-update.service
journalctl --user -u cachy-update.service -f   # second terminal
systemctl --user status cachy-update.service
```

Look for `status=0/SUCCESS`.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `exists but is not an executable file` | `chmod +x daily-update.exp` |
| `visudo: no editor found` | `sudo EDITOR=nano visudo -f ...` |
| `arch-update: command not found` in service | Use `/usr/bin/arch-update` in `spawn` |
| Script hangs | Check the journal for the last prompt, add a matching `-re` branch |
| Sudo asks for a password | Re-check paths in the sudoers rule with `sudo -n <cmd>` |
| Timer never fires | Confirm with `systemctl --user list-timers`, user timers only run while logged in |

## Security notes

- Passwordless `pacman` is effectively root for anything running as your user. Fine for a personal desktop, weigh it on machines with untrusted workloads.
- Unattended updates on a rolling release can occasionally break things (kernel, drivers, Arch news manual interventions). Keep a snapper or Timeshift snapshot.
- `systemctl` is restricted to `restart` only.

## Uninstall

```bash
systemctl --user disable --now cachy-update.timer
rm ~/.config/systemd/user/cachy-update.{service,timer}
sudo rm /etc/sudoers.d/arch-update
```

## Credits

Built with the [arch-update](https://github.com/Antiz96/arch-update) tool.
