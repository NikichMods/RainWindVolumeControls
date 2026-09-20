# Rain & Wind Volume Controls — Project Rules

The global engineering baseline for this repository is `NikichMods/DevRules`. Before substantive implementation, read `ENGINEERING_RULES.md`, `CI_POLICY.md`, `GIT_WORKFLOW.md`, and `PROJECT_BOOTSTRAP.md` there. This file contains only project-specific additions and explicit exceptions.

## Project identity and scope

- Public project: **Rain & Wind Volume Controls**.
- Game: `Graveyard Keeper 1.407`.
- Repository: `NikichMods/RainWindVolumeControls`.
- Canonical project: `RainWindVolumeControls.csproj`.
- Canonical runtime source: `src/RainWindVolumeControls.cs`.
- Stable BepInEx GUID: `rainwind.gyk.volume.control`.
- Scope: independent player-facing volume controls for rain and wind while preserving weather visuals and unrelated audio/gameplay behavior.

Preserve the accepted lightweight architecture:

- no per-frame polling solely for configuration;
- no repeated hierarchy/global scans during steady-state use;
- no save-data mutation;
- no change to weather visuals;
- changes affect only the verified native rain/wind audio groups.

Do not turn this into a general audio mixer or weather overhaul without explicit user approval.

## Closed post-audit architecture verdict

Read `docs/POST_AUDIT_WEATHER_AUDIO_1.1.0.md` before reopening the weather/audio Harmony-seam question.

The 2026-09-19 post-audit pass closed the former concern that the production
`SmartAudioEngine.SetSoundVolume(string, float)` Prefix should automatically be replaced by
`SmartWeatherState.UpdateWeatherVolume(float)`.

Accepted verdict: **keep current implementation**.

The current general setter is broad by call surface but narrow by effect because it immediately
returns for every group except exact `rain_environment` / `wind_environment` IDs. It preserves
the already accepted immediate `SettingChanged` behavior with only two raw float caches and no
weather-instance ownership. A weather-local replacement requires additional state/lifecycle
coupling to reapply an already-active sound, and no material performance benefit has been proven.

Do not repeat this research solely for semantic purity. Reopen it only for new concrete evidence,
such as a measured performance cost, a reproduced collision/behavior bug, or a newly established
state-free weather-local seam that preserves immediate live apply.

## Public/research boundary

This public repository contains only redistributable project material: our source, documentation, build definitions, and our own release binaries/assets. Reverse-engineering material that genuinely needs retention belongs in the private `NikichMods/GraveyardKeeperResearch` repository; durable verified facts needed by production belong in public project documentation.

## Repository and release contract

- `main` is the accepted stable public line.
- Runtime or packaging candidates remain off `main` until the required acceptance gate is satisfied.
- Every numbered DLL handed to the user is immutable and tied to exact source.
- `docs/TEST_BUILD_LOG.md` is the durable test-build record.
- Public stable binaries are published through GitHub Releases after acceptance.
- For development/test handoff, the user prefers a ready raw versioned DLL such as `Rain & Wind Volume Controls 1.2.0.dll`, not a ZIP.
- For end-user installation and public distribution surfaces such as Nexus, the canonical installed filename is `RainWindVolumeControls.dll` with no version in the filename; version identity belongs in plugin metadata and the surrounding release/store entry.
- A filename-only rename of an already accepted DLL for public packaging is allowed only when the bytes are unchanged and the accepted SHA-256 still matches.
- Existing historical releases do not need retroactive repackaging solely to adopt the stable installed filename convention.

## CI policy for this repository

Follow `DevRules/CI_POLICY.md` and `DevRules/GIT_WORKFLOW.md`.

- Use hosted CI only at coherent candidate/handoff boundaries.
- Documentation-only changes do not require hosted CI.
- Windows remains the canonical runner until a cheaper runner is explicitly proven equivalent for this project.
- A clean Release build is required before a new DLL is handed to the user.
- After acceptance, publish the exact tested artifact to GitHub Releases; do not rebuild different bytes under the same version.

## Long-lived sources of truth

Use `README.md`, `CHANGELOG.md`, `docs/MIGRATION_PROVENANCE.md`, `docs/TEST_BUILD_LOG.md`, `docs/POST_AUDIT_WEATHER_AUDIO_1.1.0.md`, the canonical source/project files, and current public repository history. Historical pre-public evidence remains available in `NikichMods/RainAndWindVolumeControl-legacy-private`.

When chat memory conflicts with accepted repository evidence, investigate the conflict before changing code.
