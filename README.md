# OmniStore

OmniStore is a single-executable front end for Windows and Linux that unifies multiple existing package managers into a single app store interface. Instead of hosting its own apps, it searches across your favorite managers, merges the results into unified app cards, and handles installations and background updates automatically.

## Key Features

* **Unified Search & Installation:** Searches multiple catalogs simultaneously and merges results so that a single app appears as one unified card.


* **Automatic Background Updates:** Keeps your installed applications up to date silently in the background, featuring per-app rules for auto-updating, notifying, or pinning specific versions.


* **OS System Updates:** Manages Windows Update and Linux distribution upgrades directly from the application interface.


* **Safety & Trust Ratings:** Provides a 0–100 safety score, trusted author labels, and look-alike detection to ensure secure software downloads.


* **My Lists & Bundling:** Allows users to create lists of apps to install in one go, which can even be baked directly into a new copy of the executable.


* **Multilingual Support:** Offers ten interface languages, including complete Dutch support and basic support for Spanish, German, French, Italian, and more.



## Supported Package Managers

OmniStore seamlessly integrates with a wide variety of package managers:

* **Windows:** WinGet, Chocolatey, Scoop


* **Linux:** apt, dnf, pacman, aur, zypper


* **Universal:** Flatpak, Snap, Homebrew


* **Developer Tools:** pip, pipx, npm, Cargo, .NET tools


* **Other Sources:** GitHub Releases, AppImage, Web..



## How It Works

OmniStore requires no installation and runs as a single static Go binary. It serves its own embedded web UI to a local Chromium-family browser window running in app mode. The backend directly interfaces with your local package manager CLIs—elevating via UAC, pkexec, or sudo when necessary—and online APIs to fetch catalog data. It can also function purely as a command-line utility for terminal users.

## Developer Resources

If you want to understand how the architecture fits together at runtime, how to add a new package manager provider, or how to build the project from source, please reference `DEVELOPER-GUIDE.md`.
