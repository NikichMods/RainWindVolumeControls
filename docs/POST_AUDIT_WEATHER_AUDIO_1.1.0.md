# Weather Audio Post-Audit — 1.1.0

Date: 2026-09-19

## Status

**Verdict: keep current implementation.**

The post-audit question from the general Graveyard Keeper mod review is closed. The accepted
`SmartAudioEngine.SetSoundVolume(string, float)` Prefix remains the least-sufficient mechanism
for the required behavior. A narrower `SmartWeatherState.UpdateWeatherVolume(float)` seam is
semantically attractive for ordinary weather updates, but preserving immediate live configuration
changes without the current audio setter requires additional weather-instance/current-value
ownership. No performance defect or functional collision has been established that would justify
that extra state and lifecycle coupling.

This is a research/documentation verdict only. Production source, plugin version, config keys,
defaults, ranges, and accepted DLL are unchanged.

## Accepted baseline rechecked

- Version: `1.1.0`
- Accepted runtime source: `f27c489075b7091d41a04609303b91995fdcfcef`
- Frozen ref: `baseline/1.1.0-accepted`
- Accepted CI run: `34611974218`
- Accepted artifact: `RainWindVolumeControls-1.1.0` (`10268822659`)
- Accepted DLL SHA-256:
  `aa422a7ebe16071c6bd45d7fd6590149953aa2cdd235a6e67b6a4fb8aadb9a28`

The frozen ref still points exactly at the accepted source commit. Current `main` contains later
documentation/release bookkeeping but no production-source change relative to that accepted
runtime source.

## Current implementation

The production patch resolves and prefixes:

`SmartAudioEngine.SetSoundVolume(string sound_group_id, float volume)`

The Prefix:

1. compares the sound-group ID with `rain_environment`;
2. otherwise compares it with `wind_environment`;
3. caches the raw, game-provided pre-mod volume for the matching channel;
4. multiplies the call-local volume by the corresponding BepInEx config value;
5. leaves every unrelated sound-group call unchanged.

Recurring audio-path work contains no reflection, scene/global scan, polling loop, collection
enumeration, or intentional allocation. The only custom runtime state used for live apply is two
floats containing the last raw rain/wind values.

`SettingChanged` calls `ApplyCurrent()`. That rare/event-driven path obtains
`SmartAudioEngine.me` and invokes `SetSoundVolume` with the cached raw value. The same Harmony
Prefix then applies the newly selected multiplier exactly once. Reflection and the invocation
argument arrays therefore occur on configuration changes, not on the normal recurring audio path.

Event cleanup is symmetric in `OnDestroy()`: both `SettingChanged` handlers are removed and the
plugin's Harmony patches are unpatched.

## Established native weather/audio path

Prior accepted game-side research established the relevant Graveyard Keeper 1.407 path as:

`SmartWeatherState.Update`
→ changing weather amount
→ `SmartWeatherState.UpdateWeatherVolume(float weather_value)`
→ `GetWeatherMusicId()`
→ Rain/Wind normalization using the state's `max`
→ `SmartAudioEngine.me.SetSoundVolume(weatherMusicId, volume)`

For Rain and Wind, `GetWeatherMusicId()` resolves the same stable groups used by production:
`rain_environment` and `wind_environment`.

Separate runtime reflection evidence from the installed game also confirms a live
`SmartWeatherState` component and the methods `GetWeatherMusicId()`,
`SetWeatherSoundEnable(Boolean)`, and `UpdateWeatherVolume(Single)`.

No proprietary decompiled game source is stored in this public repository.

## Candidate B — UpdateWeatherVolume input scaling

For the established Rain/Wind normalization formula, changing the call-local input from
`weather_value` to `weather_value * multiplier` is mathematically equivalent to applying the
same multiplier after normalization:

`(weather_value * multiplier) / max == (weather_value / max) * multiplier`

for ordinary real-number arithmetic. The configured multiplier domain is `0..1`, so scaling does
not expand the native weather-value range. Because the method receives a `float` value argument,
a Prefix changing that invocation argument does not by itself mutate the caller's weather state.

As floating-point expressions, the two operation orders need not be bit-identical at every value;
any difference is limited to normal floating-point rounding and is not a reason to choose the
candidate.

The candidate's problem is not ordinary weather-update correctness. It is live reapply.

## Immediate SettingChanged behavior

The accepted behavior requires a slider change to alter already-playing rain/wind immediately.

With the current hook, the mod already owns exactly the information needed for that operation:
the final native audio group and the last raw volume that the game sent to that group.

