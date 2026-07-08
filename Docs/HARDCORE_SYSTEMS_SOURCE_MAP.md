# Hardcore Systems Source Map

Recon date: 2026-07-07. Branch: `fable-hardcore-foundation`. Read-only pass over `Assets/Runtime/Assembly-CSharp/` (paths below are relative to that root unless noted). Confidence labels: `[SOURCE]` (read in this repo, file:line), `[LOCAL]`, `[INFER]`, `[RUNTIME]`. Line numbers drift as upstream changes land — treat them as anchors, re-grep before relying on them.

This pass independently re-verified the key claims inherited from `Docs/U3_ARCHITECTURE_AUDIT.md` that it builds on: `AlertTool.alert` overloads (`AlertTool.cs:55,218`), the `UseableGun` server block and silencer gate (`UseableGun.cs:912-954`), `Player.LocalPlayer` throwing on dedicated server (`Player.cs:226-239`), and the mod-hook folder contents. `[SOURCE]`

---

## 1. Player life / vitals / death

**Files/classes:** `Unturned/Player/PlayerLife.cs` (~2,560 lines), `EDeathCause.cs`, `EPlayerTemperature.cs`, `Unturned/Player/Player.cs` (module aggregation), `NetGen/NetInvokable/PlayerLife_NetMethods.cs` (protocol — protected).

**Confirmed `[SOURCE]`:**
- Stats are small scalars with private backing + read-only properties: `health/food/water/virus/vision/stamina/oxygen` (bytes), `warmth` (uint), `isBleeding`, `isBroken`, `temperature` (`EPlayerTemperature`: FREEZING, COLD, WARM, BURNING, NONE, COVERED, ACID) (`PlayerLife.cs:154-194`, `EPlayerTemperature.cs`).
- Per-stat update delegates on the instance (`onHealthUpdated`, `onFoodUpdated`, `onWaterUpdated`, `onVirusUpdated`, `onStaminaUpdated`, `onBleedingUpdated`, `onBrokenUpdated`, `onTemperatureUpdated`, `onDamaged`, `onHurt` with damage/cause/limb/killer) plus statics `onPlayerLifeUpdated`, `OnPreDeath`, `onPlayerDied`, `OnSelectingRespawnPoint` (ref position/yaw!), `OnFallDamageRequested` (ref damage, ref shouldBreakLegs — an official override hook) (`PlayerLife.cs:13-94, 79-80, 2390-2391`).
- Server mutators: `askDamage` (4 overloads → private `doDamage`), `askStarve/askEat/askDehydrate/askDrink/askInfect/askDisinfect/askTire/askRest/askSuffocate/askBreath/askRadiate`, `serverModifyHealth/Food/Water/Virus/Stamina/Warmth/Hallucination`, `serverSetBleeding`, `serverSetLegsBroken` (`PlayerLife.cs:530-1553`).
- `simulate(uint)` is the metabolic heartbeat: starvation/dehydration/infection only when `Provider.isServer` **and** `Level.info.type == ELevelType.SURVIVAL`; intervals from `Provider.modeConfigData.Players.{Food_Use_Ticks, Food_Damage_Ticks, Water_*, Virus_*, Bleed_*, Health_Regen_*, Leg_Regen_Ticks}` scaled by skills (`PlayerLife.cs:1944-2056`). Bleed-out and starvation damage arrive as `askDamage(1, …, EDeathCause.BLEEDING/FOOD/WATER, ELimb.SPINE, …)`.
- Stamina: sprint (or boost-driving) drains 1 per interval widened by EXERCISE skill; regen gated by CARDIO (`SimulateStaminaFrame`, `PlayerLife.cs:1794-1813`). Oxygen: `breathability` model with oxygen volumes and underwater-apparatus check on clothing proofs (`PlayerLife.cs:1830-1918`).
- Radiation: `player.movement.isRadiated` + `ActiveDeadzone` (types incl. `FullSuitRadiation`), mask+filter quality degradation per second constants documented in-code (`PlayerLife.cs:2088-2143`).
- Death → `doDamage` sends `SendDeath`/`SendDead` RPCs, aggro bookkeeping, respawn selection (`PlayerLife.cs:637-709`); god-mode flag exists in editor/dev builds only (`enableGodMode`, `PlayerLife.cs:154,578-581`).
- Save format versioned: `SAVEDATA_VERSION_LATEST = 3`; `load()/save()` at `PlayerLife.cs:2476,2529` — **protected**.

