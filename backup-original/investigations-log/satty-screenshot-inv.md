# Satty — Interactive Screenshot Annotation Shortcut (Print Screen)

A guide for setting up **satty** (modern screenshot/annotation tool) so that pressing
**Print Screen** captures a region and opens satty for annotation.

Tested on: **Fedora 44 Workstation, GNOME 50, Wayland session**.

---

## Table of Contents

1. [What this sets up](#what-this-sets-up)
2. [Install satty](#1-install-satty)
3. [Verify satty runs](#2-verify-satty-runs)
4. [Understand the landscape: satty needs a screenshot source](#3-understand-the-landscape-satty-needs-a-screenshot-source)
5. [Why gnome-screenshot fails on modern GNOME Wayland](#4-why-gnome-screenshot-fails-on-modern-gnome-wayland)
6. [The working solution: XDG Desktop Portal capture script](#5-the-working-solution-xdg-desktop-portal-capture-script)
7. [Register the Print Screen shortcut](#6-register-the-print-screen-shortcut)
8. [Troubleshooting](#7-troubleshooting)

---

## What this sets up

- Press **Print Screen** → GNOME region-selection UI appears
- Drag to select an area → screenshot saved to `~/Pictures/Screenshots/`
- **satty** opens fullscreen with the captured image for annotation
- Annotate, then copy / save as usual (e.g. `Save as` → pick location)

---

## 1. Install satty

Download from [Flathub](https://flathub.org/apps/org.satty.Satty) or via CLI:

```bash
flatpak install flathub org.satty.Satty
```

---

## 2. Verify satty runs

```bash
flatpak run org.satty.Satty --help
```

Shortcut-invokable binary:

```
flatpak run org.satty.Satty --filename <image> --fullscreen
```

satty reads an existing image file (or `-` from stdin) — it does **not** take
screenshots itself. It always needs an external capture step.

### satty config location (startup notes)

```
~/.var/app/org.satty.Satty/config/satty/
```

Missing config/CSS files are non-fatal; satty falls back to built-in defaults
(`config file not found`, `overrides.css does not exist` — both harmless).

---

## 3. Understand the landscape: satty needs a screenshot source

| Tool                | Works on GNOME Wayland?          | Notes                                        |
|---------------------|-----------------------------------|----------------------------------------------|
| `grim` / `slurp`    | No (wlroots-only)                 | Mutter doesn't support wlr-screencopy        |
| `gnome-screenshot`  | Broken on GNOME 45+ (see below)   | Last version 41.x, uses dead Shell D-Bus API |
| ImageMagick `import`| No (X clients only)               | Captures Xwayland, not the whole desktop     |
| **XDG Desktop Portal** (`org.freedesktop.portal.Screenshot`) | **Yes** | Same path GNOME's own UI uses; supported     |

The correct modern approach on GNOME is the **XDG Desktop Portal**, which is
installed and running by default on Fedora:

```bash
rpm -q xdg-desktop-portal xdg-desktop-portal-gnome    # expect both installed
ps -eo comm | grep portal                              # expect xdg-desktop-portal-gnome
```

---

## 4. Why gnome-screenshot fails on modern GNOME Wayland

If you try `gnome-screenshot -a -f /tmp/satty.png && flatpak run ...`, **nothing
happens** — no crosshair, no satty, no file. The journal reveals why:

```bash
journalctl --user -b | grep -i screenshot
```

```
gnome-screenshot[16120]: Unable to select area using GNOME Shell's builtin screenshot
                         interface, resorting to fallback X11.
gnome-screenshot[16120]: gdk_pixbuf_get_from_surface: assertion 'width > 0 && height > 0' failed
gnome-screenshot[16120]: Unable to capture a screenshot of any window
```

Calling the old GNOME Shell D-Bus interface directly returns:

```
GDBus.Error:org.freedesktop.DBus.Error.AccessDenied: Screenshot is not allowed
```

**Root cause:** `gnome-screenshot` is unmaintained (last released 2021, v41.x).
It relies on `org.gnome.Shell.Screenshot`, which modern mutter/portal builds deny,
and its X11 fallback cannot capture a Wayland desktop. It exits silently (code 0)
with no output file, which is why the shortcut "does nothing".

---

## 5. The working solution: XDG Desktop Portal capture script

The portal (`org.freedesktop.portal.Screenshot` with `interactive: true`) shows the
native GNOME selection UI and returns the captured file's URI via a D-Bus `Response`
signal. A tiny Python script handles that handshake and then launches satty.

### Create the script

Save as `~/.local/bin/satty-screenshot` and make it executable:

```bash
mkdir -p ~/.local/bin
```

**Requirement:** `python3-gi` (present in Fedora's default GNOME install;
check with `python3 -c "import gi"`).

```python
#!/usr/bin/env python3
import os
import sys
import subprocess
from urllib.parse import unquote
import gi

gi.require_version('Gio', '2.0')
from gi.repository import Gio, GLib


def main():
    bus = Gio.bus_get_sync(Gio.BusType.SESSION, None)
    try:
        args = GLib.Variant.parse(
            GLib.VariantType.new("(sa{sv})"),
            '("", {"interactive": <true>, "modal": <true>})',
            None, None)
        reply = bus.call_sync(
            "org.freedesktop.portal.Desktop",
            "/org/freedesktop/portal/desktop",
            "org.freedesktop.portal.Screenshot",
            "Screenshot",
            args,
            None, Gio.DBusCallFlags.NONE, 10000, None)
    except GLib.Error as e:
        print("portal error:", e.message, file=sys.stderr)
        return 1
    req = reply.unpack()[0] if isinstance(reply.unpack(), (tuple, list)) else reply.unpack()

    loop = GLib.MainLoop()
    outcome = {}

    def on_response(_c, _s, path, _i, _sig, params, _d):
        if path != req:
            return
        code = params[0]
        results = params[1] if len(params) > 1 else {}
        outcome['code'] = code.unpack() if hasattr(code, 'unpack') else code
        outcome['results'] = {}
        if isinstance(results, dict):
            outcome['results'] = results
        else:
            for k in results:
                outcome['results'][k] = results[k].unpack() if hasattr(results[k], 'unpack') else results[k]
        loop.quit()

    bus.signal_subscribe(
        "org.freedesktop.portal.Desktop",
        "org.freedesktop.portal.Request",
        "Response", req, None,
        Gio.DBusSignalFlags.NONE, on_response, None)

    GLib.timeout_add_seconds(300, lambda: (outcome.setdefault('timeout', True), loop.quit()))
    loop.run()

    uri = None
    if outcome.get('code') == 0:
        uri = outcome['results'].get('uri')
        if uri is None:
            return 1
        fpath = unquote(uri.replace("file://", ""))
        print("captured:", fpath, file=sys.stderr)
        if not os.path.exists(fpath):
            print("capture file missing:", fpath, file=sys.stderr)
            return 1
        try:
            rc = subprocess.run(
                ["flatpak", "run", "org.satty.Satty",
                 "--filename", fpath, "--fullscreen"],
                cwd="/").returncode
            print("satty rc:", rc, file=sys.stderr)
        except OSError as e:
            print("failed to launch satty:", e, file=sys.stderr)
            return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

```bash
chmod +x ~/.local/bin/satty-screenshot
```

### How it works

1. Asynchronously calls the portal's `Screenshot` method with `interactive: true` —
   this pops the native GNOME selection crosshair.
2. Subscribes to the `org.freedesktop.portal.Request.Response` signal on the
   portal's returned request object for the result.
3. On success, decodes the returned `file://` URI (URL-encoded spaces via `%20` —
   a common gotcha) and launches satty fullscreen on that file.
4. Excess time: request is cancelled cleanly on a 5-minute timeout.

### Test it manually

```bash
/home/shams/.local/bin/satty-screenshot
```

The selection UI appears → draw a box → satty opens with the capture.
Expected log lines (all good):

```
captured: /home/shams/Pictures/Screenshots/Screenshot From 2026-09-12 22-15-15.png
Fullscreen Some(CurrentScreen) | Resize None | Floatinghack false
```

`VK_SUBOPTIMAL_KHR` warnings from satty are harmless Vulkan swapchain notices.

---

## 6. Register the Print Screen shortcut

### GUI method

1. **Settings → Keyboard → View and Customize Shortcuts → Custom Shortcuts**
2. Click **+** (Add Shortcut)
   - Name: `ScreenshotAnnotate` (anything)
   - Command: `/home/shams/.local/bin/satty-screenshot`
3. Click **Set Shortcut** and press **Print Screen**
4. Toggle it on.

### CLI method (gsettings)

Find your custom shortcut slot:

```bash
gsettings get org.gnome.settings-daemon.plugins.media-keys custom-keybindings
```

Set the command and binding for a slot (here `custom4`):

```bash
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom4/ name 'ScreenshotAnnotate'
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom4/ command '/home/shams/.local/bin/satty-screenshot'
gsettings set org.gnome.settings-daemon.plugins.media-keys.custom-keybinding:/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom4/ binding 'Print'
```

Use the absolute path in the command — the shortcut's shell may not have
`~/.local/bin` on `PATH`.

---

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Pressing Print does nothing | Custom shortcut isn't firing, or command is the broken `gnome-screenshot` chain | Set command to the absolute script path; verify with `journalctl --user -b | grep -i screenshot` |
| `Unable to select area using GNOME Shell's builtin screenshot interface, resorting to fallback X11` | `gnome-screenshot` v41 can't screenshot on modern Wayland | Use the portal script (Section 5) |
| `Screenshot is not allowed` (D-Bus) | Old `org.gnome.Shell.Screenshot` API denied | Use the XDG Desktop Portal instead |
| `Error: couldn't load image ... No such file or directory` | URI passed raw with `%20` escapes | `urllib.parse.unquote()` the path (handled in the script) |
| `PORTAL CALL FAILED` | `python3-gi` missing / session bus issue | `sudo dnf install python3-gi`; run from the graphical session |
| selection UI never appears | Script not run from the desktop session | Test with `/home/shams/.local/bin/satty-screenshot` in a terminal |

---

## Key files

| Path | Purpose |
|------|---------|
| `/home/shams/.local/bin/satty-screenshot` | The capture → satty launcher script |
| `~/Pictures/Screenshots/` | Where the GNOME portal saves captures |
| `~/.var/app/org.satty.Satty/config/satty/` | satty flatpak config directory |

---

## Reference: satty CLI options (v0.22.0)

```bash
flatpak run org.satty.Satty --help
```

```
-f, --filename <FILENAME>        Path to input image or '-' to read from stdin
--fullscreen [<MODE>]            Start satty fullscreen [possible values: all, current-screen]
--resize [<MODE|WxH>]            Resize to coordinates or smart mode
-o, --output-filename            Filename for save action ('-' = stdout)
--early-exit [<TRIGGER>...]      Exit after save/copy action
--initial-tool <TOOL>            pointer, crop, line, arrow, rectangle, ellipse, text, marker, blur, highlight, brush
--auto-copy                      Copy to clipboard after every annotation change
--actions-on-enter <ACTIONS>     save-to-clipboard, save-to-file, save-to-file-as, copy-filepath-to-clipboard, exit
--actions-on-escape <ACTIONS>    same actions as above
--copy-command <CMD>             Custom copy command (e.g. wl-copy)
-h, --help                       Print help
```