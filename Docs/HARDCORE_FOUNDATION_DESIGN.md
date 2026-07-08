# Hardcore Foundation Design

Design date: 2026-07-07. Branch: `fable-hardcore-foundation`. Read-only design pass; no source, assets, or settings modified. Companion docs: `HARDCORE_SYSTEMS_SOURCE_MAP.md` (source anchors), `VERTICAL_SLICE_ROADMAP.md` (first playable slice), plus the three `U3_*` audit docs.

**Confidence labels:** `[SOURCE]` read in this repo (file:line), `[LOCAL]` local environment, `[INFER]` inference, `[RUNTIME]` needs Play Mode/build verification.

**Locked decisions (user-approved 2026-07-07):** slice map = **PEI**; code home = **hybrid** (develop in-repo under a new `Assets/Runtime/Assembly-CSharp/Unturned/Hardcore/` folder, with a documented extraction path to a `Builds/Shared/Modules` module once APIs stabilize); debug overlay = **IMGUI `OnGUI()`**.

---

## 1. Pitch

A hardcore realism total-conversion of Unturned built *on top of* the U3-SDK's existing survival simulation rather than beside it: Tarkov-style raid tension, STALKER-style environmental hostility, and Zomboid-style slow-burn body simulation, delivered as a layered "hardcore state" that first *observes* the vanilla vitals/damage/noise/loot systems (all of which already exist in source — bleeding, fractures, infection, gun jamming, suppressor wear, armor, radiation deadzones, noise-driven zombie alerts, per-region loot respawn), then gradually *takes authority over* them through the game's own sanctioned seams (damage-request events, consume events, difficulty config, .dat-driven content) — never by rewriting protected protocol, save, or manager internals.

## 2. Design pillars

1. **The body is the inventory that matters.** Health is not a bar; it is limbs, wounds, fluids, and time. Every fight should be won or lost twice: once in the exchange, once in the aftermath triage.
2. **Noise is currency.** Every action has an acoustic price and the world (zombie pressure) collects. The suppressor/jam/wear loop that already exists in source (`UseableGun.cs:936-953` `[SOURCE]`) becomes a first-class economy.
3. **Scarcity creates stories.** Loot is sparse, located where fiction says it should be, and worth the trip. Finding a full magazine should feel like an event.
4. **The loop is leave–risk–return.** Under-equipped exit from a safe point, escalating risk with time and noise, and a deliberate decision about when to bank what you carry.
5. **Systems over scripts.** Everything emergent, data-driven (.dat/config) where the engine already supports it; no bespoke one-off scripted encounters.
6. **Respect the host game.** Server-authoritative, plugin-compatible, save-compatible. Vanilla behavior is the default; hardcore is a layer that can always be switched off.

## 3. Anti-pillars (what this project is not)

- **Not a UI reskin.** No pretending depth via HUD widgets over unchanged vanilla numbers (the foundation overlay is a *dev tool*, not the product).
- **Not PvP-first.** Tarkov's extraction tension without Tarkov's PvP; co-op PvE is the eventual social mode. No server-browser presence, no retail-server interaction — off-limits per risk register #16 `[SOURCE: Docs/U3_RISK_REGISTER.md]`.
- **Not a content mod.** Success is measured in systemic behavior, not new item count. New items only exist to serve systems (medical, ammo families).
- **Not grind/meta progression.** No XP-gated body mechanics, no season passes, no daily quests. Skill mastery may be *reduced* in influence, never expanded into a treadmill.
- **Not a fork-of-everything.** We do not touch: net protocol, save formats, Steam identity, public plugin API surface, `Provider.cs` boot order, `ZombieManager` internals (audit §16 `[SOURCE]`).
- **Not fake difficulty.** No bullet-sponge zombies or RNG deaths without telegraphs; hardship must be legible and player-attributable.

## 4. Dream systems by category

Each entry names the existing source seam it will eventually attach to. Details per system: §10 and `HARDCORE_SYSTEMS_SOURCE_MAP.md`.

### A. Body & medical
- Body-region trauma → per-`ELimb` hit data already flows through `DamageTool.damagePlayer` / `PlayerLife.onHurt` `[SOURCE]`; organ simulation later as an extension of region states.
- Bleeding, pain, infection, fractures, shock → vanilla already has bleeding/broken-legs/virus (`PlayerLife`); pain and shock are new derived states.
- Medical treatment items → `ItemMedicalAsset`/`ItemConsumeableAsset` vocabulary (health, bleeding modifier, bones modifier, disinfectant) `[SOURCE: ItemConsumeableAsset.cs:263-300]`; deepened via new .dat items + consume-event handlers.