**Inferred `[INFER]`:** hallucination/vision mechanics tie into `askView/askBlind` and berry items; exact UI coupling untraced. Pain/shock do not exist anywhere — genuinely new ground.

**Needs runtime verification `[RUNTIME]`:** simulation tick rate (`PlayerInput.RATE` = 0.08 s per in-code comment, i.e. 12.5 TPS — verify), which events fire client-side in singleplayer (client is also server there), event ordering on death/revive.

**Safe read-only observation points:** every delegate listed above; all stat properties; `Player.LocalPlayer.life` (guard `!Dedicator.IsDedicatedServer`).

**Risky modification points:** `simulate()` interval logic (multiplayer sync + balance), `doDamage` internals (death/RPC sequence), `load/save` (format v3), anything `tell*/Receive*/Send*`.

**Grep terms:** `onHealthUpdated`, `askDamage`, `serverModify`, `SimulateStaminaFrame`, `Food_Use_Ticks`, `SAVEDATA_VERSION`, `EDeathCause`, `OnFallDamageRequested`.

## 2. Player movement / stamina / stance

**Files/classes:** `Unturned/Player/PlayerMovement.cs` (~2,000 lines), `PlayerStance.cs`, `EPlayerStance.cs`, `PlayerInput.cs`; stamina itself lives in `PlayerLife` (§1).

**Confirmed `[SOURCE]`:**
- `EPlayerStance`: CLIMB, SWIM, SPRINT, STAND, CROUCH, PRONE, DRIVING, SITTING — `[NetEnum]`, order is wire format (`EPlayerStance.cs`).
- Speed composition: `totalSpeedMultiplier => pluginSpeedMultiplier * player.clothing.movementSpeedMultiplier …` (`PlayerMovement.cs:122-127`); `pluginSpeedMultiplier/pluginJumpMultiplier/pluginGravityMultiplier` are server-set, client-replicated via dedicated RPCs (`sendPluginSpeedMultiplier`, `PlayerMovement.cs:603-666`) — **this is the sanctioned encumbrance hook**, already plugin-facing.
- Environment flags: `isGrounded`, `inRain`, `inSnow`, `isSafe` (+`isSafeInfo` safezone node), `isRadiated`, `ActiveDeadzone`, `nav` (navmesh region byte, 255 = off-mesh), `fall` (y velocity), `onLanded` delegate (`PlayerMovement.cs:70-78,181-344`).
- Client-predicted, server-simulated movement: `simulate(...)` overloads with input + recovery params; `PlayerStateUpdate` queue (`PlayerMovement.cs:979-1051`).

**Inferred `[INFER]`:** stance transitions validated server-side in `PlayerStance` (`tellStance/askStance` pattern); sprint gating on stamina lives in stance/input code.

**Needs runtime `[RUNTIME]`:** how often plugin multipliers can change without visible snapping; interaction of `pluginSpeedMultiplier` with skill (`PlayerSkills`) multipliers.

**Safe observation points:** all properties above; `player.stance.stance`; `onStanceUpdated` delegate (`PlayerStance.cs:13`).

**Risky modification points:** `simulate` bodies (prediction/desync), `PlayerStateUpdate` flow, height/collider logic (`setSize`), teleporter internals.

**Grep terms:** `pluginSpeedMultiplier`, `totalSpeedMultiplier`, `EPlayerStance`, `isRadiated`, `ActiveDeadzone`, `onLanded`, `inSnow`.

## 3. Player inventory / equipment / clothing

**Files/classes:** `Unturned/Player/PlayerInventory.cs` (~2,240 lines), `PlayerEquipment.cs` (~3,200 lines), `PlayerClothing.cs`, `Unturned/Items/` (`Item`, `ItemJar`, `Items` page containers `[INFER: folder]`).

