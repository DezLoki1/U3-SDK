# Vertical Slice Roadmap — "First Raid"

Design date: 2026-07-07. Branch: `fable-hardcore-foundation`. Companions: `HARDCORE_FOUNDATION_DESIGN.md` (architecture), `HARDCORE_SYSTEMS_SOURCE_MAP.md` (source anchors). Confidence labels as usual (`[SOURCE]`/`[LOCAL]`/`[INFER]`/`[RUNTIME]`).

**Slice definition:** a 20–30 minute hardcore PvE survival loop on **PEI** (user-approved test target; vanilla map loaded read-only from the Steam install `[SOURCE: Level.cs:1137-1141]`): spawn under-equipped, scavenge, manage wounds/food/water/stamina/noise, avoid or fight zombies, return to an extraction point, keep what you carried out.

Guiding constraint: the slice is assembled almost entirely from systems that **already exist in source** (vitals, bleeding, fractures, jamming, suppressors, armor, noise alerts, loot respawn, per-player saves) plus the local-only hardcore foundation layer. Nothing in slice 1 writes saves, sends new network messages, or changes protected systems.

---

## 1. First 30 minutes, step by step

Times assume default walk speeds; verify pacing in Play Mode `[RUNTIME]`. Location names from vanilla PEI `[LOCAL: retail knowledge — confirm in-game]`.

| T | Beat | Systems exercised |
|---|---|---|
| 0:00 | Load PEI singleplayer; hardcore flag on. Overlay confirms hardcore session + run clock starts. Spawn on the coast with vanilla starting nothing (under-equipped by default). | `Level.onLevelLoaded`, foundation boot, `OnSelectingRespawnPoint` (observed only) |
| 0:02 | Read the overlay: 100/100 vitals, projected time-to-hunger/thirst from config ticks. Pick nearest town (e.g. a coastal settlement) and move; sprint drains stamina visibly. | `NutritionState` projections, stamina (`SimulateStaminaFrame`) |
| 0:05 | First structures: loot food/water/melee. Item pickups appear in overlay inventory census; each item's quality is visible (food <50% flagged as risky). | `ItemManager` regions, `ItemJar.quality`, spoilage messaging `[SOURCE: ItemConsumeableAsset.cs:228]` |
| 0:08 | First zombie contact. Choice: sneak around (crouch — smaller detect radius via stealth/detect config) or engage with melee (silent) vs found pistol (loud: overlay shows its `alertRadius`, e.g. 48 m default). | `AlertTool` vision path, `EPlayerStance.CROUCH`, `NoiseTelemetryState` |
| 0:12 | Take a hit: overlay's limb panel logs the hit (`ELimb`), bleeding may start. Bandage or bleed 1 HP per bleed tick — triage under pressure. | `onHurt`, `isBleeding`, consumable `Bleeding_Modifier: Heal` |
| 0:15 | Deeper push to a POI with better tables (police/military presence on PEI): gun, magazines matching caliber, maybe a suppressor. Overlay shows caliber compatibility of found mags vs carried gun. | caliber search `[SOURCE: PlayerInventory.cs:2087]`, spawn tables |
| 0:18 | Fire the gun: noise event — zombies in radius converge (reaction, not clean pathing — ASPFP stubbed, expected `[SOURCE: UnturnedPathfinding_Empty]`). Weapon quality ticks down with use; below the jam threshold, jams become possible. | `AlertTool.alert(position, radius)`, wear (`:941-953`), jam (`:1345`) |
| 0:20 | Consequence management: eat/drink (watch virus from dirty items), disinfect, splint-equivalent (vanilla leg regen is config-gated), decide whether stamina/food budget allows more looting. | `askEat/askDrink/askInfect` effects (observed), `Can_Fix_Legs` config |
| 0:24 | Return leg: heavier decision-making — night may be approaching (`LightingManager.isDaytime`), zombies rebuilt aggro. Route back to the marked **extraction zone** (a coordinate + radius the slice defines near spawn). | day/night, extraction check (new, local) |
| 0:27 | Stand in the zone for a dwell time (e.g. 10 s) with no zombies aggroed → run summary overlay: duration, items gained, shots fired, noise emitted, damage log, vitals trend. | run summary (new, local) |
| 0:30 | Quit to menu. Loot persists because vanilla per-player save already does that — no new persistence code. | vanilla `PlayerSavedata` (untouched) |