### B. Metabolism
- Nutrition, hydration, contamination, micronutrients → vanilla food/water/virus bytes plus food-quality spoilage (`quality < 50` messaging exists, `ItemConsumeableAsset.cs:228`) `[SOURCE]`; micronutrients are a new shadow ledger.
- Temperature/exposure → `EPlayerTemperature` (FREEZING…ACID), `warmth`, `inRain`/`inSnow` `[SOURCE]`.

### C. Weapons & ammo
- Ammo families, real calibers, subtypes → caliber ushort IDs already join guns↔magazines↔attachments (`ItemCaliberAsset.calibers`, inventory caliber search) `[SOURCE]`; realism = new .dat content + tuning, minimal code.
- Weapon condition & jams → `Can_Ever_Jam`/`Jam_Quality_Threshold`/`Jam_Max_Chance` + quality-driven spread/recoil/damage `[SOURCE: ItemGunAsset.cs:825-856, UseableGun.cs:991-1050,1345-1357,1619]`.
- Suppressor wear → barrel attachment quality (gun state byte 16) already gates silencing and degrades `[SOURCE: UseableGun.cs:936, ItemBarrelAsset.cs:21-27]`.

### D. Equipment & encumbrance
- Armor/clothing protection & degradation → per-slot clothing quality + per-limb armor lookup `[SOURCE: DamageTool.getPlayerArmor:483, ItemClothingAsset.cs:11-41]`.
- Carry weight, fatigue, movement penalties → new weight ledger over `PlayerInventory`; applied later via the existing `pluginSpeedMultiplier` server API `[SOURCE: PlayerMovement.cs:122,649-666]`.

### E. World pressure
- Noise-driven zombie pressure → `AlertTool.alert(position, radius)` is the sanctioned stimulus entry `[SOURCE: AlertTool.cs:218]`.
- Weather/exposure/radiation/contamination → deadzones with mask/filter simulation already exist (`PlayerLife.simulate` radiation block) `[SOURCE: PlayerLife.cs:2088-2143]`; weather events on `LightingManager` `[SOURCE]`.

### F. Economy of stuff
- Loot scarcity & location-based scavenging → per-region spawn tables + `Spawn_Chance`/`Respawn_Time` config `[SOURCE: ItemManager.cs:764-830, PlayConfigData.cs:481]`.
- Repair/crafting/tools → blueprint system with request veto + crafting tags/workstations `[SOURCE: PlayerCrafting.cs:78-90]`.
- Stash/extraction/safehouse loop → new loop logic; leans on existing volumes/claims/storage later; slice 1 uses a local extraction check (see roadmap).

