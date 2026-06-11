# MicaKDE
KDE Plasma with unified Windows 11 Design and Tweaks



# MicaKDE — Project Specification v0.1

> A KDE Plasma fork targeting the look and feel of a debloated Windows 11,
> rebuilt with modern Rust backends, Vulkan-native rendering, and a unified
> design language across the entire desktop.

-----

## Vision

MicaKDE is not a theme. It is a full desktop environment fork that replaces
KDE Plasma’s defaults with a cohesive, modern, Windows 11-inspired experience
— without any of Windows’ telemetry, lock-in, or bloat.

-----

## 1. Rendering Stack

### Wayland Only

- **No X11 / XWayland support.** Zero legacy protocol surface.
- KWin configured as pure Wayland compositor from day one.
- No fallback to X11 session.

### Vulkan Only

- KWin’s **Vulkan rendering backend** is the only supported backend.
- **OpenGL is disabled** — no GL compositor fallback.
- **Vulkan Video** (VK_KHR_video_decode_queue / VK_KHR_video_encode_queue)
  for hardware-accelerated media decode/encode.
- GStreamer pipeline uses Vulkan Video sink where available.
- Target GPUs: NVIDIA Blackwell (RTX 50xx), AMD RDNA3+, Intel Arc.

-----

## 2. Visual Design Language

### Aesthetic: Fluent Design / Mica

- Mica material effect (background-aware blur + tint) on all window chrome.
- Acrylic (frosted glass) on panels, popups, context menus.
- Rounded corners: `8px` default, `4px` for small controls.
- Dark mode first. Light mode supported.
- Subtle drop shadows, no harsh borders.
- Windows 11 Snap Layouts-style window snapping overlay.

### Typography