**Confirmed `[SOURCE]`:**
- Inventory is paged grids: `getWidth/getHeight/getItemCount/getItem(page,index)` returning `ItemJar`; add/remove/update/resize delegates (`onInventoryAdded/Removed/Updated/Resized/StateUpdated`) and `onDropItemRequested` veto handler (`PlayerInventory.cs:140-196`).
- Caliber-aware searching is built in: `FindAttachmentsByCaliber`, `search(EItemType type, ushort[] calibers)` (`PlayerInventory.cs:296-336,2087-2194`).
- `PlayerEquipment`: current `asset`, `useable`, `quality`, `state` (byte[]); for guns, attachment quality bytes at fixed indices — sight 13, tactical 14, grip 15, barrel 16, magazine 17 (`UseableGun.cs:36-40`); `updateQuality/sendUpdateQuality` mutate + replicate (`PlayerEquipment.cs:1888-1930`); `onEquipRequested/onDequipRequested` veto handlers (`PlayerEquipment.cs:87-88`).
- `PlayerClothing`: 7 slots (shirt, pants, hat, backpack, vest, mask, glasses), each with asset ref, `quality` byte, `state` byte[], update delegates, and per-slot ask/tell/send quality RPC triples (`PlayerClothing.cs:35-131,173-312`).
- Inventory `load()/save()` at `PlayerInventory.cs:1861,1932` — **protected**.

**Inferred `[INFER]`:** page indices (hands/clothing pages) follow known Unturned conventions (0–1 hands, 2 backpack, etc.) — verify constants in `PlayerInventory` before use. There is **no weight system anywhere** — encumbrance would be a genuinely new ledger over `ItemJar` contents.

**Needs runtime `[RUNTIME]`:** which inventory events fire on the client in singleplayer vs pure client; storage (`isStoring`) interactions.

**Safe observation points:** all delegates + getters above; `player.clothing.*Asset/*Quality`.

**Risky modification points:** `tryAddItem/forceAddItem` (dupe/loss bugs), any `Receive*/send*` pair, `load/save`, hotkey state.

**Grep terms:** `onInventoryAdded`, `ItemJar`, `FindAttachmentsByCaliber`, `onDropItemRequested`, `shirtQuality`, `state\[16\]`, `GunStateIndices`.

## 4. Items / assets / .dat-driven definitions

**Files/classes:** `Unturned/Bundles/ItemAsset.cs` + ~58 `Item*Asset.cs` siblings (same folder), `Unturned/Bundles/Assets.cs` (database), `UnturnedNexus.cs` (type-name → class registration), `Assets/Runtime/UnturnedDat/` (parser assembly), `SpawnAsset.cs`, `SpawnTableReward.cs`.

**Confirmed `[SOURCE]`:**
- Every item class parses fields in `PopulateAsset(in PopulateAssetParameters p)` via `p.data.Parse*("Key")` — items are data-defined; new items need only .dat content (+ bundles for visuals), zero C# for existing archetypes (e.g. `ItemConsumeableAsset.cs:263-300`, `ItemGunAsset.cs:1221-1252`, `ItemClothingAsset.cs:186-218`).
- Consumable stat vocabulary: `Health, Food, Water, Virus, Disinfectant, Energy, Vision, Oxygen, Warmth, Bleeding_Modifier (Cut/Heal/None), Bones_Modifier, Aid, Experience`, plus quest/spawn-table rewards (`ItemConsumeableAsset.cs:21-136`). Food/water items show a spoilage warning below 50% quality (`:228`) — **food degradation already exists as data**.
- Clothing: `Armor`, `Armor_Explosion`, `Proof_Water/Fire/Radiation`, `Movement_Speed_Multiplier`; comment documents that hats/shirts/pants/vests are armor-eligible (`ItemClothingAsset.cs:11-41,82,142-218`).
- Weapon base (`ItemWeaponAsset.cs`): `durability` (chance of wear per use), `wear` (quality lost), per-limb `PlayerDamageMultiplier` (`Player_Damage` × leg/arm/spine/skull multipliers), zombie/animal multiplier variants (`:20-130,340-391`).
- Attachment base `ItemCaliberAsset`: `calibers` ushort[] (`Calibers`/`Caliber_N` keys), recoil/spread/sway/shake/aim-duration/movement multipliers, `Ballistic_Damage_Multiplier`, `Ballistic_Drop` (`ItemCaliberAsset.cs:10-226`). Magazines add pellets/range/per-target damage/tracer/speed (`ItemMagazineAsset.cs`). Barrels add `isSilenced`, `volume`, `durability`, gunshot rolloff (`ItemBarrelAsset.cs:12-56`).
- Asset resolution falls back to the Steam install for core bundles (`Assets.cs:2323-2325`, per audit, re-cited).

