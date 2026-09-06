# Conventional touchpad and mouse scrolling

On this Omarchy/Hyprland system, scroll settings are kept in:

```text
~/.config/hypr/input.lua
```

Open it with an editor, for example:

```bash
nano ~/.config/hypr/input.lua
```

Inside the `input = { ... }` block, use these settings:

```lua
-- Conventional scrolling for an external mouse wheel.
natural_scroll = false,

-- This particular touchpad reports its scroll axis reversed.
-- `true` makes its physical scrolling conventional.
touchpad = {
  natural_scroll = true,
},
```

The two values are intentionally different: the mouse uses the normal direction
with `false`, while this touchpad needs `true` to compensate for its reversed
reported axis.

Hyprland normally reloads Lua configuration files after saving. From a terminal
running inside your desktop session, you can apply and verify the change with:

```bash
hyprctl reload
hyprctl configerrors
```

An empty result from `hyprctl configerrors` means the configuration is valid.
If scrolling becomes inverted again, swap only the affected device's
`natural_scroll` value, save the file, and test it.

---

# Installing Stirling PDF Application:

Search AUR (The Arch User Repository)

and search for package `stirling-pdf-desktop` and install it.

For manual install, run command:
```bash
paru -S stirling-pdf-desktop
# or: yay -S stirling-pdf-desktop
```

---

# 12-hour clock in Omarchy

The desktop clock is configured in `~/.config/omarchy/shell.json`, in the
`omarchy.clock` entry under `bar.layout.center`.

Use this format for a 12-hour clock with AM/PM:

```json
"format": "dddd h:mm AP"
```

`h` is the 12-hour value; `AP` adds uppercase `AM` or `PM`. For a shorter
clock without the weekday, use:

```json
"format": "h:mm AP"
```

The Omarchy shell normally hot-reloads this file after saving. If it does not,
run:

```bash
omarchy restart shell
```

You can also right-click the clock to cycle its built-in display formats; that
choice is saved back to the same configuration file.
