# Public Repository Migration Provenance

This repository intentionally starts with fresh public Git history rather than publishing the historical private development repository.

## Legacy source

- Private legacy repository: `NikichMods/RainAndWindVolumeControl-legacy-private`
- Legacy mod name: **Rain and Wind Volume Control**
- Accepted legacy version: **1.0.1**
- Accepted freeze branch: `baseline/1.0.1-release`
- Accepted freeze commit: `40be3ce41d13e33e3ea6ec8ff83f91de416071b3`
- Accepted DLL SHA-256: `cc2e6b60adf602effda346beb3e913c8893c5844cedb64737796847a477c9a7c`

## Public migration

The public repository uses the name **Rain & Wind Volume Controls** and technical identity `RainWindVolumeControls` for the project/assembly/DLL.

The BepInEx GUID remains `rainwind.gyk.volume.control` intentionally so plugin/config identity remains continuous.

Version 1.1.0 is used for the renamed public package so the immutable accepted 1.0.1 binary is never silently replaced by different bytes under the same version.

Historical experimental branches and private repository bookkeeping are not imported into this fresh public history.
