## Flatpak on Fedora 44

Flatpak  is a system for installing and running desktop applications in isolated sandboxes. Fedora commonly uses Flatpak alongside RPM packages, particularly for GUI applications.

 ### Useful commands

 | Command | Purpose |
| --- | --- |
| `flatpak --version` | Show the installed Flatpak version |
| `flatpak remotes` | List configured repositories/remotes |
| `flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo` | Add Flathub |
| `flatpak search <name>` | Search for an application |
| `flatpak install flathub <APP_ID>` | Install an application |
| `flatpak list` | List installed Flatpak apps and runtimes |
| `flatpak info <APP_ID>` | Show information about an installed app |
| `flatpak run <APP_ID>` | Launch an application |
| `flatpak update` | Update installed apps and runtimes |
| `flatpak uninstall <APP_ID>` | Remove an application |
| `flatpak uninstall --unused` | Remove unused runtimes/dependencies |
| `flatpak repair` | Check and repair the local Flatpak installation |
| `flatpak history` | Show Flatpak installation/update history |
| `flatpak override <APP_ID>` | View or modify an application's sandbox permissions |

### Common examples

```
# Search
flatpak search firefox

# Install
flatpak install flathub org.mozilla.firefox

# Update everything
flatpak update

# Remove an app
flatpak uninstall org.mozilla.firefox

# Remove unused runtimes
flatpak uninstall --unused

# See an app's permissions
flatpak info --show-permissions org.mozilla.firefox
```

 ### System-wide vs per-user

 By default, Flatpak can operate system-wide or for your user account:

```
flatpak --system list
flatpak --user list

flatpak --user install flathub <APP_ID>
flatpak --user uninstall <APP_ID>
```

 For everyday Fedora use, the most important commands to remember are **`search` → `install` → `update` → `uninstall`**, plus **`list`** and **`info`** for inspection.