**Inferred `[INFER]`:** modules can ship additional asset .dat folders (module system loads content); the exact search path union for item .dats (game `Bundles` vs workshop vs module) needs a trace.

**Needs runtime `[RUNTIME]`:** hot-reload behavior of .dat edits (likely none — restart), how overriding a vanilla item ID behaves vs adding new IDs.

**Safe observation points:** `Assets.find(...)` lookups; asset property getters; `player.equipment.asset` casts.

**Risky modification points:** `Assets.cs` load pipeline; changing parse defaults (silently rebalances all vanilla content); GUID/ID collisions with retail content.

**Grep terms:** `PopulateAsset`, `ParseFloat\("Armor"`, `Bleeding_Modifier`, `Caliber_`, `ItemCaliberAsset`, `SpawnTableReward`, `Assets.find`.

## 5. Weapons / guns / ammo / attachments

**Files/classes:** `Unturned/Useable/UseableGun.cs` (~3,700 lines), `Useable.cs` base, `ItemGunAsset.cs`, `ItemMagazineAsset.cs`, `ItemBarrelAsset.cs`, `ItemSightAsset/ItemTacticalAsset/ItemGripAsset` (all `ItemCaliberAsset` children), `Unturned/ModHooks/UseableGunEventHook.cs`, `GunAttachmentEventHook.cs`.

**Confirmed `[SOURCE]`:**
- Server fire block (`UseableGun.cs:912-954`): `SendPlayShoot` RPC → mod-hook shot events → fragile shot-count/rechamber bookkeeping (in-code warning at `:925-928`) → **noise** (`AlertTool.alert(transform.position, equippedGunAsset.alertRadius)` unless barrel `isSilenced` and state byte 16 > 0, `:936-939`) → durability wear (`ShouldWeaponTakeDamage` config, `durability` chance, `wear` amount, `:941-953`).
- Jamming: `canEverJam` guns roll `jamChance = lerp(0, jamMaxChance, 1 - quality/jamQualityThreshold)` server-side; `SendPlayChamberJammed` corrects predicted ammo (`UseableGun.cs:1345-1357,3087-3095`; asset fields `ItemGunAsset.cs:825-856,1247-1252`; defaults 0.4 threshold / 0.1 max chance).
- Quality → handling: spread (`CalculateSpreadAngleRadians(quality, …)`, `:991`), recoil ×(1 + (1−2q)) below 50% (`:1049-1050`), bullet damage ×(0.5+q) below 50% (`:1619`); magazine quality decrements by `stuck` cost and sets drop quality (`:1261-1275`).
- Ammo linkage: gun ↔ magazine ↔ attachments joined by caliber ushort IDs (§4); inventory search by caliber exists (§3). `alertRadius` defaults to 48 when unspecified (`ItemGunAsset.cs:1225`).
- `UseableGunEventHook` UnityEvents: `OnShotFired, OnReloadingStarted, OnChamberingStarted, OnAimingStarted/Stopped, OnMagazineVisible/Hidden` — asset-driven, zero-code observation (`UseableGunEventHook.cs:19-49`).

**Inferred `[INFER]`:** barrel (suppressor) quality decrement per shot happens in the shoot path using `ItemBarrelAsset.durability` — fields confirmed, exact decrement line not read; find via `state[GunStateIndices.BARREL_QUALITY]` writes.

**Needs runtime `[RUNTIME]`:** hit registration path (raycast vs projectile) end-to-end; how jam UX presents; whether `alertRadius` interacts with barrel `volume`.

**Safe observation points:** the mod-hook UnityEvents; `equippedGunAsset` fields; equipment state bytes (read).

**Risky modification points:** anything inside the `Provider.isServer` fire block (explicit in-code warning), `SendPlayShoot`/NetGen, reload/rechamber state machine, ballistics buffers.

**Grep terms:** `alertRadius`, `canEverJam`, `SendPlayChamberJammed`, `BARREL_QUALITY`, `InvokeModHookShotFiredEvents`, `CalculateSpreadAngleRadians`, `bulletsInRange` `[INFER]`.