With an `UpdateWeatherVolume`-only patch, an immediate reapply instead needs enough state to call
the correct native weather owner again. The practical designs require one or more of:

- retaining active Rain/Wind `SmartWeatherState` instances;
- retaining their last unscaled weather values;
- identifying Rain versus Wind before the original method selects its audio ID;
- proving invalidation across scene/load/destruction lifecycle;
- adding reflection, per-instance caches, lifecycle patches, or discovery scans if those values
  are not available through a stable direct field seam.

A one-time scene/global scan on every setting change is broader and more fragile than the accepted
two-float cache. Keeping the general audio patch only as a live-reapply fallback while also
patching `UpdateWeatherVolume` creates two interception mechanisms and double-scaling/guarding
complexity. Neither is an improvement.

## Architecture comparison

| Mechanism | Frequency | Native owner preserved | Harmony breadth | Extra state | Live config behavior | Runtime cost | Complexity / fragility | Verdict |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Current filtered `SetSoundVolume` Prefix | Every call to the general setter; effect only on two exact IDs | Native weather computes final pre-mod volume and selects group | Broad call surface, narrow effect | Two raw float caches | Already accepted and immediate | Two cheap ID checks on unrelated calls; multiply/cache on Rain/Wind; reflection only on rare setting change | Low | **Keep** |
| `UpdateWeatherVolume` Prefix | Weather-volume updates only | Strongest semantic ownership for normal weather updates | Narrow | Needs additional state for immediate reapply | Not sufficient by itself | Lower theoretical interception count, but no measured user-relevant gain | Medium once live apply is preserved | Reject |
| Narrow hybrid | Weather path plus explicit live-reapply machinery | Mostly native | Narrow normal path | Weather refs/current values and lifecycle handling, or equivalent | Can be made immediate | Low steady-state, but more moving parts | Highest of the three for no proven benefit | Reject |

## Performance verdict

The current hook is broader than ideal by method ownership, but no evidence shows it to be a
performance problem.

The steady-state Prefix contains only exact string checks and returns without mutation for
unrelated groups. It performs no reflection or allocations of its own on that recurring path.
The exact total number of unrelated `SetSoundVolume` calls has not been runtime-instrumented, so
no numeric performance claim is made.

Likewise, no measured performance benefit exists for moving to `UpdateWeatherVolume`. Its
advantage is architectural/semantic, not demonstrated frame-time or allocation savings. Under the
project engineering rules, that is insufficient reason to replace already accepted code with a
stateful lifecycle solution.

No runtime harness is required to close this decision. A runtime frequency probe would only be
needed if future evidence suggests the two string checks on the general setter are materially hot.

## Rain/Wind independence

Production remains independent by exact native audio group:

- `rain_environment` uses only Rain Volume;
- `wind_environment` uses only Wind Volume;
- unrelated groups are returned unchanged.

Using weather-type state inside `SmartWeatherState` could be semantically cleaner, but adding
reflection or new state only to avoid two accepted native IDs would increase coupling without
solving a functional problem.

## Save and lifecycle safety

The mod stores user configuration only in BepInEx config. It does not write custom game-save data
or persist weather state. The two cached floats and reflected method handles are process runtime
state only.

Removing the plugin and restarting therefore returns weather audio handling to vanilla. No
uninstall save migration is required.

## Rejected alternatives

Do not reopen the same architecture question merely because `SetSoundVolume` is a general audio
API. Reconsider only if new evidence establishes at least one of these:

- the Prefix has measurable material cost on the real game's audio workload;
- another system uses the exact Rain/Wind group IDs with semantics that make the current scaling
  incorrect;
- Graveyard Keeper exposes a stable, state-free weather-local seam that preserves immediate
  `SettingChanged` behavior without scans, polling, instance tracking, or duplicated host logic;
- a real lifecycle/save/scene bug is reproduced in the current accepted implementation.

Until then, `SmartAudioEngine.SetSoundVolume` with exact Rain/Wind filtering is the accepted
least-sufficient production mechanism.

## Validation status

- Static/repository: accepted baseline, current source, current config/event lifecycle, and
  production hot-path behavior rechecked.
- Prior runtime evidence: accepted rain and wind live adjustment behavior remains the gameplay
  baseline; installed-game reflection confirms the relevant `SmartWeatherState` methods exist.
- New automated test: not required; no production behavior changed.
- New in-game acceptance test: not required; no production behavior changed.
- New artifact/release: none.