Death at any point = run summary with "KIA", vanilla death/respawn flow untouched.

## 2. Slice 1 features (only these)

1. **Hardcore foundation layer** (design doc §5–6): runtime, shadow states, IMGUI overlay, default-off flag.
2. **Run clock + extraction zone + run summary** — local-only, display-only; zone is data in code/config local to the feature (no map edits).
3. **Noise telemetry** — estimated footprint + real fire events, silenced/unsilenced awareness, zombie count deltas.
4. **Limb hit ledger** — per-`ELimb` shadow trauma from `onHurt` (display-only).
5. **Scarcity preset (data-only)** — a documented, locally-applied difficulty config for testing (reduced `Spawn_Chance`, longer `Respawn_Time`, harsher food/water ticks, `ShouldWeaponTakeDamage` on, bleeding frequent). Applied via the game's own difficulty config data — **values documented in-repo, applied to local save-dir config by hand, never committed as changed game defaults** `[SOURCE: PlayConfigData.cs; mode config loaded in Provider.awake]`.
6. **Vanilla systems used as-is:** bleeding/fractures/infection, jamming/wear/suppressors, armor, loot respawn, day/night, stamina, saves.

## 3. Explicitly out of scope for slice 1

- Writing anything: no save-format additions, no new files under `Builds/Shared`, no config files written by code.
- New network messages/RPCs/replicated state; any co-op/multiplayer feature.
- Gameplay mutation by hardcore code: no damage rescaling, no speed penalties, no forced jams, no emitted `AlertTool` stimuli (branch 5 begins the first sanctioned exception, see below).
- Organ simulation, pain/shock *effects*, micronutrients, contamination beyond vanilla virus.
- Custom items/.dat content, custom maps or map edits, ammo family rework.
- Stash/safehouse persistence, in-raid menus, Sleek/player-facing HUD.
- Zombie behavior/count/spawn changes; pathfinding fixes (licensed plugin absent).
- Any UI beyond the debug overlay + run summary panel.

## 4–5. First five implementation branches

Common rollback ground rule (all branches): all logic in new files under `Assets/Runtime/Assembly-CSharp/Unturned/Hardcore/` (+ their `.meta`s); `git status` must show only expected paths; flag-off must equal vanilla; revert = delete new files + revert the explicitly listed touched files. Never `git add -A` (CLAUDE.md rule 9). Each branch ends with the audit's testing ladder: editor Play Mode → Build Test → `-DedicatedServerInEditor` no-op check `[SOURCE: Docs/U3_RISK_REGISTER.md ladder]`.

### Branch 1 — `hardcore/foundation-overlay`
- **Gameplay goal:** none yet — trustworthy eyes. HardcoreRuntime + HardcorePlayerState + ConditionState + NutritionState + overlay showing vitals/stance/equipment/world panels live.
- **Inspect first:** `PlayerLife.cs` delegates (`:82-94`), `Player.cs:226-247`, `PlayerDashboardInformationUI.cs` (read patterns), `CommandDebug.cs`, `CommandLineFlag` + `EditorSettingsTool.cs`, `Setup.cs:21-54`.
- **Max existing files modified:** **1** (`Setup.cs` one-line bootstrap, only if `[RuntimeInitializeOnLoadMethod]` ordering fails `[RUNTIME]`); target 0.
- **Protected/avoid:** `Provider.cs`, all managers, `PlayerUI.cs`, anything in §16 of the audit.
- **Play Mode test:** `GameStartup.unity` → Play → PEI singleplayer → flag off: no overlay → flag on + toggle key: panels live → take fall damage, eat, sprint: values update ≤1 refresh tick → die/respawn: state resets, no duplicate handlers (toggle twice, watch for double lines).
- **Build Test:** Build Test → run exe → same checks; then `-DedicatedServerInEditor`: boot completes, zero hardcore objects, no `LocalPlayer` exception.
- **Rollback:** delete `Unturned/Hardcore/` + metas; revert `Setup.cs` if touched.