## 6. Damage flow

**Files/classes:** `Unturned/Tools/DamageTool.cs` (~1,540 lines), `Unturned/Damage/` (`DamagePlayerParameters.cs`, `PlayerDamageMultiplier.cs`, `ELimb.cs`, `IDamageMultiplier.cs`, zombie/animal variants, `ExplosionParameters.cs`), `NetGen/NetInvokable/DamageTool_NetMethods.cs` (protected).

**Confirmed `[SOURCE]`:**
- Canonical player pipeline (`DamageTool.damagePlayer`, `:332-443`): build `DamagePlayerParameters` → fire `damagePlayerRequested(ref parameters, ref bool shouldAllow)` (**modify-or-veto event — the single most valuable hardcore seam**) → optional per-limb armor `getPlayerArmor(limb, player)` (`:483`, clothing armor by slot vs limb) → global `Players.Armor_Multiplier` → floor/round → `player.life.askDamage(...)` → post-effects from parameters: `bleedingModifier` (Default/Always/Never/Heal), `bonesModifier`, food/water/virus/hallucination deltas (`:355-442`).
- Parallel events: `damageZombieRequested`, `damageAnimalRequested`, `onPlayerAllowedToDamagePlayer` (`:133-152,1531-1535`).
- `ELimb` granularity feeds both armor and the `onHurt` event; weapon assets carry per-limb multipliers (§4).
- Explosions: `explode(ExplosionParameters, out kills)` (`:1044`); raycast helper `DamageTool.raycast(Ray, range, mask, ignorePlayer)` (`:1459-1464`).

**Inferred `[INFER]`:** all gameplay damage funnels through `DamageTool` (grep found no direct `askDamage` callers outside `PlayerLife.simulate` self-damage) — verify by broader grep before relying on the veto event catching *everything*.

**Needs runtime `[RUNTIME]`:** event firing context in singleplayer (client==server) — confirm `damagePlayerRequested` fires locally.

**Safe observation points:** subscribe to `damagePlayerRequested`/`damageZombieRequested` with `shouldAllow` untouched (pure telemetry); `onHurt`.

**Risky modification points:** mutating `parameters` (rebalances everything incl. PvP), armor lookup internals, explosion iteration, NetGen file.

**Grep terms:** `damagePlayerRequested`, `DamagePlayerParameters`, `getPlayerArmor`, `ELimb`, `bleedingModifier`, `explode\(`.

## 7. Zombies / AlertTool / noise

**Files/classes:** `Unturned/Tools/AlertTool.cs`, `Unturned/Managers/ZombieManager.cs`, `ZombieRegion.cs`, `Unturned/Zombies/Zombie.cs`, `NonPathfindingZombieMovementComponent.cs`, `Unturned/Level/LevelZombies.cs`, `LevelNavigation.cs`, `ModHooks/MobAlertSpawner.cs`.

