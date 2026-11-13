# niri-ds

Touchscreen gesture support for [niri](https://github.com/niri-wm/niri) on the
AYN Thor: multi-finger gestures across its two internal panels, so a niri
session is usable with the keyboard detached. The on-screen keyboard is not
part of this patchset — it lives in
[plasma-keyboard-custom](https://github.com/theodotdot/plasma-keyboard-custom/tree/feature/niri-backend).


[![License: GPL-3.0-only](https://img.shields.io/badge/license-GPL--3.0--only-blue)](LICENSE)
[![niri base](https://img.shields.io/badge/niri-v26.04%20%2B%2020%20patches-blueviolet)](#status--credits)

## Gestures

The table is the shipped default: `config/gestures.kdl` is the config this
scheme is developed and daily-driven on, so copying it in gives you exactly
what's below. Nothing is locked in: every node in the `gestures` block takes
any niri action, same as keybinds. Change the file as you see fit.

| Gesture | Action (as shipped) |
|---|---|
| **3-finger swipes** | |
| horizontal drag | resize the focused window's width, live, tracking your fingers 1:1 (set `horizontal-swipe "move-view"` to scroll the view instead) |
| vertical drag | workspace carousel (animated) |
| tap | maximize / restore the focused column |
| **3-finger hold + swipe** | |
| rest ~400ms, then swipe left/right | move the focused window one column left/right |
| rest ~400ms, then swipe up/down | move the focused window one workspace up/down |
| **4-finger swipes** (one finger more than `fingers`; 4 by default) | |
| tap | window overview (niri's own, works with zero setup); prefer an app launcher? Noctalia, fuzzel, wofi and rofi lines sit commented in the config, ready to swap in |
| flick down | close the focused window |
| flick up | toggle the focused window fullscreen |
| flick left / flick right | unbound by default: they fall back to the horizontal drag (resize/move-view); examples commented in the config |
| **Edge swipes** | |
| one finger, swiping up from the bottom edge | unbound: bind your on-screen keyboard's `Exec` here |
| one finger, swiping down from the top edge | toggle the overview |
| one finger, swiping inward from the left edge | focus the column to the left |
| one finger, swiping inward from the right edge | focus the column to the right |
| **Corner swipes** | |
| one finger, swiping diagonally from a corner | unbound on purpose: pick your own (examples commented in the config) |

Two footnotes. With no `gestures {}` config at all, only the two animated
swipes and the 3-finger tap do anything (the compiled defaults); everything
else above comes from the config file, which is the point of shipping it.
And touchpad behavior is untouched.

Prefer picking actions over editing KDL? `niri-tablet-easysetup/` in the
repo is a small GTK app that does exactly that: it loads your current
scheme, offers installed apps and niri actions for every gesture, fires
test runs through `niri msg`, and saves only what `niri validate`
accepts (the running compositor confirms the reload). On Arch,
`update-niri-ds.sh` builds it for you, first install included, and
adds it to your app launcher (niri logo icon); elsewhere,
`cargo build --release` inside the directory.

### Hold-swipe: move windows

Rest three (or more) fingers still for a moment (~400ms), then swipe: instead
of the animated view scroll, the dominant direction runs a discrete action.
The example config binds it to window management: held swipe left/right
moves the focused column (`move-column-left`/`right`, like Ctrl+Mod+arrows),
held swipe up/down moves the focused window to the workspace above/below
(`move-window-to-workspace-up`/`down`). Quick swipes are unaffected: how long
your fingers lingered below the movement threshold decides. Remove the `hold`
block (or a single direction) and held swipes behave like quick ones again.

### Gesture ownership

From the moment `touchscreen-swipe.fingers` fingers are down, the whole touch
sequence belongs to the compositor: apps never see it (touches they already
received are cancelled), and normal delivery resumes when the last finger
lifts. So a 3-finger swipe doesn't pinch-zoom a browser or drag-select
terminal text mid-gesture. Single- and two-finger input, including app
pinch-zoom, is untouched. If an app genuinely needs 3-finger touches, raise
`fingers`.

### Touch point visualization

`show-touch-points` renders translucent dots that follow your fingers,
drawn by the compositor itself, so any screen recording captures them and
input is never touched. Meant for demo videos and for debugging gestures:

```kdl
gestures {
    show-touch-points "gestures"   // dots only while a multi-finger gesture is down
    // show-touch-points "all"     // every finger (debugging)
}
```

Off by default. In `gestures` mode dots appear only while enough fingers
are down concurrently for a gesture (`touchscreen-swipe.fingers`), so
ordinary taps and scrolling stay clean.

### Gesture debug logging

`debug-log` turns on per-event logging of the touchscreen gesture stack:
every finger down/up/motion with slot ids, positions and timestamps, the
recognition decision (finger count, direction, hold vs quick), and the
action dispatched for taps, discrete swipes and edge/corner swipes. Every line is
prefixed `gesture-debug:`.

```kdl
gestures {
    debug-log
}
```

Made for bug reports: enable it, reproduce the misbehaving gesture, then
collect the lines. Under systemd sessions:

```bash
journalctl --user -u niri.service --since -10m | grep gesture-debug
```

Running niri from a terminal instead? The lines go to its stderr. The
option can be toggled live: niri reloads the config on save and the flag
applies from the next touch. Off by default.

## How it works

niri has no native touchscreen gestures (see the upstream
[discussion](https://github.com/niri-wm/niri/discussions/463)). This repo
maintains a small patch series on top of a current niri release:

- **`pkg/`** - the Arch PKGBUILD plus the 20 patches (`git am`-able, authorship
  preserved) sitting next to it, as makepkg requires: animated multi-finger
  swipes reusing niri's touchpad gesture pipeline, taps and discrete
  flicks at one finger more than the base count (3 or 4), hold-swipes,
  gesture ownership (multi-finger touches are cancelled
  client-side so apps don't react to them), single-finger edge and corner
  swipes (8 zones, any action), a horizontal-swipe mode that resizes the
  focused window's width live (on by default in the shipped config), touch
  point visualization, clean recovery when a touch device disappears
  mid-gesture (hotplug/unplug no longer wedges the session), and opt-in
  gesture event logging for bug reports. The animated
  swipes run through the same spring-physics
  pipeline as touchpad gestures: rotation-proof logical coordinates, live
  follow, inertia.
- **`config/gestures.kdl`** - the gesture config block (see above).

The base swipe implementation comes from
[niri-touch-gestures](https://github.com/pir0c0pter0/niri-touch-gestures) by
Mario St Jr (author of the Noctalia shell); this repo extends it with taps,
discrete flicks, finger-count and deadlock fixes, and packaging.

Upstream is working on [configurable gesture binds](https://github.com/niri-wm/niri/pull/3771);
when that lands, this patchset can be retired.

## Install (Arch Linux)

The easiest path is the bundled updater (it handles first installs too):

```bash
git clone https://github.com/theodotdot/niri-ds.git
cd niri-ds
./update-niri-ds.sh
```

It fetches the latest release, builds it (the `sudo` prompt for the
install comes from `makepkg`), pins `IgnorePkg = niri` in
`/etc/pacman.conf` if that pin is missing, and sets up the EasySetup GUI
(build, launcher entry and icon; no sudo). Manual equivalent:

```bash
cd niri-ds/pkg
makepkg -si        # builds upstream niri + patches; replaces stock niri
```

Then protect it from repo updates in `/etc/pacman.conf`:

```ini
IgnorePkg = niri
```

pacman will ask to remove stock niri (`Remove niri? [y/N]`, default N):
answer **y**; with N the install simply aborts. `niri-ds` provides
and conflicts with `niri`, so the swap itself is automatic.
Needs `rust`, `cargo`, `clang` and `git` to build. First build takes a while;
the PKGBUILD shares a cargo target dir (`~/.cache/niri-tablet-target`) so
rebuilds are incremental.

After a re-login, `niri --version` prints `26.04 (v26.04-modified)`: the
patchset applies uncommitted on top of the release tag, so git-describe
reports that rather than the pkgver's `26.04.20…`. The positive check that
the patched binary is running: `niri validate` accepts a `gestures {}`
block; stock niri rejects the unknown node.

## Updating

One command, from anywhere inside the clone:

```bash
./update-niri-ds.sh
```

It fetches the latest release tag, shows the changelog since your installed
version, builds and installs the package, and re-checks the
`IgnorePkg = niri` pin. After v26.04.20 it also checks your config for the
renamed gesture nodes and offers to migrate them (dated backups; undone
unless `niri validate` passes). The GUI configurator is rebuilt too,
whenever its sources changed (a plain cargo build, no sudo), and its
launcher entry and icon are kept installed. The updater refreshes
itself from origin/main at startup, so fixes to it don't wait for a
release tag.
Useful flags: `--check` reports without
touching anything, `--force` rebuilds an up-to-date install,
`--tag vX.Y.Z` picks a specific release, `--main` tracks the development
branch, `--yes` skips prompts for non-interactive runs. The manual equivalent:

```bash
git pull
cd pkg
makepkg -si        # incremental: the shared target dir is reused
```

`update-niri-ds.sh` at the repo root is the user-facing updater;
`update.sh` and `install.sh` are maintainer scripts (they rebase and
regenerate the patch series against a dev clone of niri). After a big Rust
toolchain jump, deleting `~/.cache/niri-tablet-target` is harmless: the
next build is just a slow cold one again.

## On-screen keyboard

Not part of this patchset. Use
[plasma-keyboard-custom](https://github.com/theodotdot/plasma-keyboard-custom/tree/feature/niri-backend)
— the `feature/niri-backend` branch adds `zwp_input_method_v2`, which niri
implements — and bind its `Exec` to a gesture or a key.

## Repo layout

```
pkg/       Arch PKGBUILD + the patch series against upstream niri
update-niri-ds.sh   user-facing updater (install/update/pin)
config/    example niri config fragments
niri-tablet-easysetup/   GUI gesture configurator (GTK4)
test.kdl   minimal config for nested (in-window) testing
niri/      maintainer's dev clone for rebasing (not part of the repo)
```

## Tested hardware

- AYN Thor (SM8550, DSI-1 + DSI-2 panels, Pocknix)

Nothing else is supported: the patchset is developed against this device only.

## Status & credits

- Patchset: `v26.04 + 20 patches`, unit-tested (full suite runs in CI).
- One design note for anyone hacking on the gesture code: never run a niri
  action from inside a smithay touch-grab callback (seat touch mutex
  deadlock); actions are deferred via `Niri::pending_touch_action`.

License: **GPL-3.0-only** (the patches modify niri, also GPL-3.0-only).
Swipe foundation by Mario St Jr; taps, flicks, fixes, scripts and packaging
by [GGEZUS](https://github.com/GGEZUS). EasySetup's app icon is
niri's logo, used under
[CC BY-SA 4.0](https://github.com/niri-wm/niri/wiki/Name-and-Logo).
