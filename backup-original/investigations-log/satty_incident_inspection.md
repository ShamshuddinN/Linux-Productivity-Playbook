# Incident Inspection — `satty-screenshot` fails with `ModuleNotFoundError: No module named 'gi'`

**Date:** 2026-10-04
**Host:** Fedora Linux 44 (Workstation), GNOME 50, Wayland
**User:** `shams` (uid 1000)
**Affected artifact:** `/home/shams/.local/bin/satty-screenshot`
**Status:** ✅ **Resolved** — verified fixed
**Severity:** High (the documented Print Screen → satty annotation workflow was fully broken)

---

## Table of Contents

1. [Reported symptom](#1-reported-symptom)
2. [Root cause](#2-root-cause)
3. [Evidence](#3-evidence)
4. [The fix](#4-the-fix)
5. [Verification](#5-verification)
6. [Why it happened (contributing factors)](#6-why-it-happened-contributing-factors)
7. [Prevention / hardening](#7-prevention--hardening)
8. [Rollback](#8-rollback)
9. [Files touched](#9-files-touched)
10. [Related documents](#10-related-documents)

---

## 1. Reported symptom

Running the shortcut helper directly produced an immediate, total failure — the
traceback occurs on line 6, before any portal logic runs, so no screenshot UI
ever appears:

```console
$ /home/shams/.local/bin/satty-screenshot
Traceback (most recent call last):
  File "/home/shams/.local/bin/satty-screenshot", line 6, in <module>
    import gi
ModuleNotFoundError: No module named 'gi'
```

The `gi` module is **PyGObject**, the Python binding to GLib/GObject. The script
depends on it for `Gio` (D-Bus) and `GLib` (main loop / `GLib.Variant`), which it
uses to call the XDG Desktop Portal `Screenshot` method and then launch satty.

Exit status is `1`; nothing is written to `~/Pictures/Screenshots/`.

---

## 2. Root cause

**A shebang/interpreter mismatch: `#!/usr/bin/env python3` resolved to
Homebrew's Python, which cannot see the distro-packaged PyGObject.**

Two independent Python 3.14 installations exist on this machine:

| Interpreter | Path | Has `gi`? |
|-------------|------|-----------|
| Homebrew (Linuxbrew) | `/home/linuxbrew/.linuxbrew/bin/python3` → `/home/linuxbrew/.linuxbrew/opt/python@3.14/bin/python3.14` | ❌ **No** |
| Fedora system | `/usr/bin/python3` | ✅ Yes |

The script began with the *relative* shebang:

```python
#!/usr/bin/env python3
```

`env` resolves `python3` by searching `$PATH`. On this host, Homebrew's bin
directory is **prepended** to `PATH` (positions 1 and 4, ahead of `/usr/bin` at
position 11):

```console
$ echo "$PATH" | tr ':' '\n' | grep -n -E 'linuxbrew|/usr/bin'
1:/home/linuxbrew/.linuxbrew/bin
2:/home/linuxbrew/.linuxbrew/sbin
4:/home/linuxbrew/.linuxbrew/bin
5:/home/linuxbrew/.linuxbrew/sbin
10:/usr/local/bin
11:/usr/bin
```

So the kernel executed the script with **Homebrew Python 3.14.8**.

PyGObject was installed only by the operating system, as an RPM, into Fedora's
system site-packages:

```
/usr/lib64/python3.14/site-packages/gi/__init__.py
```

Homebrew's interpreter is built against its **own** Cellar prefix and therefore
has a completely disjoint `sys.path` — it never looks in `/usr/lib64/...`:

```console
$ python3 -c "import sys; print('\n'.join(sys.path))"

/home/linuxbrew/.linuxbrew/Cellar/python@3.14/3.14.8/lib/python314.zip
/home/linuxbrew/.linuxbrew/Cellar/python@3.14/3.14.8/lib/python3.14
/home/linuxbrew/.linuxbrew/Cellar/python@3.14/3.14.8/lib/python3.14/lib-dynload
/home/linuxbrew/.linuxbrew/lib/python3.14/site-packages
```

No `/usr/lib64/python3.14/site-packages` entry → `import gi` raises
`ModuleNotFoundError`.

> **Key insight:** the identical major/minor version on both sides (3.14.7 vs
> 3.14.8) is what makes this bug deceptive. The failure looks like "PyGObject is
> not installed" or "PyGObject is broken", when in fact it is installed and
> healthy — it is simply invisible to the wrong interpreter. **Installing
> `python3-gi` again, or `pip install PyGObject` into Homebrew Python, would not
> have been the right fix** (and the latter would likely fail to build against
> the system `gobject-introspection` headers).

---

## 3. Evidence

Collected during the investigation, in order:

```console
# 1. Which interpreters exist, and which one wins on PATH
$ which -a python3
/home/linuxbrew/.linuxbrew/bin/python3
/home/linuxbrew/.linuxbrew/bin/python3
/usr/bin/python3

$ python3 -V
Python 3.14.8                      # Homebrew

# 2. The winning interpreter cannot import gi
$ python3 -c "import gi; print(gi.__file__)"
ModuleNotFoundError: No module named 'gi'

# 3. gi exists on disk — installed by the distro
$ find / -maxdepth 6 -name gi -type d -path '*packages*'
/usr/lib64/python3.14/site-packages/gi

# 4. The system interpreter imports it fine  → the package is NOT missing
$ /usr/bin/python3 -V
Python 3.14.7
$ /usr/bin/python3 -c "import gi; print('gi OK ->', gi.__file__)"
gi OK -> /usr/lib64/python3.14/site-packages/gi/__init__.py

# 5. Homebrew python's sys.path has no /usr/lib64 entry (see above)

# 6. Distro confirmation
$ cat /etc/os-release | head -3
NAME="Fedora Linux"
VERSION="44 (Workstation Edition)"
ID=fedora
```

Conclusion: the dependency was present and functional; only the **interpreter
selection** was wrong.

---

## 4. The fix

Pin the shebang to the absolute path of the **system** interpreter, so `$PATH`
order and Homebrew can no longer influence which Python runs the script:

```diff
--- /home/shams/.local/bin/satty-screenshot
+++ /home/shams/.local/bin/satty-screenshot
-#!/usr/bin/env python3
+#!/usr/bin/python3
```

A short explanatory comment was added directly beneath the shebang to stop this
from being "helpfully" reverted to `env python3` later:

```python
#!/usr/bin/python3
# Interpreter is pinned to the system python on purpose: `python3` on PATH
# resolves to Homebrew python, which cannot see the distro's PyGObject
# (`gi` lives in /usr/lib64/python3.14/site-packages). See
# ~/Desktop/Linux-Productivity-Playbook/backup-original/Linux-Setup/satty_incident_inspection.md
```

Nothing else in the script's logic was changed — the portal handshake and the
`flatpak run org.satty.Satty --filename <path> --fullscreen` launch were already
correct.

### Ownership note

The original file was owned by **`root`** (`-rwxr-xr-x root root`) even though it
lives in a user-owned directory, and `sudo` on this host requires a password.
Because `/home/shams/.local/bin` is owned by `shams`, the file could be replaced
via a directory-level rename without `sudo`. The new file is therefore owned by
`shams` — which is the correct state for a per-user script. A verbatim copy of
the original was preserved first (see [Rollback](#8-rollback)).

---

## 5. Verification

**a. The reported error is gone.** The failing case vs. the fixed case:

```console
$ /usr/bin/env python3 -c "import gi"      # old shebang's interpreter
ModuleNotFoundError: No module named 'gi'

$ head -1 ~/.local/bin/satty-screenshot | sed 's|#!||' | xargs -I{} \
      {} -c "import gi; print('gi imports OK:', gi.__file__)"
gi imports OK: /usr/lib64/python3.14/site-packages/gi/__init__.py
```

**b. The script's real import chain works** (`gi.require_version` +
`Gio`/`GLib`, and the exact D-Bus payload the script constructs):

```console
$ /usr/bin/python3 - <<'EOF'
import gi
gi.require_version('Gio', '2.0')
from gi.repository import Gio, GLib
print("gi           :", gi.__file__)
print("Gio / GLib   : OK")
print("Variant parse:", GLib.Variant.parse(GLib.VariantType.new("(sa{sv})"),
      '("", {"interactive": <true>, "modal": <true>})', None, None).print_(True))
EOF
gi           : /usr/lib64/python3.14/site-packages/gi/__init__.py
Gio / GLib   : OK
Variant parse: ('', {'interactive': <true>, 'modal': <true>})
```

**c. Syntax check passes** for the installed file: `/usr/bin/python3 -m py_compile` → `SYNTAX OK`.

**d. Remaining runtime dependencies are healthy** — nothing else was blocking
the workflow:

```console
$ flatpak list | grep -i satty
Satty   org.satty.Satty   v0.22.0   satty-origin   system

$ busctl --user list | grep -i portal
org.freedesktop.portal.Desktop   3297   xdg-desktop-portal   ...   (running)
org.freedesktop.impl.portal.desktop.gnome   3385   ...              (running)
```

**e. Not verified interactively:** the end-to-end region-select → annotate flow
requires a human to drag on the GNOME screenshot UI, so it was deliberately not
triggered during a headless check. Press **Print Screen** (or run
`~/.local/bin/satty-screenshot` in a terminal) to confirm; expect:

```
captured: /home/shams/Pictures/Screenshots/Screenshot From <timestamp>.png
satty rc: 0
```

---

## 6. Why it happened (contributing factors)

1. **Documented shebang was wrong from the start.** The playbook's setup guide
   published `#!/usr/bin/env python3`. It presumably worked when first written
   (before Homebrew Python was installed, or from a shell with a different
   `PATH`). The docs also told the reader to verify with `python3 -c "import gi"`
   — a check that *itself* passes/fails depending on which `python3` is first,
   so it could not have caught this.
2. **`env python3` is PATH-dependent.** Any GUI/keybinding context, cron job,
   SSH session, or future edit to `~/.bashrc`/`~/.profile` can silently change
   the interpreter and therefore the available packages.
3. **Two same-minor-version Pythons side by side** masked the cause — it looked
   like a packaging problem rather than an interpreter-selection problem.
4. **Script ran as root-owned file**, so the fix needed a directory-level
   replacement rather than a normal in-place edit.

---

## 7. Prevention / hardening

- **Rule of thumb:** for scripts that need OS-packaged bindings (`gi`, `dbus`,
  `systemd`, distro `apt`/`dnf` helpers), use an **absolute interpreter
  shebang** — `#!/usr/bin/python3`. Reserve `#!/usr/bin/env python3` for scripts
  that only use `pip`-installable packages.
- **Verify with the same interpreter that will run the file**, not with bare
  `python3`:
  ```bash
  head -1 script | sed 's|#!||' | xargs -I{} {} -c "import gi"
  ```
- **Keep user scripts user-owned.** If a script under `~/.local/bin` is owned by
  `root`, in-place edits fail and force `sudo`; fix with
  `sudo chown "$USER" ~/.local/bin/<script>`.
- **Audit other scripts** for the same latent failure:
  ```bash
  grep -rn '^#!/usr/bin/env python3' ~/.local/bin/ ~/.config/systemd/user/ 2>/dev/null
  ```
  Any hit that imports `gi` (or other distro modules) needs the same pinning.
- **Optionally**, if Homebrew Python must stay first on `PATH`, install PyGObject
  into it — but this is *not* recommended: it duplicates a system package,
  requires `gobject-introspection-devel` + a C toolchain to build, and makes the
  script depend on a Homebrew-managed environment.

---

## 8. Rollback

The original file was preserved verbatim before the change:

```
/home/shams/.local/bin/satty-screenshot.orig.bak
```

To revert:

```bash
cp ~/.local/bin/satty-screenshot.orig.bak ~/.local/bin/satty-screenshot
chmod 755 ~/.local/bin/satty-screenshot
```

> Reverting reintroduces the `ModuleNotFoundError`. If the old behaviour is
> needed temporarily, run the script explicitly with the system interpreter:
> `/usr/bin/python3 ~/.local/bin/satty-screenshot`

The documentation changes are tracked in the playbook git repository and can be
inspected with `git diff` / reverted with `git checkout -- <file>`.

---

## 9. Files touched

| File | Change |
|------|--------|
| `/home/shams/.local/bin/satty-screenshot` | **Fixed** — shebang `env python3` → `/usr/bin/python3`, plus explanatory comment; now owned by `shams` |
| `/home/shams/.local/bin/satty-screenshot.orig.bak` | **New** — verbatim backup of the original (root-owned, `env python3`) |
| `backup-original/Linux-Setup/satty_incident_inspection.md` | **New** — this report |
| `backup-original/investigations-log/satty-screenshot-inv.md` | **Updated** — corrected shebang, added warning callout + troubleshooting row |
| `backup-original/Linux-Setup/Linux_Setup_Fd.md` | **Updated** — corrected shebang and `import gi` check, added warning callout |

---

## 10. Related documents

- [`investigations-log/satty-screenshot-inv.md`](../investigations-log/satty-screenshot-inv.md)
  — the original setup guide for the Print Screen → satty workflow (Sections 5
  and 7 carry the corrected guidance).
- [`Linux_Setup_Fd.md`](Linux_Setup_Fd.md) — Fedora setup notes containing the
  same script.

**Quick-reference summary**

| | |
|---|---|
| **Symptom** | `ModuleNotFoundError: No module named 'gi'` at line 6 |
| **Root cause** | `#!/usr/bin/env python3` picked Homebrew Python 3.14.8 (first on `$PATH`); PyGObject only exists in Fedora's `/usr/lib64/python3.14/site-packages`, which Homebrew's `sys.path` excludes |
| **Fix** | Pin shebang to `#!/usr/bin/python3` |
| **Verified** | `gi`/`Gio`/`GLib` import + `GLib.Variant` payload OK; portal and Satty flatpak present |
| **Do not** | `pip install PyGObject` into Homebrew Python — wrong layer, wrong fix |