### Branch 2 — `hardcore/noise-telemetry`
- **Gameplay goal:** make noise legible: per-shot log (position, weapon, `alertRadius`, silenced?, zombies-in-radius before/after), ambient footprint estimate from stance/speed/equipped weapon.
- **Inspect first:** `UseableGun.cs:905-955`, `AlertTool.cs:55-270`, `UseableGunEventHook.cs`, `ZombieManager.getZombiesInRadius` (reuse its preallocated-list pattern), `RuntimeGizmos.cs`, `ItemBarrelAsset.cs`.
- **Max existing files modified:** **2** (ideally 0: poll equipped state + subscribe to existing seams; `UseableGun.cs` only if no hook reaches the fire moment, and then a single event-raise line, no reordering).
- **Protected/avoid:** the `Provider.isServer` fire block's logic/order, `SendPlayShoot`, rechamber bookkeeping (`UseableGun.cs:925-928` warning), `Zombie`/`ZombieManager` internals, committing `LOG_ALERTS`-style define changes.
- **Play Mode test:** spawn pistol + suppressor (sandbox/cheats) → fire unsilenced: log entry with radius, zombie delta > 0 nearby → fit suppressor: silenced flag, no alert → suppressor quality to 0 (long fire session or spawn worn): alerts resume `[SOURCE: state[16]==0 gate]`.
- **Build Test:** same script standalone; dedicated-server editor run stays silent (telemetry is client-side observation; the alert itself is server logic — in singleplayer both roles coexist, verify log context `[RUNTIME]`).
- **Rollback:** delete new files; revert any single-line hook.

