# Faugus Launcher: Agent & Project Documentation

This document provides a comprehensive overview of **Faugus Launcher**—a simple, lightweight, GTK-based Linux game launcher designed to run Windows games on Linux using UMU-Launcher. It details the project's architecture, technology stack, directory structure, and key workflows to help developers and AI agents understand, navigate, and modify the codebase.

---

## 1. Project Overview & Tech Stack

Faugus Launcher provides a graphical user interface (GUI) and runner wrappers to organize, configure, and launch Windows games using compatibility layers on Linux. It leverages **UMU-Launcher** (`umu-run`), which is a unified launcher interface built to run non-Steam games using Proton/Wine runtimes and Steam Linux Runtime (sniper/soldier) containers.

### Core Technology Stack
* **Language:** Python 3
* **GUI Toolkit:** GTK 3 (via PyGObject: `gi.repository.Gtk`, `Gdk`, `GLib`, `Pango`)
* **System Tray integration:** `AyatanaAppIndicator3` (provides a persistent menu icon)
* **Build System:** Meson & Ninja (compilation and script deployment/installation)
* **Translation & Localization:** Gettext engine (`languages/` subdirectory)
* **External Python Dependencies:**
  * `requests`: Interfacing with GitHub/Gitea release APIs.
  * `pillow`: Image processing and scaling.
  * `vdf`: Reading and writing Steam's binary VDF shortcut files.
  * `psutil`: Monitoring and terminating game process trees.
  * `pygame`: Initializing and handling gamepad event loops.
  * `icoextract` & `imagemagick` (system command): Extracting and converting Windows `.exe` icon resources into `.png` formats.

---

## 2. Directory & Component Structure

The repository is structured as follows:

```
faugus-launcher/
├── assets/                 # Branding assets, icons, and UI images
├── data/                   # Desktop files, icons, and UI database templates
├── languages/              # Localization files (.po/.mo files)
├── screenshots/            # UI screenshots for documentation
├── faugus-launcher         # Wrapper shell script (routes system execution)
├── faugus_run.py           # Legacy wrapper script (updates shortcuts & execs runner)
├── meson.build             # Project-wide build script
└── faugus/                 # Main Python package directory
    ├── meson.build         # Package build script
    ├── launcher.py         # Main GTK 3 UI (grid views, category filter, list management)
    ├── runner.py           # Handles game execution, environments, and locks
    ├── config_manager.py   # Config parser and serializer
    ├── path_manager.py     # Path resolver and flatpak environment detection
    ├── steam_setup.py      # Steam shortcuts integration
    ├── components.py       # Auto-updater for umu-run, BattlEye, and EAC runtimes
    ├── proton_downloader.py# Downloads Proton versions (GE-Proton, CachyOS, EM, DW)
    ├── proton_manager.py   # GUI/Backend to install/remove Proton versions
    ├── backup.py           # Game prefix and application backup tool
    ├── gamepad.py          # Pygame-based controller navigation
    ├── keyboard.py         # Keyboard shortcuts/navigation handler
    ├── ea_fix.py           # Workarounds for EA App launcher compatibility
    ├── shortcut.py         # Handles .desktop file creation and modification
    └── utils.py            # Shared utility functions (UI wrappers, icon extractors)
```

---

## 3. Detailed Component Breakdown

### A. Execution Entrypoints
1. **`faugus-launcher` (Shell Wrapper):** Acts as the primary system-wide binary. It routes incoming commands:
   * `--shortcut <args>`: Runs the shortcut editor (`faugus.shortcut`).
   * `--game <game_id>`: Launches the game directly (`faugus.runner --game`).
   * `--run <args>`: Executes a raw wine tool/command inside the prefix (`faugus.runner`).
   * Default: Starts the launcher UI (`faugus.launcher`).
2. **`faugus_run.py` (Script):** Ensures backward compatibility. It scans the desktop and Steam configuration directories to replace legacy launcher execution commands with modern wrappers before exec-ing `faugus.runner`.

