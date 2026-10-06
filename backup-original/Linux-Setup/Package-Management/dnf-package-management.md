Fedora uses the [DNF Package Management System](https://docs.fedoraproject.org/en-US/quick-docs/package-management/) as its default high-level command-line tool for installing, updating, and removing software. [1, 2] 

## Essential DNF Commands

* `sudo dnf search` <package>: Search for available software packages matching a term.
* `sudo dnf install <package>`: Download and install a package along with its required dependencies.
* `sudo dnf remove <package>`: Uninstall a package from your system.
* `sudo dnf upgrade --refresh`: Refresh repository metadata and upgrade all system packages to their latest versions.
* `sudo dnf autoremove`: Remove orphaned dependencies that were installed automatically but are no longer needed.
* `dnf info <package>`: Display detailed specifications, size, license, and description of a package. [2, 3, 4] 