### G. Social
- Co-op PvE → existing dedicated-server/authority architecture; strictly a later phase (risk register #3, #5) `[SOURCE]`.

## 5. First foundation prototype — "Hardcore Shadow State"

A **local-only, non-persistent, non-networked, default-off, debug-visible** observation layer:

- **Local-only:** created only when `!Dedicator.IsDedicatedServer` and a local flag is enabled; no behavior change for any other player or mode.
- **Non-persistent:** lives on a runtime GameObject; all state is rebuilt from live reads; discarded on level exit/death; writes no files (not even under `Builds/Shared`).
- **Non-networked:** never calls `Send*`/`ask*`/`tell*`/RPC methods; only subscribes to client-visible events and reads statics.
- **Default-off:** enabled via the existing `CommandLineFlag` pattern (editor prefs expose flags via Window > Unturned > Editor Settings `[SOURCE: Setup.cs:27, EditorSettingsTool]`) — e.g. `-HardcoreFoundation` — plus an in-session toggle key. With the flag off, the only cost is one boot-time check.
- **Debug-visible:** a single IMGUI `OnGUI()` overlay (audit §12 ranked this the safest seam `[SOURCE: Docs/U3_ARCHITECTURE_AUDIT.md]`) showing every shadow state live, updated at ~4 Hz with cached strings (risk register #11–13).

What it does: mirrors the vanilla player into hardcore-shaped state objects (below), derives *display-only* interpretations (per-limb trauma accumulation, estimated noise footprint, nutrition trajectory), and proves we can read every value the future systems need — before any of them writes anything.

## 6. Proposed internal architecture

All new code in `Assets/Runtime/Assembly-CSharp/Unturned/Hardcore/` (new folder, new `.meta`s committed together; hybrid plan: keep types `internal`-ish in spirit — no promises to plugins — so extraction to a module later stays possible). Bootstrapping should need **0 existing files modified**: a `[RuntimeInitializeOnLoadMethod]` static initializer that checks the flag and subscribes to `Level.onLevelLoaded` `[INFER: standard Unity API; verify ordering vs `Setup.Awake` in Play Mode → RUNTIME]`. Fallback if ordering misbehaves: one-line hook in `Setup.cs` (1 existing file, within budget).

| Type | Role | Reads (never writes) |
|---|---|---|
| `HardcoreRuntime` | Static entry + MonoBehaviour host. Flag check, lifecycle (create on `Level.onLevelLoaded` when client + flag; teardown on level exit), owns update cadence (low-Hz sampling, not per-frame). | `Dedicator.IsDedicatedServer`, `Provider.isServer/isClient`, `Level.info`, `Player.LocalPlayer` (guarded `[SOURCE: Player.cs:226-239]`) |
| `HardcorePlayerState` | Aggregates the sub-states for the local player; rebuilt on spawn/revive; exposes one read API for the overlay and future systems. | `PlayerLife` events (`onHealthUpdated`, `onFoodUpdated`, `onWaterUpdated`, `onVirusUpdated`, `onStaminaUpdated`, `onBleedingUpdated`, `onBrokenUpdated`, `onTemperatureUpdated`, `onHurt`, static `onPlayerDied`) `[SOURCE: PlayerLife.cs:82-94,50]` |
| `BodyRegionState` | Shadow per-region ledger keyed on `ELimb` (skull/spine/arms/legs…): accumulated damage, last-hit cause/time, derived flags (e.g. "legs compromised" when `isBroken`). Display-only. | `onHurt(Player, byte damage, …, EDeathCause, ELimb, CSteamID)` `[SOURCE: PlayerLife.cs:26]`; `isBroken`, `isBleeding` |
| `ConditionState` | Cross-cutting conditions: bleeding, fracture, infection trend (virus delta), temperature band, oxygen, radiation exposure (in-deadzone + mask/filter status). | `PlayerLife` stats; `player.movement.isRadiated`, `ActiveDeadzone`; `player.clothing.maskAsset/maskQuality` `[SOURCE: PlayerLife.cs:2088-2120]` |
| `NutritionState` | Food/water/virus values + rates (estimated from `PlayersConfigData` tick constants `[SOURCE: PlayerLife.cs:1953-2013, PlayConfigData.cs:1450]`), projected time-to-starve/dehydrate; future home of micronutrients/contamination ledger. | `food`, `water`, `virus`, `Provider.modeConfigData.Players.*` |
| `NoiseTelemetryState` | Estimated current acoustic footprint: equipped weapon `alertRadius`, silenced-or-not (barrel asset + state byte 16), stance (`EPlayerStance`), movement speed multipliers; later enriched with real shot events. | `player.equipment.asset as ItemGunAsset` (`alertRadius`), `thirdAttachments`/state `[SOURCE: UseableGun.cs:936-939]`, `player.stance.stance`, `PlayerMovement.totalSpeedMultiplier` |
| `HardcoreDebugOverlay` | MonoBehaviour with `OnGUI()`; panels per sub-state + world context (zombie count via `ZombieManager.tickingZombies.Count` `[SOURCE: CommandDebug.cs:28]`, day/night via `LightingManager.isDaytime`, weather flags). Cached strings, ~4 Hz refresh, zero per-frame allocation target. | everything above |

## 7. What the foundation reads from existing systems

- **Vitals & events:** all `PlayerLife` stats (health/food/water/virus/stamina/oxygen/warmth/vision), bleeding/broken flags, temperature enum, and their update delegates `[SOURCE: PlayerLife.cs:82-94,154-194]`.
- **Hit context:** `onHurt` gives damage, cause, limb, killer per hit `[SOURCE: PlayerLife.cs:26,94]`.
- **Movement/stance:** `EPlayerStance` (8 values `[SOURCE: EPlayerStance.cs]`), `isGrounded`, `nav`, `isSafe`, `isRadiated`, `inRain/inSnow`, speed multipliers `[SOURCE: PlayerMovement.cs]`.
- **Equipment/clothing:** equipped `ItemAsset`/quality/state bytes (attachment qualities at indices 13–17 `[SOURCE: UseableGun.cs:36-40]`), 7 clothing slots with per-slot asset/quality, armor/proof/movement-speed fields `[SOURCE: PlayerClothing.cs:103-131, ItemClothingAsset.cs]`.
- **Inventory:** page grids, `ItemJar`s, add/remove events `[SOURCE: PlayerInventory.cs:140-196]`.
- **World:** `Level.info` (name/type — note vitals only tick on `ELevelType.SURVIVAL` `[SOURCE: PlayerLife.cs:1948]`), `LightingManager` time/weather events, `ZombieManager` counts.
- **Config:** `Provider.modeConfigData.Players/Items/Zombies` tunables (read for display/projection only) `[SOURCE: PlayConfigData.cs:437-1450]`.

## 8. What it must not write to (yet)

- Any `PlayerLife` mutator: `askDamage/askStarve/askEat/serverModify*/serverSet*` — gameplay authority comes later, server-side, behind its own approvals.
- `player.equipment.state`/`quality`, inventory contents, clothing quality — item mutation touches replicated + saved state (risk #4, #5).
- Any `Send*/tell*/ask*` RPC or anything in `NetGen/`, `NetMessaging/`, `NetInvokable/`.
- Any file under `Builds/Shared` (saves, `Config.json`, `Preferences.json`) or any `ReadWrite`/`Block`/`River` API.
- `AlertTool.alert(...)` — emitting stimuli is Experiment-3 territory (audit §17), not foundation.
- Static game state (no new statics with init-order dependencies — risk #2), difficulty config objects, and anything listed in audit §16.

## 9. Transient-only at first

- All shadow ledgers (`BodyRegionState` accumulations, nutrition trends, noise history ring buffer) — reset on death/revive/level load, never serialized.
- Toggle state may live in `EditorPrefs`-backed `CommandLineFlag` (editor convenience only `[SOURCE: CommandLineFlag.applyEditorPreferencesToAllFlags]`); no new preference files.
- Derived interpretations (pain/shock estimates) are labels over live data, recomputed, never stored.
- Rationale: zero save-compat exposure (risk #4), instant rollback, and honest iteration — if a value can't be derived live, we learn that *now* rather than corrupting a format later.

## 10. How each dream system plugs into this foundation later

| Dream system | Foundation attachment point | Sanctioned write seam (later, server-side, with approval) |
|---|---|---|
| Body-region trauma → organs | `BodyRegionState` becomes authoritative ledger | `DamageTool.damagePlayerRequested` (ref params: rescale per-limb damage before it applies) `[SOURCE: DamageTool.cs:133-138]` |
| Bleeding/pain/infection/fractures/shock | `ConditionState` gains new conditions | `serverSetBleeding/serverSetLegsBroken/serverModifyVirus` + `DamagePlayerParameters.bleedingModifier/bonesModifier` `[SOURCE: DamageTool.cs:378-421]` |
| Medical items | consumption observed via `UseableConsumeable` events | `ConsumeRequestedHandler`/`PerformingAidHandler` (veto/redirect) + new `ItemMedicalAsset` .dat content `[SOURCE: UseableConsumeable.cs:14-35]` |
| Nutrition/micronutrients/contamination | `NutritionState` ledger | `serverModifyFood/Water/Virus`; food spoilage via item quality `[SOURCE: PlayerLife.cs:1503-1553]` |
| Ammo families/calibers/subtypes | `NoiseTelemetryState` + equipment reads | new .dat: `ItemMagazineAsset` (damage/speed/pellets), caliber IDs; no code for data-only families `[SOURCE: ItemMagazineAsset.cs]` |
| Weapon condition/jams | overlay already shows quality | tune `Durability/Wear/Jam_*` in .dat; jam logic exists `[SOURCE: UseableGun.cs:1345-1357]` |
| Suppressor wear | silenced/wear display | barrel `Durability` .dat values; state byte 16 decrement exists `[SOURCE: ItemBarrelAsset.cs:27; decrement site INFER → verify]` |
| Armor/clothing degradation | clothing panel in overlay | armor .dat + `getPlayerArmor` inputs; quality decrement APIs exist per slot `[SOURCE: PlayerClothing.cs:173-312]` |
| Carry weight/fatigue | new weight sum in `HardcorePlayerState` (pure read) | `sendPluginSpeedMultiplier` (existing plugin-facing server API) `[SOURCE: PlayerMovement.cs:649-666]` |
| Noise-driven zombie pressure | `NoiseTelemetryState` + alert observation | `AlertTool.alert(Vector3, float)` emissions, rate-limited `[SOURCE: AlertTool.cs:218]` |
| Loot scarcity/scavenging | loot census tooling (read-only) | `ItemsConfigData` (`Spawn_Chance`, `Respawn_Time`, `Despawn_*`) via difficulty config data; spawn-table .dat `[SOURCE: ItemManager.cs:745-779]` |
| Stash/extraction/safehouse | run-clock + zone checks in `HardcoreRuntime` | vanilla per-player save already persists inventory (no new writes); later: storage/claims |
| Repair/crafting/tools | blueprint availability display | `onCraftingRequested` veto + blueprint .dat + crafting tags `[SOURCE: PlayerCrafting.cs:78]` |
| Weather/exposure/radiation | `ConditionState` bands | `serverModifyWarmth`, deadzone/weather .dat/level data; `WeatherEventHook` mod hooks `[SOURCE]` |
| Co-op PvE | none (foundation is client-local) | entire layer re-hosted server-side behind `Provider.isServer`; foundation's read API designed to be role-agnostic from day 1 |

## 11. Risks (from `Docs/U3_RISK_REGISTER.md`)

| Register # | Relevance to foundation | Mitigation baked into this design |
|---|---|---|
| #1 boot order | bootstrap timing vs `Setup.Awake` | subscribe to `Level.onLevelLoaded` only; `[RuntimeInitializeOnLoadMethod]` ordering verified in Play Mode before trusting `[RUNTIME]` |
| #2 static singletons | our own statics could leak across level loads | single runtime object, explicit teardown on `onLevelExited`; no init-order-dependent statics |
| #3 authority mistakes | `Player.LocalPlayer` throws on dedicated server | every entry guarded by `!Dedicator.IsDedicatedServer`; tested with `-DedicatedServerInEditor` |
| #4 save formats | none while read-only | hard rule §8; no `ReadWrite` usage at all |
| #5/#6 protocol/enums | none while read-only | no RPCs, no enum changes; overlay reads enums by value only |
| #7 plugin API | new public types become de-facto API | keep foundation types un-promised (docs state "unstable"); extraction-to-module path preserves this |
| #11–13 perf | `OnGUI` + event handlers on hot paths | 4 Hz sampling, cached strings, no LINQ/allocs in handlers, no `FindObjectsOfType` |
| #15 editor-API leaks | flag tooling uses `EditorPrefs` | mirror existing `#if UNITY_EDITOR` guard patterns; Build Test (Scripts Only) after every change |
| #17 meta/GUID | new folder + new files | new `.meta`s committed with files; nothing moved/renamed |
| #20 Provider god-class | temptation to hook boot | we don't touch `Provider.cs`; reads via public statics only |

## 12. Definition of done (foundation)

1. With the flag **off** (default): behavior byte-identical to vanilla; no overlay, no logs beyond one boot line, `git status` clean except the new `Hardcore/` folder.
2. With the flag **on**, in editor Play Mode on PEI (singleplayer): overlay renders all six state groups with live values; vitals panel tracks eating/drinking/damage within one refresh tick; limb panel registers hits with correct `ELimb`; noise panel changes when equipping/silencing a gun and when switching stance.
3. Same checks pass in a **Build Test** standalone run (proves no editor-API leaks).
4. On a `-DedicatedServerInEditor` run, nothing is created and nothing throws.
5. Zero writes verified: no new/modified files under `Builds/Shared` after a session; no `Send*/tell*/ask*` call sites in the new code (grep-audit).
6. Death → respawn and level reload rebuild state cleanly (no stale ledgers, no duplicate subscriptions).
7. Profiler spot-check: no measurable per-frame GC alloc from the overlay at 4 Hz refresh.
8. Rollback demonstrated: deleting `Unturned/Hardcore/` (and reverting the ≤1 hook line if the fallback was needed) restores a clean tree.