### B. The User Interface (`faugus/launcher.py`)
This file implements the main GTK 3 window. Key features include:
* **Display Modes:** Supports List (compact), Blocks (icon grid), and Banners (artwork layout) views.
* **Filtering and Sorting:** Sorts list by Alphabetical order, Playtime, Last Played, or Custom order. Allows categorization of games into distinct tabs/sidebar folders.
* **Drag-and-Drop (DnD):** Custom sorting uses GTK's DnD capabilities to reorganize grid items and update the `custom-order.json` configuration file.
* **Game Management:** Modals for adding/editing games, managing additional launcher settings (launch arguments, additional applications to run, FPS limits, MangoHud overlays, and system tweaks).
* **System Tray:** Implements `AyatanaAppIndicator3` to hide the window to the tray or launch recent games from a quick menu.

### C. The Runner Engine (`faugus/runner.py`)
Responsible for establishing the compatibility context and launching games via UMU:
1. **Environment Setup:** Exports system variables such as `DRI_PRIME` (discrete GPU), `PROTON_ENABLE_WAYLAND` (Wayland driver), `PROTON_USE_WOW64` (Wow64 support), and runtimes paths for EAC/BattlEye.
2. **Prefix Locking:** Prevents dual-boot or concurrent instances of the same Wine prefix using Python's `fcntl.flock` on `.faugus.lock`.
3. **Execution Pipeline:** Downloads runtime components if missing (`faugus.components` and `faugus.proton_downloader`), runs the final command under `umu-run`, and shows a GTK splash screen indicating setup progress.
4. **Playtime tracking:** Spawns a background thread that monitors child process trees (using `psutil`) to aggregate playing duration. It writes playtime metadata to the game configuration on exit.

### D. Utility & Integration Components
* **`faugus/config_manager.py`:** Standardizes a set of 40+ configuration parameters (like UI states, paths, backup frequencies, and gamepad toggles) stored in raw config format.
* **`faugus/path_manager.py`:** Handles environment detection. Specifically, it exposes flatpak detection (`IS_FLATPAK`) to determine folder permissions, system compatibility paths, and user data locations.
* **`faugus/gamepad.py`:** Hooks into Pygame to support game controller navigation, mapping confirmations, cancels, configurations, settings, and power options to standard Playstation/Xbox button layouts.
* **`faugus/utils.py`:** Contains image validity checkers, file dialog builders, gettext helpers, and PE icon extraction functions using `icoextract` and `magick`.

---

## 4. Key Developer Workflows

### Running Locally
To test the launcher locally in the development workspace:
```bash
# To run the main launcher GUI
python3 -m faugus.launcher

# To run a game directly by its ID
python3 -m faugus.runner --game <game-id>
```

### Packaging & Build
The project uses Meson for installation. To configure and compile:
```bash
meson setup builddir --prefix=/usr
cd builddir
ninja
sudo ninja install
```

### AppImage Packaging
Faugus Launcher can be packaged as a standalone AppImage using `appimage-builder`. 
* **Recipe:** `AppImageBuilder.yml` defines the package definitions, runtime paths, and GTK 3 system libraries.
* **Build workflow:** The `.github/workflows/appimage.yml` file automates the build process:
  1. Installs Meson, Ninja, and Python dependencies.
  2. Runs Meson installation targeting a local `AppDir` folder.
  3. Bundles external PyPI libraries (`vdf`, `icoextract`, `pefile`) into the `AppDir`.
  4. Runs `appimage-builder` to generate a self-contained executable.

### Path Conventions
* **Default Prefixes Location:** `~/Faugus/`
* **Custom Config / Metadata:** `~/.config/faugus-launcher/`
* **Proton / Runner Tools:** `~/.local/share/Steam/compatibilitytools.d/`
* **System/Application Shortcuts:** `~/.local/share/applications/` and `~/Desktop/`
