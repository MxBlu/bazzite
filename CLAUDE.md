# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Bazzite is a custom Fedora Atomic desktop image optimized for gaming, built on top of Universal Blue (ublue-os/main) base images. It produces container images pushed to GHCR (`ghcr.io/ublue-os/bazzite*`) that users rebase onto their systems via `rpm-ostree`/`bootc`.

The current branch (`ayaneo_flip`) is focused on adding support for the **Ayaneo Flip DS** handheld gaming device.

## Build Commands

Requires: `just`, `podman` (or docker), Linux environment.

```bash
just build [target] [image]         # Build a container image locally
just build-iso-release [target]     # Build installable ISO
just run-container [target] [image] # Run built image in container
just list-images                    # Show available image variants
just clean-images                   # Remove locally built images
just just-check                     # Validate Just syntax (runs in CI on all PRs)
just just-fix                       # Auto-fix Just syntax formatting
```

Targets: `bazzite`, `bazzite-gnome`, `bazzite-nvidia`, `bazzite-gnome-nvidia`, `bazzite-deck`, `bazzite-deck-gnome`, etc.

Images: `kinoite` (KDE) or `silverblue` (GNOME).

## Testing

Tests use `dgoss` (Docker-based Goss). They run during CI against built images and are located in `tests/dgoss/tests.d/`. Tests verify package presence, service status, and system configuration.

## Architecture

### Build Pipeline

```
Containerfile (multi-stage)
├── FROM kernel-bazzite       → custom kernel stage
├── FROM nvidia-drivers       → NVIDIA drivers stage
├── FROM scratch AS ctx       → mounts build_files/
├── FROM base AS bazzite      → desktop build (KDE or GNOME)
│   ├── Copies system_files/desktop/shared/ and system_files/desktop/{kinoite|silverblue}/
│   ├── Installs packages via dnf5 from COPRs, RPMFusion, Terra
│   └── Runs build_files/ scripts (cleanup, finalize, image-info)
├── FROM bazzite AS bazzite-deck    → extends desktop for handheld gaming
│   ├── Copies system_files/deck/shared/ and system_files/deck/{kinoite|silverblue}/
│   └── Adds HHD (Handheld Daemon), gamescope, Steam gaming mode
└── FROM bazzite AS bazzite-nvidia  → desktop + proprietary NVIDIA
```

### Directory Map

- **`Containerfile`** — single source of truth for build order and package installation
- **`system_files/desktop/`** — files layered into all desktop variants
  - `shared/` — KDE + GNOME common config, systemd units, udev rules, polkit rules, `just` scripts
  - `kinoite/` — KDE-specific overrides (GNOME shell extensions in `silverblue/`)
- **`system_files/deck/`** — additional files for handheld/deck variants (layered on top of desktop)
  - Key subdirs: `usr/lib/udev/rules.d/`, `usr/share/pipewire/hardware-profiles/`, `usr/share/wireplumber/hardware-profiles/`, `usr/share/gamescope/scripts/`
- **`system_files/nvidia/`** — NVIDIA-specific additions
- **`system_files/overrides/`** — system-wide overrides (bootloader, icons)
- **`build_files/`** — shell scripts run during container build (not shipped in final image)
- **`firmware/`** — hardware firmware files included in all variants
- **`just_scripts/`** — CI/local orchestration wrappers for `just` commands
- **`system_files/desktop/shared/usr/share/ublue-os/just/`** — user-facing `just` commands installed on the running system

### Adding Device-Specific Support (Handheld Devices)

Device support in `system_files/deck/shared/` follows established patterns. For a new device:

1. **udev rules** — `usr/lib/udev/rules.d/` for device matching, fingerprint readers, special hardware
2. **Audio profiles** — `usr/share/pipewire/hardware-profiles/<dmi-board-vendor>-<dmi-board-name>/pipewire.conf.d/` and `usr/share/wireplumber/hardware-profiles/` for device-tuned audio
3. **Display config** — `usr/share/gamescope/scripts/00-gamescope/displays/` for custom display tuning (Lua scripts)
4. **WiFi/sleep quirks** — `usr/lib/udev/rules.d/50-wifi-sleep.rules` matches on `ATTR{board_vendor}` and `ATTR{board_name}` DMI attributes
5. **systemd services** — device-specific services in `usr/lib/systemd/system/`

Existing device examples to reference:
- ROG Ally: `usr/lib/udev/rules.d/50-ally-fingerprint.rules`, `usr/share/pipewire/hardware-profiles/asustek computer inc.-rog ally rc71l_rc71l/`
- GPD Win 4: `usr/lib/udev/rules.d/50-gpd-win-4-fingerprint.rules`, `usr/share/gamescope/scripts/00-gamescope/displays/gpd.win4.lcd.lua`
- AYANEO Air: `usr/lib/udev/rules.d/50-wifi-sleep.rules` (WiFi sleep quirk)
- Lenovo Legion Go: `usr/lib/udev/rules.d/50-lenovo-legion-controller.rules`, `usr/share/wireplumber/hardware-profiles/lenovo-83e1/`

### Commit/PR Conventions

Follow Conventional Commits format. Scope with the device/component name:

```
fix(flip): fix screen rotation on Ayaneo Flip DS
feat(deck): add audio profile for Ayaneo Flip DS
```

Generic `feat:` or `fix:` is acceptable when scope is unclear. These are used to auto-generate changelogs.