**Confirmed `[SOURCE]` (re-verified this pass):**
- Two alert entries: `alert(Player, position, radius, sneak, spotDir, isSpotOn)` (vision/proximity, LOS raycasts, `Detect_Radius_Multiplier`, `LevelAsset.minStealthRadius`) at `AlertTool.cs:55`; positional noise `alert(Vector3 position, float radius)` at `:218` — alerts zombies in radius when position is on navigation, startles animals.
- Guns call the positional overload server-side unless suppressed (§5). Debug defines `LOG_ALERTS`, `ALERT_SPHERE_GIZMOS` draw via `RuntimeGizmos` (audit; define names re-seen in `AlertTool.cs` header region).
- Zombie counts cheaply via `ZombieManager.tickingZombies.Count` (used by `CommandDebug.cs:28`); regions per navmesh bound created on `onLevelLoaded` (`ZombieManager.cs:1523-1534` per audit).
- Pathfinding stubbed (`UnturnedPathfinding_Empty`) — judge AI experiments by reaction, not path quality (risk #10).

**Inferred `[INFER]`:** zombie target/alert state fields on `Zombie` are readable but accessibility (public vs internal) needs checking at implementation time; movement-noise alerts from sprinting players originate in zombie sense ticks calling the Player overload.

**Needs runtime `[RUNTIME]`:** actual investigate/chase behavior quality with stubbed pathfinding on PEI; alert radius feel at default 48.

**Safe observation points:** counting zombies; reading region arrays; enabling gizmo defines *locally without committing*; watching `alert` effects visually.

**Risky modification points:** `ZombieManager.Update` tick loop, region arrays, spawn tables, `Zombie` targeting internals, `ZombieManager_NetMethods` (NetGen).

**Grep terms:** `AlertTool.alert`, `minStealthRadius`, `Detect_Radius_Multiplier`, `tickingZombies`, `LOG_ALERTS`, `getZombiesInRadius`, `MobAlertSpawner`.

## 8. Loot spawning / item spawning / ItemManager

**Files/classes:** `Unturned/Managers/ItemManager.cs` (~950 lines), `ItemRegion.cs`, `ItemData.cs`, `InteractableItem` `[INFER: Unturned/Interactables]`, `Unturned/Level/LevelItems.cs` (spawn points/tables per region), `Unturned/Bundles/SpawnAsset.cs`, `Unturned/Tools/SpawnTableTool.cs`.

**Confirmed `[SOURCE]`:**
- Server loot lifecycle: `respawnItems()` walks regions on a cursor; per region, target count = `spawnpointCount × Items.Spawn_Chance`, one item per `Items.Respawn_Time` from `LevelItems.spawns[x,y]` spawnpoints resolving through `LevelItems.tables[spawn.type]` (`ItemManager.cs:764-830`). `despawnItems()` removes items older than `Despawn_Dropped_Time`/`Despawn_Natural_Time` (`:734-752`).
- Utilities: `getItemsInRadius`, `findSimulatedItemsInRadius`, `dropItem(Item, point, playEffect, isDropped, wideSpread)`, `ServerClearItemsInSphere` (`:86-157,450`).
- Spawn tables are assets: `SpawnTableTool.Resolve(SpawnAsset, …)` with legacy-ID paths; **`LogAllSpawnTables()` debug dump already exists** (`SpawnTableTool.cs:19-198,719`).
- All spawn/despawn tunables in `ItemsConfigData` (`PlayConfigData.cs:481`) — data, not code.

**Inferred `[INFER]`:** map spawnpoint placement comes from per-level files loaded by `LevelItems` (PEI's tables live in the Steam-installed map); scarcity tuning = difficulty config + (later) custom spawn-table .dats, no manager edits.

**Needs runtime `[RUNTIME]`:** actual PEI loot density under default config; whether `Spawn_Chance` changes require level reload.

**Safe observation points:** region item counts; `LogAllSpawnTables()`; radius queries at low Hz.

**Risky modification points:** respawn/despawn loops (server perf + economy), `ReceiveItem*` RPCs, `instanceID` handling.

**Grep terms:** `respawnItems`, `Spawn_Chance`, `LevelItems.spawns`, `SpawnTableTool`, `Despawn_Natural_Time`, `ItemSpawnpoint`.

## 9. Crafting / repair

**Files/classes:** `Unturned/Player/PlayerCrafting.cs` (~1,370 lines), `Blueprint` (+ `EBlueprintType` incl. repair `[INFER]`) in `Unturned/Items|Bundles`, `ModHooks/CraftingTagProviderComponent.cs`, `CraftingTagModifierComponent.cs`.

**Confirmed `[SOURCE]`:**
- `onCraftingRequested` (`PlayerCraftingRequestHandler`) veto handler + `onCraftingUpdated` (`PlayerCrafting.cs:78-85`); server flow `ReceiveCraft` → `HandleCraftRequestInternal` (`:679-740`); blueprint blacklisting and per-level permanent disables (`:198,370`); nearby crafting-tag providers (workstation-style gating via `TagAsset`, `:24-190`); attachment stripping (`ReceiveStripAttachments`, `:304`).
- Blueprints are item-asset data (crafting recipes ship as .dat entries on items) `[SOURCE: Blueprint usage; INFER for full field list]` — repair-type blueprints restore quality `[INFER: EBlueprintType.REPAIR convention — verify enum]`.

**Needs runtime `[RUNTIME]`:** repair blueprint behavior on quality restoration; tag providers on vanilla PEI objects.

**Safe observation points:** subscribe to both delegates; enumerate `ItemAsset` blueprints for display.

**Risky modification points:** `HandleCraftRequestInternal` (item deletion/creation = dupe risk), blueprint indices (saved in preferences by index `[INFER]`).

**Grep terms:** `onCraftingRequested`, `Blueprint`, `EBlueprintType`, `CraftingTag`, `HandleCraftRequestInternal`, `Strip`.

## 10. UI / HUD / debug overlay seams

**Files/classes:** `Unturned/Player/PlayerUI.cs`, `Unturned/UI/Player/PlayerLifeUI.cs`, `PlayerDashboardUI.cs` + sub-UIs, `Unturned/Sleek/` widget layer, `Glazier_*` backends selected by `GlazierFactory` at boot (`Setup.cs:50`), `Framework/Debug/`, `Unturned/Tools/RuntimeGizmos.cs` `[SOURCE folder/file listing + audit re-cite]`.

**Confirmed `[SOURCE]`:** HUD is code-built Sleek widget trees (no scene prefabs to break); `PlayerDashboardInformationUI.cs` reads `Player.LocalPlayer.transform.position` (audit-cited pattern for safe reads); `CommandDebug` prints UPS/TPS + zombie/animal counts; `RuntimeGizmos` draws runtime spheres/lines (used by alert debug defines).

**Decision (user-approved):** foundation overlay = plain IMGUI `OnGUI()` MonoBehaviour — no Sleek/Glazier coupling, trivially removable. Sleek widgets reconsidered only when a *player-facing* hardcore HUD ships.

**Needs runtime `[RUNTIME]`:** `OnGUI` interplay with the IMGUI Glazier backend if that backend is active (default backend selection unverified).

**Safe observation points:** everything §1–§9 lists; overlay renders reads only.

**Risky modification points:** `PlayerUI` lifecycle, Sleek internals, Glazier factory.

**Grep terms:** `PlayerLifeUI`, `ISleek`, `GlazierFactory`, `OnGUI`, `RuntimeGizmos`, `PlayerDashboardInformationUI`.

## 11. Save / persistence (protected — future work only)

**Files/classes `[SOURCE]`:** `Unturned/Files/ReadWrite.cs` (root `Builds/Shared` in editor via `UnturnedPaths.cs:24`), `PlayerSavedata.cs` (`/Players/{steamID}_{char}/{level}/…`), `ServerSavedata.cs`, `SaveManager.cs`, per-system `save()/load()` incl. `PlayerLife.cs:2476-2529` (v3), `PlayerInventory.cs:1861-1932`.

**Status:** entirely off-limits for the foundation and slice 1 (risk #4: Block/River field order **is** the format; no migration framework). Vanilla persistence keeps working untouched — which is precisely how slice 1 "keeps loot" without new writes. When hardcore state eventually persists, it goes in **new, separately versioned files** under a hardcore-specific subfolder, never appended to vanilla streams.

**Grep terms (future):** `writeBlock`, `openRiver`, `PlayerSavedata`, `SAVEDATA_VERSION`.

## 12. Networking authority boundaries (protected — future work only)

**Files/classes `[SOURCE]`:** `NetMessaging/`, `NetInvokable/`, generated `NetGen/*_NetMethods.cs` (committed; regenerated only via Window > Unturned > Net Gen — never run casually, risk #19), transport in `SDG.NetTransport` + `NetTransport_Loopback` for singleplayer.

**Boundaries that shape hardcore work:**
- Role checks: `Provider.isServer` (simulation authority), `Dedicator.IsDedicatedServer` (process role), `channel.IsLocalPlayer` (ownership). Singleplayer is client+server in one process via loopback — foundation code must still branch correctly for the co-op future.
- Existing replication vocabulary already covers most hardcore needs (life stats, equipment quality, clothing quality, plugin multipliers) — design future systems to *compose these existing RPCs* server-side rather than adding messages; new replicated state = new NetGen surface = explicit user approval first.
- `Player.LocalPlayer` throws on dedicated server in editor/dev builds (`Player.cs:231-235`).

**Grep terms (future):** `ClientInstanceMethod`, `ServerInstanceMethod`, `InvokeAndLoopback`, `GatherRemoteClientConnections`, `NetEnum`.
