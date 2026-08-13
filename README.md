# no-sleep

Keep a GNOME/systemd laptop awake — lid closed, idle, on battery — until you
toggle it off.

```
$ no-sleep on
☕ no-sleep ON  — close the lid and keep working.
   • blocks lid-close suspend (logind) + GNOME idle auto-suspend
   • run 'no-sleep off' when you're done
   • heat builds in a closed bag; Super+L locks the screen first

$ no-sleep off
🌙 no-sleep OFF — normal: lid-close and idle suspend again.
```

No sudo. No config edits. No daemon. The locks are held by a single process
tree and vanish the moment it's killed — or when your login session ends — so
the machine can never get stuck awake by accident.

## Why not just `systemd-inhibit`?

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

## Install

```sh
curl -fsSL https://raw.githubusercontent.com/Hugo0/no-sleep/main/no-sleep \
  -o ~/.local/bin/no-sleep && chmod +x ~/.local/bin/no-sleep
```

(Or just copy the script anywhere on your `PATH` — it's a single POSIX sh file.)

## Usage

```
no-sleep on       # hold the locks
no-sleep off      # release them
no-sleep          # show current state
```

## Requirements

- systemd-logind (any systemd distro)
- GNOME for the idle-auto-suspend layer (optional — degrades gracefully)
- polkit must allow your active session to take a `handle-lid-switch`
  inhibitor (`org.freedesktop.login1.inhibit-handle-lid-switch`,
  `implicit active: yes` on stock Ubuntu/Fedora/Arch)

## Caveats

- Blocks *suspend*, not battery drain: a closed laptop in a bag builds heat
  and keeps eating battery. Lock the screen (Super+L) before closing the lid.
- A **critical-battery** hibernate/shutdown (upower) intentionally bypasses
  inhibitors. That's a feature.
- The lock dies with your login session — logging out releases it.

## License

MIT
