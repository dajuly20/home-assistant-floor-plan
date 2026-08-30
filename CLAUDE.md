# CLAUDE.md — Home Assistant Floor Plan Plugin for Sweet Home 3D

## Project Overview

Java 8 plugin for Sweet Home 3D that generates Home Assistant floor plan dashboards. It renders floor plans as images and produces Lovelace YAML configs with entity overlays (lights, switches, sensors, etc.).

## Build System

```bash
make              # Build plugin → produces .sh3p file
make test         # Build + install + launch SH3D for manual testing
make clean        # Remove build dir and .sh3p files
make distclean    # clean + remove dl/ (downloaded deps)
make install      # Install to ~/.eteks/sweethome3d/plugins/
```

- Requires Java 8 (or Docker with Eclipse Temurin 8 — auto-used if local Java missing)
- Dependencies (SweetHome3D-7.5.jar, j3dcore.jar, vecmath.jar) are downloaded automatically by the Makefile
- Output artifact: `HomeAssistantFloorPlan.sh3p`

## Key Source Files

All source files are in `src/com/shmuelzon/HomeAssistantFloorPlan/`:

| File | Role |
|------|------|
| `Plugin.java` | SH3D plugin entry point |
| `Controller.java` | Core logic: rendering, YAML generation, HA API, OAuth2 |
| `Panel.java` | Main UI panel with entity trees and config |
| `Entity.java` | HA entity model (position, icon, conditions, actions) |
| `EntityOptionsPanel.java` | Per-entity config UI |
| `Scene.java` | Single render scene (lights on/off combination) |
| `Scenes.java` | Manages rendering modes: CSS, Room Overlay, Complete Renders |
| `Settings.java` | Persists config in the SH3D project file |
| `Utils.java` | Fuzzy matching, string normalization helpers |

## Architecture Notes

- **No external frameworks** — pure Java with SH3D APIs
- **Three rendering modes:** CSS overlays, Room Overlay (partial renders), Complete Renders (full per-state images)
- **HA connection:** OAuth2 flow via `java.net`, entities fetched from HA REST API
- **Persistence:** Settings stored inside the `.sh3d` project file via SH3D's own persistence API

## Testing

No automated tests. Testing = `make test` → manual verification in Sweet Home 3D.

## Git Remotes

- `origin` — dajuly20's fork (main development)
- `upstream` — shmuelzon's original repo
- `nstod` — another fork with base-folder feature

## Branches

- `master` — stable
- `feature/ha-integration-improvements` — current dev branch (active)
- `feature/ha-api-entity-loader`, `feature/auto-upload`, `feature/person-entities-ha-floor-plan` — feature branches

## Plugin Versions

Built `.sh3p` files in root: v0.15.3–v0.15.7 (for reference/rollback)

## CI/CD

GitHub Actions (`.github/workflows/`): builds on push/PR to master and on releases, uploads `.sh3p` to GitHub Releases automatically.