### Branch 3 — `hardcore/limb-ledger`
- **Gameplay goal:** body-region shadow ledger: every hit binned by `ELimb` with cause/time/damage; overlay body panel; foundation for future trauma authority.
- **Inspect first:** `PlayerLife.cs:26,94` (`onHurt`), `DamageTool.cs:133-145,332-443`, `ELimb.cs`, `DamagePlayerParameters.cs`, `PlayerDamageMultiplier.cs`.
- **Max existing files modified:** **0** (subscribe to `PlayerLife.onHurt` + static `DamageTool.damagePlayerRequested` with `shouldAllow` untouched).
- **Protected/avoid:** never write to `ref parameters`/`ref shouldAllow` (that's a later, approved gameplay change); `DamageTool` internals; NetGen damage files.
- **Play Mode test:** let a zombie hit you (melee = arms/spine variety), fall (legs), headshot yourself with low-damage weapon impossible — instead verify limb variety via zombie hits + falls; panel bins match `EDeathCause`/`ELimb` logs; bleeding onset reflected.
- **Build Test:** standalone repeat; dedicated no-op check.
- **Rollback:** delete new files; nothing else touched.

### Branch 4 — `hardcore/raid-loop`
- **Gameplay goal:** the loop skeleton: run clock from spawn, extraction zone (coordinate + radius + dwell timer on PEI near spawn coast), end-of-run summary (duration, hits taken by limb, shots/noise, items delta, vitals trend). Local, display-only; "keep loot" = vanilla save, untouched.
- **Inspect first:** `Level.onLevelLoaded`/`onLevelExited` (`Level.cs`), `PlayerLife` death events (`onPlayerDied`), `PlayerInventory` add/remove events (items delta), `LightingManager` (time context), `PlayerMovement.real` position reads.
- **Max existing files modified:** **0**.
- **Protected/avoid:** no barricade/structure/claim/volume placement (those are replicated + saved); no teleporting the player; no scene edits for zone markers — draw the zone with `RuntimeGizmos`/overlay instead.
- **Play Mode test:** full 20–30 min loop per §1 script; extraction dwell completes only inside radius; summary numbers reconcile with overlay logs; death path shows KIA summary; quit/reload PEI → loot persisted by vanilla save.
- **Build Test:** one full loop standalone (this is the slice acceptance run); dedicated no-op check.
- **Rollback:** delete new files.

### Branch 5 — `hardcore/scarcity-preset`
- **Gameplay goal:** the hardcore *feel* pass, data-first: a documented difficulty preset (loot down, metabolism harsher, weapon wear on, bleeding meaningful) + a read-only **loot census** tool (per-region item counts, `LogAllSpawnTables()` dump) to measure before/after.
- **Inspect first:** `PlayConfigData.cs:437-1450+` (`ModeConfigData`, `ItemsConfigData`, `PlayersConfigData`, `ZombiesConfigData`), `Provider.awake` config load path (`Provider.cs:6729-6736` per audit), `SpawnTableTool.cs:719`, `ItemManager.cs:734-830`.
- **Max existing files modified:** **0 code files.** The preset ships as a **documented values table in `Docs/`** applied to the local difficulty config in the save directory by hand (`[RUNTIME]`: confirm exact file location/precedence for singleplayer before writing the how-to). No committed default changes, no code that writes config.
- **Protected/avoid:** committing changed defaults into `PlayConfigData.cs` or tracked JSON; `Status.json`; anything altering server browser/mode identity (risk #16).
- **Play Mode test:** census on default PEI → apply preset locally → census again: measurable scarcity; food/water pressure noticeable within a 25-min run; gun quality degrades and jams appear on worn guns.
- **Build Test:** same preset against the standalone build's config root (`ReadWrite.PATH` differs in builds — verify `[SOURCE: UnturnedPaths.cs:17-29]` `[RUNTIME]`).
- **Rollback:** delete local config copy; docs are inert.

## 6. Definition of done — first playable slice

1. One sitting, editor **and** Build Test standalone: a tester who reads a half-page how-to can run the §1 loop end-to-end on PEI in 20–30 min with the flag on.
2. The loop produces at least three legible hardcore decisions per run (engage-vs-sneak given noise, triage under bleeding, push-vs-extract given vitals/time) — validated by playtest notes, not vibes.
3. Overlay + run summary accurately reflect: vitals, limb hits, noise events (silenced vs not), items gained/lost, run duration, and death cause on failure.
4. Flag off ⇒ demonstrably vanilla (no overlay, no logs, no perf delta beyond one boot check); dedicated-server editor run unaffected.
5. Zero writes: `git status` clean except intended new files; no new/changed files under `Builds/Shared` created *by code*; no new RPC call sites (grep audit for `Send|tell|ask` in `Hardcore/`).
6. No red flags from the risk register triggered without sign-off (no NetGen, saves, enums, `.meta`/scene edits, `Provider.cs` restructuring; ≤ the per-branch existing-file budgets above, total across slice ≤5 per CLAUDE.md rule 1 accounting per feature).
7. Perf: no visible hitching from hardcore code on PEI (spot Profiler check; overlay ≤4 Hz refresh, no per-frame allocs).
8. Rollback rehearsed once: flag off → delete `Hardcore/` → tree clean → Play Mode still vanilla-boots.

## 7. Parking lot (insane realism, later — each needs its own design + approval)

- **Body:** organ-level trauma (heart/lungs/liver per region), blood volume/pressure model, pain→tunnel-vision/aim sway coupling, shock states, wound infection distinct from vanilla virus, surgery kits, splint quality affecting gait, chronic conditions across raids (needs persistence design).
- **Metabolism:** micronutrient/vitamin ledger, caloric model replacing food byte, water contamination tiers + purification chain, cooking states (raw/cooked/burnt/spoiled via item quality), temperature-driven calorie burn, sleep/fatigue as a third resource.
- **Ballistics/weapons:** ammo subtypes per caliber (FMJ/HP/AP as magazine .dat families), chamber checks + press checks, malfunction taxonomy beyond jams (failure-to-feed vs stovepipe animations), barrel heat, zeroing, canted optics, per-part gun condition (bolt/spring wear), gunsmithing bench.
- **Equipment:** plate-based armor with plate damage states, layered clothing insulation model, load-bearing rigs vs backpacks with draw-speed effects, full carry-weight sim with stance/fall interactions, gear rattle affecting stealth radius.
- **World:** dynamic weather fronts with exposure (hypothermia chain via warmth/temperature), radiation storms (deadzone weather events), anomaly-like hazards via deadzone volume composition, noise-driven persistent horde migration between regions, darkness that matters (flashlight discipline vs `AlertTool` spot checks).
- **Loop/meta:** persistent safehouse with upgrade tree, stash tiers, insurance-free loss (true full-loot PvE), NPC trader with barter economy (NPC/quest system exists in source), dynamic extraction availability, co-op raids with shared medical interactions (`PerformingAidHandler` seam), hardcore character permadeath profiles (needs separate save namespace).
- **AI:** zombie hearing memory/investigation chains, special infected via `EZombieSpeciality` tuning, animal threat ecology — all contingent on accepting stubbed-pathfinding limits or a licensed ASPFP build.

---

*End of design pass. No source, assets, or settings were modified; the only changes are the three docs in `Docs/`.*