- **Display / UI font:** [Inter](https://rsms.me/inter/) — variable weight,
  clean, neutral, Segoe UI-compatible feel.
- **Monospace:** [Geist Mono](https://vercel.com/font) or JetBrains Mono.
- Hinting: `slight`, subpixel antialiasing off on HiDPI, grayscale AA on 1x.
- Font size default: `10.5pt` (matches Win11 default).

### Unified GTK + Qt Theming

- **Goal:** A GTK4 app and a Qt6 app must be indistinguishable at a glance.
- Qt style: custom QStyle written in Rust (via `qttypes` / `cxx-qt`) or
  maintained QSS override layer on top of Breeze.
- GTK4: custom `libadwaita` color overrides + `gtk.css` delivering identical
  radius, color, and spacing tokens.
- GTK3: `adw-gtk3` as base, patched to match token set.
- Shared design token file (JSON/TOML) consumed by both Qt and GTK layers.
- Icon theme: custom Fluent-inspired icon set (fork of `fluent-icons-theme`).
- Cursor: `Fluent` cursor theme.

-----

## 3. Shell & Panel

### No Taskbar Settings in the Taskbar

- Right-click on the panel shows **no configuration menu**.
- Panel layout is configured exclusively via **MicaSettings** (see §5).
- This removes the “Edit Panel” / “Add Widgets” workflow entirely.
- Widgets / applets are managed from Settings → Desktop → Panel.

### Panel Behavior

- Single bottom panel, Windows 11-style centered taskbar by default.
- Clock, system tray, notification bell — right-aligned.
- Search (like Win+S) opens a full Krunner-based overlay, not a widget.

-----

## 4. Application Installation

### Default Install Path

```
/home/$USER/programs/$programname/
```

Example:

```
/home/danny/programs/firefox/
/home/danny/programs/vlc/
```

- No root required for installation of user-space apps.
- Each program is self-contained in its directory (AppImage-style or
  custom ELF bundle format).
- A per-user `PATH` entry (`~/.local/bin` symlinks) is managed automatically.
- System-wide packages (via pacman/apt) still install to `/usr` as normal.

### MicaPackage Format (future)

- Rust-based install daemon (`mica-pkgd`) handles extraction, PATH wiring,
  `.desktop` file registration, and uninstall manifests.
- No root daemon needed for user-space installs.

-----

## 5. Settings Application (MicaSettings)

A full replacement for `systemsettings` — the **single source of truth**
for all desktop configuration.

### Modules (planned)

|Module                |Description                                          |
|----------------------|-----------------------------------------------------|
|**System**            |OS info, hostname, Windows Update-style update center|
|**Display**           |Resolution, scaling, refresh rate, HDR               |
|**Sound**             |PipeWire audio, per-app volume                       |
|**Network**           |NetworkManager frontend                              |
|**Bluetooth**         |BlueZ frontend                                       |
|**Personalization**   |Theme, colors, fonts, wallpaper, Mica intensity      |
|**Desktop**           |Panel layout, widgets, virtual desktops              |
|**Programs**          |All installed apps — uninstall / repair / modify     |
|**Privacy & Security**|MicaDefender (ClamAV), firewall, permissions         |
|**Accounts**          |User management, passwordless auth config            |
|**Power**             |Sleep, hibernate, battery thresholds                 |
|**Accessibility**     |Font scaling, contrast, input assist                 |

### Update Center

- Rust backend (`mica-updates`) polling distro package API + Flatpak.
- Windows-style UI: pending updates list, changelog, “Install All” button.
- Background update checks via systemd timer.

### Programs Module

- Reads from `/home/$USER/programs/` + system package DB.
- Shows: name, version, install date, disk usage.
- Actions: Uninstall, Modify (reinstall / repair), Open folder.

-----

## 6. Security & Authentication

### Passwordless Polkit (UAC-style)

- Custom polkit authentication agent in Rust.
- **No password prompt** for the primary user by default
  (configurable in Settings → Accounts).
- Instead: Windows-style modal popup:

```
┌─────────────────────────────────────────────┐
│  🛡️  Administratorrechte erforderlich        │
│                                             │
│  „pacman" möchte Änderungen am System       │
│  vornehmen.                                 │
│                                             │
│  [  Ablehnen  ]           [  Zulassen  ]    │
└─────────────────────────────────────────────┘
```

- Optional: FIDO2 / fingerprint (libfido2 / fprintd) as second factor.
- Polkit rules stored in `/etc/polkit-1/rules.d/10-mica.rules`.

### Removed Components

|Removed        |Reason                                                       |
|---------------|-------------------------------------------------------------|
|**KWallet**    |Replaced by system keyring (libsecret + KeePassXC-compatible)|
|**KWrite**     |Redundant — Kate or external editor used                     |
|**KDE Connect**|Out of scope for v1                                          |

-----

## 7. Rust Backend Strategy

New components are written in **Rust only**. C++ forks of existing KDE
components are maintained as minimal as possible.

|Component             |Language      |Notes                            |
|----------------------|--------------|---------------------------------|
|`mica-pkgd`           |Rust          |Install daemon                   |
|`mica-updates`        |Rust          |Update center backend            |
|`mica-polkit-agent`   |Rust          |UAC-style auth popup             |
|`mica-defender`       |Rust          |ClamAV frontend + scheduled scans|
|`mica-settings-daemon`|Rust          |Settings persistence + IPC       |
|KWin fork             |C++ (upstream)|Minimal patches only             |
|Plasma-workspace fork |C++ (upstream)|Minimal patches only             |

Rust ↔ C++/Qt bridge: [`cxx-qt`](https://github.com/KDAB/cxx-qt)

-----

## 8. Development Approach

- Primary tooling: **opencode** (agentic coding) with local LLM backend.
- Models: Qwen3.6-27B (NVFP4) via vLLM for code generation.
- Claude Code / Claude API for architecture decisions and complex refactors.
- Conventional Commits (`feat:`, `fix:`, `chore:`) enforced via git hook.
- CI: GitHub Actions — build check, `cargo clippy`, `cargo test`.

-----

## 9. Distribution Target

- **Base:** Arch Linux / CachyOS
- Kernel: CachyOS kernel (BORE-EEVDF, x86-64-v3)
- Init: systemd
- Display: KWin Wayland (no SDDM replacement in v1 — use SDDM with Mica theme)
- Package format: pacman + AUR + `mica-pkgd` for user-space apps

-----

## 10. Roadmap

### Phase 1 — Foundation

- [ ] Fork plasma-workspace, systemsettings, kwin
- [ ] Apply Mica/Fluent Qt theme
- [ ] GTK4 token alignment
- [ ] Inter font integration
- [ ] Disable OpenGL KWin backend

### Phase 2 — Shell

- [ ] Remove taskbar right-click config
- [ ] MicaSettings skeleton (all modules as stubs)
- [ ] UAC-style polkit agent (Rust)

### Phase 3 — Backends

- [ ] `mica-updates` update center
- [ ] `mica-pkgd` install daemon + `/home/$USER/programs/` layout
- [ ] Programs module in MicaSettings
- [ ] `mica-defender` ClamAV frontend

### Phase 4 — Polish

- [ ] Vulkan Video media pipeline
- [ ] Full icon theme
- [ ] SDDM Mica theme
- [ ] ISO build (CachyOS-based)

-----

## License

MicaKDE forks GPL-licensed KDE code and is therefore released under
**GPL-2.0-or-later**.

New Rust components (`mica-*` crates) are dual-licensed:
**GPL-2.0-or-later OR MIT**

```
Copyright (C) 2026 MicaKDE Contributors
Based on KDE Plasma — Copyright (C) KDE Contributors
```
