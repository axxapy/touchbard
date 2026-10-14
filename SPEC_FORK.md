# touchbard fork: changes against upstream

**This file describes only what is NOT in the original repository** ([0800tim/touchbard](https://github.com/0800tim/touchbard), base commit `1b29141`). Everything else (socket protocol, presets, plugins, themes, layers, Touch ID, visualiser) works as upstream documents it in `README.md` and the source; read those for the full picture.

## Branches

| Branch                          | Based on                  | Contents                                                 |
| ------------------------------- | ------------------------- | -------------------------------------------------------- |
| `main`                          | `origin/main`             | upstream, untouched                                      |
| `dev/axp`                       | `main`                    | every `feature/*` branch merged (`--no-ff`) + this file  |
| `feature/t2-kbd-backlight`      | `main`                    | `7d28150`                                                |
| `feature/t2-touchbar-backlight` | `main`                    | `b83fbb3`                                                |
| `feature/t2-battery-charge`     | `main`                    | `bc5db4b`                                                |
| `feature/workspaces-active`     | `main`                    | `6eed8cc`                                                |
| `feature/button-commands`       | `main`                    | `715868a`                                                |
| `feature/button-rec`            | `feature/button-commands` | `0406117` (the indicator sits on the per-state commands) |
| `feature/agent-sh-commands`     | `main`                    | `510dc0d`                                                |

New work: branch `feature/<name>` off `main` (or off the feature it needs), then merge it into `dev/axp` with `git merge --no-ff`. Install from `dev/axp`. Upstream PRs are made from the `feature/*` branches.

## Summary

| Commit    | Area             | Change                                                                                  | Upstream status                     |
| --------- | ---------------- | --------------------------------------------------------------------------------------- | ----------------------------------- |
| `7d28150` | agent            | Keyboard backlight found on Intel T2 Macs (`apple::kbd_backlight`)                      | offered in PR #1                    |
| `b83fbb3` | daemon / hw      | Touch Bar backlight scaled to the device range (Intel T2: 0..=2)                        | offered in PR #1                    |
| `bc5db4b` | daemon / render  | Battery percentage = charge now ÷ full                                                  | offered in PR #1                    |
| `6eed8cc` | daemon           | Workspaces item: `format = "active"`                                                    | fork only                           |
| `715868a` | daemon + agent   | `commands`: per-state tap and long-press commands                                       | fork only                           |
| `0406117` | daemon           | Recording indicator (`rec`) and toggle width (`w_on`)                                   | fork only                           |
| `510dc0d` | agent            | Button commands run with `sh -c` instead of `bash -lc`                                  | fork only                           |

## Intel T2 fixes

- **Keyboard backlight:** the agent looks for `/sys/class/leds/kbd_backlight` (Apple silicon) and then `/sys/class/leds/apple::kbd_backlight` (Intel T2, `apple-bce`/`t2bce`). The found name is used both for reading the level and for `brightnessctl -d <name>`.
- **Touch Bar backlight:** levels stay on a 0..=255 scale in config and state and are scaled to the device maximum with rounding up (`v * max / 255`, ceil). Intel T2 `appletb_backlight` only has 0..=2 (off, dim, full); rounding up keeps any non-zero level visible and dimmed distinct from full. Upstream clamps instead.
- **Battery:** the percentage is `charge_now / charge_full` (or `energy_now / energy_full`), capped at 100, falling back to `capacity`. The ACPI SBS gauge on Intel T2 Macs reports `capacity` against the design capacity, so a worn battery never showed more than its health.

## Workspaces: `format = "active"`

`{ preset = "workspaces", format = "active" }` shows only the active workspace (one cell) instead of all of them. Tapping it runs the item's command with `{id}` = that workspace.

## Button config

### `commands`

```toml
{ icon = "󰄀", toggle = "recording", commands = [
    { state = "recording", command = "screenrecord stop" },
    { hold_ms = 500, command = "capture-menu" },
  ], command = "screenshot region" }
```

| Field     | Type            | Required | Meaning                                                                 |
| --------- | --------------- | -------- | ----------------------------------------------------------------------- |
| `command` | string          | yes      | Shell command, run by the agent with `sh -c`                            |
| `state`   | string          | no       | A state key that must be true (JSON truthy boolean) for this entry      |
| `hold_ms` | integer (ms)    | no       | Makes the entry a long press of this length; without it, it is a tap    |

Rules:

1. Taps (entries without `hold_ms`) and long presses (entries with `hold_ms`) are matched separately.
2. The state is read **once, at touch-down**; a press never changes meaning while the finger is down.
3. Within each group, the **first entry in array order whose `state` key is true** wins. An entry without `state` is used **only when no conditional entry matched**; among those, the first one wins.
4. No match: the tap does nothing (or, for a tap, the item's non-command action runs if it has one: `keys`, `layer`, `toggle_layer`, or a preset's built-in action).
5. Only the first matching long-press entry counts; several hold lengths for the same state do not stack.
6. A long press **fires as soon as its time is reached**, without waiting for release; the release then does nothing. Released exactly at the threshold before the daemon ticked: it runs on release. From the moment the hold counts, the button shows a steady red dot.
7. Sliding more than 24 px off the button cancels the press: nothing runs.

`command = "x"` is kept for compatibility and is the same as one more entry **at the end**: `commands + [{ command = "x" }]`. The agent does this conversion, so in the layout sent to the daemon a user `command` appears as an entry of `commands`, not as `act = { cmd }` (presets that define `act` directly are unchanged). Upstream's `command` is otherwise unaffected: a plain `command` with no `commands` behaves exactly as before.

The agent rejects a `commands` value that is not an array of tables, and any entry without a string `command`.

### Recording indicator: `rec`

`rec = true` on a button with `toggle = "<key>"`:

| State                                   | Drawn instead of the icon                                              |
| --------------------------------------- | ---------------------------------------------------------------------- |
| `<key>` true                            | A red dot blinking every 0.5 s and the time elapsed since `<key>_since` (unix seconds, float), `m:ss` |
| `<key>` false and `<key>_pending` true  | A steady red dot (e.g. while a recording region is being selected)     |
| otherwise                               | The normal icon / label (`icon_on` / `label_on` rules as upstream)     |

While any `rec` indicator is on, the daemon redraws on every half second.

### `w_on`

Width in pixels used while the item's `toggle` key is true (for `rec` items also while `<key>_pending` is true), e.g. to make room for the recording timer. Falls back to `w`.

## Layout JSON (daemon side)

New `Item` fields, as the agent sends them in `{"t": "layout", …}`:

| Field      | Type                                              | Default |
| ---------- | ------------------------------------------------- | ------- |
| `commands` | `[{ "state"?: string, "hold_ms"?: u64, "command": string }]` | `[]`    |
| `rec`      | bool                                              | `false` |
| `w_on`     | number                                            | none    |

No message types were added or changed.

## State keys used by this setup

Not part of the daemon, a convention between the user's scripts and the config: `recording`, `recording_pending` and `recording_since` are written by the screen-recording script with `{"t": "state", …}` on the socket and drive the camera button above. Any other key works the same way.

## Agent

- Button commands run with `sh -c "<command>"` (detached, own session, no stdin, output discarded) instead of `bash -lc`. A login shell re-read the whole profile on every tap (≈ 40 ms against ≈ 9 ms). Commands no longer see variables set only in login profiles; the session environment (Hyprland, systemd user env) still applies.

## Tests

`cargo test` in `daemon/` runs the unit tests for the `commands` matching rules (`ui::tests`). Upstream has no tests.
