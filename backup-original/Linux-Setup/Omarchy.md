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


---

# Documenting the balenaEtcher Installation Challenge

**Date:** 2026-09-12
**System:** Omarchy (Arch Linux based, `pacman` + AUR via `yay`)
**Goal:** Install a software to create a bootable USB drive from an ISO file.

---

## 1. The Task

The user asked to install software to create a bootable USB drive from an ISO file.
After discussing the options, the user selected **balenaEtcher**.

## 2. The Initial Challenge: Installing from the AUR

balenaEtcher is not available in the official Arch Linux repositories. The only AUR
package found was `balena-etcher` (`yay -Ss balena-etcher` → `aur/balena-etcher 2.1.6-1`).

The AUR `balena-etcher` package builds **everything from source**, and crucially it
drags in an AUR dependency, `electron37`, which compiles a large portion of **Google
Chromium** from scratch. This surfaced two problems:

1. **Runtime**: The build timed out after 10 minutes while cloning the
   `chromium-mirror` repo (`fetch-pack: unexpected disconnect while reading sideband
   packet`), and `yay` aborted with `TERM signal caught`.
2. **Practicality**: Even if the clone finished, compiling Electron/Chromium from
   source typically takes **hours** to tens of hours and requires a huge amount of
   memory/disk. This is essentially never the right path unless you are packaging a
   distribution.

## 3. Thought Process to Solve the Problem

The key insight was to **question the packaging choice rather than the software**.
The source build was a red herring caused by the AUR packaging strategy, not by the
application itself.

Steps considered, in order:

1. **Check for a prebuilt AUR variant first.**
   - Searched for `balena-etcher-bin` and `etcher-bin` (the `-bin` suffix on AUR
     packages means "ships prebuilt binaries"). Result: not found — no quicker AUR
     route exists for this app.

2. **Reconsider the distribution channel.**
   - balenaEtcher is an Electron desktop app. The official GitHub releases for
     Etcher ship prebuilt Linux binaries. So the packaging strategy could be bypassed
     entirely by installing from the vendor's official release instead of the AUR
     source build.

3. **Fetch the official release and inspect what's available.**
   - Queried the GitHub API for the latest release:
     `https://api.github.com/repos/balena-io/etcher/releases/latest`
   - Found `balenaEtcher-linux-x64-2.1.6.zip` — a ready-to-run, prebuilt Linux x64
     build. No compilation required.

4. **Install it in user space (no root needed).**
   - Downloaded the zip (~163 MB) to `/tmp/opencode`.
   - Extracted it and found a self-contained Electron app directory containing the
     `balena-etcher` launcher binary.
   - Copied it to `~/.local/opt/balena-etcher/` (a conventional user-space install
     location) and symlinked the binary into `~/.local/bin/balena-etcher` so it is on
     `$PATH`.

5. **Recreate the desktop-integration pieces that `makepkg` normally handles.**
   - Downloaded the official app icon (`512x512.png`) to
     `~/.local/share/icons/hicolor/512x512/apps/balena-etcher.png`.
   - Created `~/.local/share/applications/balena-etcher.desktop` based on the AUR
     package's desktop entry, so the app appears in the launcher menu.

6. **Verify it runs.**
   - Ran `timeout 15 balena-etcher --version`. The app started cleanly (exit 0; only
     unrelated fontconfig warnings), confirming the sandbox/user-namespace setup works
     without the `.chromium-sandbox` helper.
   - Cleaned up the temporary download/extraction files.

## 4. Outcome

balenaEtcher v2.1.6 is installed and functional:

- Binary: `balena-etcher` (via `~/.local/bin/balena-etcher`)
- App menu entry: "BalenaEtcher"
- Usage: insert USB drive → open BalenaEtcher → select the ISO → **Flash!**

Alternatively, had a GUI/flash tool not been required, `dd` (default on Arch) could
write an ISO to a USB drive with a single command and requires no installation at all.

## 5. Lesson Learned

When an AUR package builds from source, always check the application's official
binaries first. For large apps (Electron, browsers, IDEs), a source build in the AUR
is usually impractical, and a prebuilt binary from the vendor (or a `-bin` AUR
package) is the correct, far faster path.