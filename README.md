# no-sleep

Keep a laptop awake — lid closed, idle, on battery — until you toggle it off.
Linux (systemd/GNOME) and macOS.

**Got an AI agent?** (Claude Code, etc.) — paste this and you're done:

```text
Set up no-sleep on this machine: fetch
https://raw.githubusercontent.com/Hugo0/no-sleep/main/no-sleep
review the script, install it as an executable named `no-sleep` on my PATH
(e.g. ~/.local/bin), run `no-sleep` to verify it reports state, and tell me
the on/off commands.
```

```
$ no-sleep on
☕ no-sleep ON  — close the lid and keep working.
   • blocks lid-close suspend (logind) + GNOME idle auto-suspend
   • run 'no-sleep off' when you're done
   • heat builds in a closed bag; lock the screen first

$ no-sleep off
🌙 no-sleep OFF — normal: lid-close and idle sleep again.
```

No config edits, no daemon, no sudo (except on macOS for lid-close, see below).
The locks are held by a single process and vanish the moment it's killed — or
when your login session ends — so the machine can never get stuck awake by
accident.

## Why not just `systemd-inhibit`? (Linux)

Because **two independent layers** can put your machine to sleep, and the
obvious fix only blocks one of them:

1. **systemd-logind's lid handling** (`HandleLidSwitch=suspend`). A logind
   *block* inhibitor on `handle-lid-switch` stops this. So far so good.
2. **GNOME's idle auto-suspend** (gsd-power's *"Suspend when inactive"*).
   This calls logind's `Suspend()` over D-Bus **as your own uid** — and logind
   deliberately ignores block `sleep` inhibitors owned by the user requesting
   the suspend. So `systemd-inhibit --what=sleep --mode=block` does **nothing**
   against your own desktop's idle suspend. Your laptop closes its lid, survives
   that, then quietly suspends 15 minutes later anyway.

GNOME's auto-suspend *does* honor GNOME **session** inhibitors, so `no-sleep`
holds both locks at once:

```
systemd-inhibit --what=handle-lid-switch:sleep:idle --mode=block \
  └── gnome-session-inhibit --inhibit suspend:idle --inhibit-only
```

One process tree, both layers covered. `no-sleep on` verifies the inhibitor
actually registered before claiming success, and `no-sleep` (no args) shows
the current state.

On non-GNOME systems `gnome-session-inhibit` is absent; the script falls back
to holding just the logind locks (which is all there is to hold).

## Why not just `caffeinate`? (macOS)

Same story, different split: `caffeinate -i` blocks **idle** sleep but macOS
force-sleeps on **lid close** regardless of assertions. The only way to block
that (without an external display + power) is root-level `pmset disablesleep 1`.

So on macOS, `no-sleep on` holds a `caffeinate -i` lock (process-tied,
self-releasing) and asks sudo once to set `pmset disablesleep 1`; `no-sleep off`
kills the lock and restores `disablesleep 0`. Decline the sudo prompt and you
get an idle-only lock (fine with the lid open). Unlike the caffeinate lock,
`disablesleep` is sticky system state — `no-sleep` (status) warns if it was
ever left on, and `sudo pmset -a disablesleep 0` clears it manually.

> macOS support is fresh — if something misbehaves, please open an issue.

## Install

```sh
curl -fsSL https://raw.githubusercontent.com/Hugo0/no-sleep/main/no-sleep \
  -o ~/.local/bin/no-sleep && chmod +x ~/.local/bin/no-sleep
```

(Or copy the script anywhere on your `PATH` — it's a single POSIX sh file.
On macOS you may prefer `/usr/local/bin` if `~/.local/bin` isn't on your PATH.)

## Usage

```
no-sleep on       # hold the locks
no-sleep off      # release them
no-sleep          # show current state
```

## Requirements

- **Linux:** systemd-logind; GNOME for the idle-auto-suspend layer (optional —
  degrades gracefully); polkit must allow your active session to take a
  `handle-lid-switch` inhibitor (`org.freedesktop.login1.inhibit-handle-lid-switch`,
  `implicit active: yes` on stock Ubuntu/Fedora/Arch).
- **macOS:** nothing extra (`caffeinate` and `pmset` ship with the OS); sudo
  only if you want lid-close blocking.

## Caveats

- Blocks *sleep*, not battery drain: a closed laptop in a bag builds heat
  and keeps eating battery. Lock the screen before closing the lid.
- A **critical-battery** hibernate/shutdown intentionally bypasses inhibitors.
  That's a feature.
- Linux: the lock dies with your login session — logging out releases it.
- macOS: if the machine powers off before `no-sleep off`, `disablesleep` may
  still be set; `no-sleep` (status) tells you, and
  `sudo pmset -a disablesleep 0` fixes it.

## License

MIT
