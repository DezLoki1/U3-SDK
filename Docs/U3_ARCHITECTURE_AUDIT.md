# U3-SDK Architecture Audit

Audit date: 2026-07-07. Branch: `fable-architecture`. Produced by a read-only reconnaissance pass; no source, assets, or settings were modified.

**Confidence labels used throughout:**
- `[SOURCE]` — confirmed by reading tracked source/config in this repository (file:line cited where useful).
- `[LOCAL]` — confirmed from the local environment (folder listings, git state), not necessarily true for other clones.
- `[INFER]` — probable inference from naming/structure; not directly read end-to-end.
- `[RUNTIME]` — needs runtime verification (Play Mode / build / server) before being relied on.

---

## 1. Executive summary

This repository is the **U3 SDK**: the near-complete C# source and Unity project for Unturned (Steam AppID 304930), published by Smartly Dressed Games for building non-commercial mods of the game. `[SOURCE: README.md, LICENSE.txt, steam_appid.txt]`

Key architectural facts every future session must know:

1. **The repo is not self-contained.** Large binaries, vanilla maps, core asset bundles, localization, loading screens, and economy data are read at runtime from the **installed Steam copy of Unturned** via `Provider.steamAppInstallDirectory`. In the editor and development builds, startup **hard-exits if Steam is not running or Unturned is not installed**. `[SOURCE: Provider.cs:6754-6783, SteamworksProvider.cs:112-120, Level.cs:1137-1141]`
2. **One scene bootstraps everything.** `Assets/GameStartup.unity` contains a `Setup` object whose `Awake()` runs the entire deterministic boot sequence (Dedicator → Logs → ModuleHook → Provider → Glazier UI → pathfinding). Almost all game systems are **static-heavy singleton managers** living on a single `Managers` GameObject marked `DontDestroyOnLoad`. `[SOURCE: Setup.cs:21-54, GameStartup.unity]`
3. **Third-party pathfinding is intentionally absent.** Navigation is abstracted behind `IUnturnedPathfindingInterface`; the SDK compiles `UnturnedPathfinding_Empty` because the A* Pathfinding Project ("ASPFP") implementation is not licensed for redistribution. Degraded zombie navigation is expected, not a setup bug. `[SOURCE: UnturnedPathfinding.cs:9-27]`
4. **A game-level noise/alert abstraction already exists** (`AlertTool`), and firearms already emit alerts server-side (`UseableGun.cs:938`). "Zombies hear gunshots" is an existing, observable feature — a superb learning seam. `[SOURCE: AlertTool.cs, UseableGun.cs:912-939]`
5. **Client/server authority is explicit and pervasive.** `Provider.isServer` / `Dedicator.IsDedicatedServer` gate simulation; RPCs are generated code (`NetGen/*_NetMethods.cs`, committed to the repo) driven by the `NetInvokable` reflection layer. Changing message shapes or generated code by hand breaks protocol compatibility. `[SOURCE: NetMessaging/, NetInvokable/, NetGen/, .gitignore:154-157]`
6. **Saves are custom binary streams** (`Block`/`River` via `ReadWrite`), rooted at `Builds/Shared` when running in the editor. Save-format edits are among the most dangerous changes possible in this codebase. `[SOURCE: UnturnedPaths.cs:24, ReadWrite.cs:23, PlayerSavedata.cs]`

## 2. Baseline environment and known-good setup

| Fact | Status |
|---|---|
| Unity **2022.3.62f3** required | `[SOURCE: ProjectSettings/ProjectVersion.txt, README.md]` |
| Entry scene `Assets/GameStartup.unity`, then Play | `[SOURCE: README.md]`, `[LOCAL]` verified working by user |
| Steam running + retail Unturned installed required for editor Play Mode | `[SOURCE: Provider.cs:6764-6782]` |
| `Window > Unturned > Build Tool > Build Test` produces a runnable Windows copy in `Builds/Test` | `[SOURCE: BuildTool.cs, .gitignore:62-63]`, `[LOCAL]` verified working by user |
| Working tree clean on `fable-architecture`; `origin` = user fork, `upstream` = SmartlyDressedGames/U3-SDK (push disabled) | `[LOCAL]` |
| `Build_Scripts/` exists locally; it is gitignored generated tooling output — leave it alone | `[LOCAL]`, `[SOURCE: .gitignore:140-143]` |
| This fork tracks only `ModInfo.json` + `Status.json` under `Builds/Shared`; **no vanilla maps are in the repo** — they load from the Steam install fallback | `[LOCAL: git ls-files]`, `[SOURCE: Level.cs:1137-1141]` |

`Builds/Shared/Status.json` carries the game version (26.3.4 at audit time) used to compute `Provider.APP_VERSION`; `ModInfo.json` names the mod ("Default Mod", version 0.0.0) and is logged at startup. `[SOURCE: Provider.cs:6550-6559, Builds/Shared/*.json]`

## 3. Repository structure overview

| Path | Purpose | Confidence |
|---|---|---|
| `Assets/Runtime/Assembly-CSharp/` | ~1,650 C# files: the entire game (`SDG.Unturned` and friends). Compiled into Unity's default `Assembly-CSharp` (no asmdef). | `[SOURCE]` |
| `Assets/Runtime/<Name>/` (LiveConfig, SDG.Glazier, SDG.HostBans, SDG.NetPak, SDG.NetTransport, SystemEx, UnityEx, UnturnedDat) | Modular runtime assemblies, each with an `.asmdef`. | `[SOURCE]` |
| `Assets/Editor/` | Editor-only assemblies: `Assembly-CSharp-Editor` (default), `Jenkins.Editor`, `SteamCmd.Editor`, `UnityEditorEx`. Build/bundle/netgen tools live here. | `[SOURCE]` |
| `Assets/Game/Sources/` | Art content: Animations, Models, Scenes (Menu/Game/Loading), Shaders, Skins, Textures. | `[LOCAL ls]` |
| `Assets/com.rlabrecque.steamworks.net/` | Vendored Steamworks.NET (runtime + editor asmdefs). | `[SOURCE]` |
| `Assets/GameStartup.unity` | Boot scene (see §5). | `[SOURCE]` |
| `Assets/Tests/` | `SDG.NetPak.Tests`, `UnturnedDat` test asmdefs. | `[SOURCE]` |
| `Builds/Shared/` | The game's runtime data root in editor (`ReadWrite.PATH`): saves (`Worlds`, `Servers`, `Cloud`), `Modules`, `Maps` (absent here), `Sandbox`, logs. Mostly gitignored. | `[SOURCE: UnturnedPaths.cs:24, .gitignore]` |
| `Builds/Test/` | Gitignored output of "Build Test" — a runnable Windows copy of the game. | `[SOURCE: .gitignore:62-63]` |
| `Editor_Test_Config/` | Text lists of expected vanilla/curated/server-test maps + news test file, used by editor test tooling. | `[LOCAL ls]`, `[INFER]` purpose from names |
| `JenkinsBootstrapper/` | Small standalone C# console project run by CI before launching Unity. | `[SOURCE: .gitignore:163-171]` |
| `WindowsAssertionFix/` | C++ DLL project for Windows dedicated servers (assertion popup suppression). | `[SOURCE: .gitignore:173-176]`, `[INFER]` exact behavior |
| `ProjectSettings/`, `Packages/` | Unity project config; `manifest.json` pins packages (PostProcessing 3.4.0, TextMeshPro 3.0.7, Burst 1.8.21, Linux toolchain, etc.). No `packages-lock.json` is tracked at repo root. | `[SOURCE]` |
| Root `*.csproj` / `UnturnedSDK.sln` | **Generated by Unity** (gitignored: `.gitignore:121-124`); never hand-edit. | `[SOURCE]` |
| `steam_appid.txt` | `304930` — retail Unturned AppID used for Steam init. | `[SOURCE]` |
| `.editorconfig` | Tabs, `System` usings first, Allman braces, etc. Match it when writing code. | `[SOURCE]` |
| `.vsconfig` | Requests the VS "Managed Game" workload only. | `[SOURCE]` |
| `THIRDPARTYNOTICES.txt` | Notices for Json.NET, Liberation Sans, and other bundled OSS; must ship with mods per LICENSE §1(b). | `[SOURCE]` |

**License constraints that shape engineering work** `[SOURCE: LICENSE.txt]`: mods must be free/non-commercial, must include LICENSE + THIRDPARTYNOTICES, PC platforms only. The UNTURNED trademark may not be used as a brand for a mod.

## 4. Assembly and Unity project architecture

- The gameplay monolith is **`Assembly-CSharp`** (default assembly, `Assets/Runtime/Assembly-CSharp`). Namespaces observed: `SDG.Unturned` (game), `SDG.Framework.*` (modules, devkit, utilities, debug), `SDG.SteamworksProvider` + `SDG.Provider` (platform service layer), plus Glazier/NetPak/NetTransport glue folders compiled into Assembly-CSharp (`Glazier/`, `NetMessaging/`, `NetInvokable/`, `NetGen/`, `NetTransport_*`). `[SOURCE]`
- Support assemblies (asmdef-isolated): `SystemEx`, `UnityEx`, `UnturnedDat` (.dat parsing), `SDG.NetPak.Runtime` (bit packing), `SDG.NetTransport` (transport abstraction), `SDG.Glazier.Runtime`, `SDG.HostBans.Runtime`, `LiveConfig`. `[SOURCE: asmdef list]`
- Editor-only code is separated both by asmdef (`Jenkins.Editor`, `SteamCmd.Editor`, `UnityEditorEx`) and by `#if UNITY_EDITOR` blocks inside runtime files (e.g. `Provider.cs` uses `EditorPrefs`, `EditorUtility` under the define). Follow that pattern exactly; editor APIs outside those guards break player builds. `[SOURCE]`
- Compile-time feature defines observed: `DEDICATED_SERVER`, `DEVELOPMENT_BUILD`, `WITH_ASPFP` (pathfinding), `WITH_THIRDPARTYAC` (anti-cheat), `WITH_NOREDIST`, `EXPERIMENTAL`, `DISABLESTEAMWORKS`, `STATDEBUG`, `LOG_ALERTS`, `ALERT_SPHERE_GIZMOS`. The SDK build has the third-party ones **off**. `[SOURCE: Dedicator.cs, Provider.cs, UnturnedPathfinding.cs, AlertTool.cs, CommandDebug.cs]`
- Scenes in build: `GameStartup`, `Menu`, `Game`, `Loading`, `Menu_Fallback`. `[SOURCE: ProjectSettings/EditorBuildSettings.asset]`

## 5. Startup flow

Everything begins in `Assets/GameStartup.unity`. Root objects and their scripts (all paths under `Assets/Runtime/Assembly-CSharp/`) `[SOURCE: scene GUID → .meta resolution]`:

| Scene object | Components (class — file) |
|---|---|
| **Setup** | `Dedicator`, `SDG.Framework.Modules.ModuleHook`, `Provider`, `Setup`, `Logs` (Unturned/Provider/, Framework/Modules/, Unturned/Loading/Setup.cs, Unturned/Managers/Logs.cs) |
| **Managers** | `Managers`, `SteamChannel`, `BeaconManager`, `SafezoneManager`, `OxygenManager`, `TemperatureManager`, `ClaimManager`, `VehicleManager`, `BarricadeManager`, `ZombieManager`, `AnimalManager`, `ItemManager`, `SaveManager`, `LightingManager`, `EffectManager`, `ResourceManager`, `ChatManager`, `StructureManager`, `ObjectManager`, `LevelManager`, `PlayerManager`, `TimeUtility`, `GroupManager` (Unturned/Managers/, Unturned/Provider/SteamChannel.cs, Framework/Utilities/TimeUtility.cs) |
| **Tools** | `DontDestroyOnLoad` (General/), `ItemTool`, `VehicleTool` (Unturned/Tools/) |
| **Loading** | `LoadingUI` (Unturned/Loading/LoadingUI.cs) |
| **Bundles** | `Bundles`, `Assets` (Unturned/Bundles/) |
| **Level** | `Level`, `LevelGround`, `LevelObjects` (Unturned/Level/) |
| **PlaceholderAudioListener / PostProcess / EventSystem** | audio listener; `UnturnedPostProcess`; uGUI EventSystem + `DontDestroyOnLoad` |

**Boot order inside `Setup.Awake()`** `[SOURCE: Setup.cs:21-54]`:

1. `UnturnedPlayerLoop.initialize()` — custom player-loop tweaks; `ThreadUtil.setupGameThread()`.
2. Editor only: `CommandLineFlag.applyEditorPreferencesToAllFlags()` — editor-prefs-backed emulation of command-line flags.
3. `Dedicator.awake()` — decides client vs dedicated server. Server mode comes from `+InternetServer/{ID}` / `+LANServer/{ID}` command line, or the editor flag `-DedicatedServerInEditor` (`Dedicator.cs:81-97,159`). Creates `CommandWindow` and caps `Application.targetFrameRate = 50` for servers.
4. `Logs.awake()` — log file setup.
5. `ModuleHook.awake()` then `.start()` — discovers and loads **modules** (mod DLLs + `.module` configs) from `<ReadWrite.PATH>/Modules` (`ModuleHook.cs:494`). Game code itself registers `UnturnedNexus : IModuleNexus`, which registers every asset type name → C# asset class (`UnturnedNexus.cs:14+`).
6. `Provider.awake()` — the big one (§ below).
7. `Provider.start()`.
8. Client only: `GlazierFactory.Create()` — instantiates the UI backend (IMGUI/uGUI/UIToolkit variants exist).
9. `UnturnedPathfinding.Initialize()` — installs `UnturnedPathfinding_Empty` (SDK) or ASPFP (licensed builds).

**`Provider.awake()` essentials** `[SOURCE: Provider.cs:6548-6940]`:
- Loads `Status.json`/`ModInfo.json`, computes `APP_VERSION`; singleton guard + `DontDestroyOnLoad`; subscribes `Level.onLevelLoaded`.
- **Dedicated-server branch**: Steam GameServer init via `SDG.SteamworksProvider.SteamworksProvider`, console localization, defaults (`maxPlayers=8`, `port=27015`, `map="PEI"`, editor override via `EditorPrefs "AutoLoadLevel"`), `Commander.init()`, executes command line + `Server/Commands.dat` commands, creates server folder tree (`/Bundles`, `/Maps`, `/Workshop/...`), loads `ConfigData`/`ModeConfigData`, HostBans refresh.
- **Client branch**: Steam client init (Play Mode exits with a dialog if Steam is missing, `Provider.cs:6765`, or if the Unturned install can't be found, `:6776`), Steam callbacks, user identity, workshop UGC refresh (`provider.workshopService.refreshUGC()`), economy promo grants, LiveConfig refresh, server-list curation, language selection (Steam UI language, workshop localization, or Steam-install `Localization/` fallback).

After boot, `LoadingUI.updateScene()` drives scene changes; the menu scene loads for clients, while servers proceed to load the level directly. `[SOURCE: Setup.cs:69, Level.cs:798-801]`, flow details `[INFER]` — exact menu handoff not fully traced.

## 6. World and level lifecycle

- `Level` (static + MonoBehaviour hybrid, `Unturned/Level/Level.cs`) owns the level lifecycle. `Level.load(LevelInfo, hasAuthority)` sets state and calls `SceneManager.LoadScene("Game")` (`Level.cs:697`); `Level.loading()` shows the "Loading" scene; `Level.exit()` fires `onLevelExited`. `[SOURCE]`
- During load, the level's sub-systems initialize from per-map files: `LevelNavigation.load()` (`:1746`), `LevelZombies.load()` (`:1823`), plus objects/ground/etc.; when complete, `Level.onLevelLoaded` is invoked (`:1989`) — **the** hook every manager uses to (re)initialize (`ZombieManager.cs:1828`, `Provider.cs:6571`). `[SOURCE]`
- Level saving symmetrically calls `LevelNavigation.save()` / `LevelZombies.save()` etc. (`Level.cs:604,628`). `[SOURCE]`
- **Level discovery** searches, in order: `<ReadWrite.PATH>/Maps` — falling back to `<SteamInstall>/Maps` when the local folder is absent (`Level.cs:1137-1141`) — then legacy workshop folders, then subscribed workshop content. `[SOURCE + INFER for full ordering]`
- Map identity: `Level.info.name`, level config via `Level.getAsset()` (`LevelAsset`, e.g. `minStealthRadius` used by AlertTool). `[SOURCE: AlertTool.cs:57-58]`
- Time of day / weather: `LightingManager` (scene manager). `[INFER from name + scene placement]`

## 7. Player systems

All under `Assets/Runtime/Assembly-CSharp/Unturned/Player/`. `Player` is a component aggregating sibling modules exposed as properties: `PlayerLife`, `PlayerMovement`, `PlayerLook`, `PlayerStance`, `PlayerInput`, `PlayerEquipment`, `PlayerInventory`, `PlayerClothing`, `PlayerSkills`, `PlayerQuests`, `PlayerCrafting`, `PlayerAnimator`, `PlayerInteract`, `PlayerVoice`, `PlayerWorkzone`. `[SOURCE: folder listing; aggregation INFER — standard `player.life` / `player.movement` / `player.equipment` accessors observed in AlertTool.cs:73 (`player.movement.nav`) and UseableGun.cs (`player.equipment.*`)]`

- **Local player access:** `Player.LocalPlayer` (static). It **throws `NotSupportedException` on dedicated servers** (`Player.cs:226-247`); any client-side overlay must check `Dedicator.IsDedicatedServer` first. Obsolete aliases `Player.player` / `Player.instance` remain for plugin compatibility — do not remove. `[SOURCE]`
- **Lifecycle/ownership:** `PlayerManager` (scene manager) tracks connected players; each player has a `SteamChannel` (`channel.owner`, `channel.IsLocalPlayer`) binding network identity to the object. `[SOURCE: UseableGun.cs:918, AlertTool.cs:73; manager internals INFER]`
- **Vitals:** `PlayerLife` (health/stamina/food/water/virus per enum types `EPlayerBoost`, `EDeathCause`, etc.). Server-authoritative simulation with client replication. `[INFER from file names + authority patterns; exact flows RUNTIME]`
- **Movement:** `PlayerMovement` includes `nav` — the index of the navmesh region the player currently occupies (255 = none), which is exactly what the zombie alert system keys off. `[SOURCE: AlertTool.cs:79-81]`
- **Equipment/weapons:** `PlayerEquipment` owns the equipped `Useable` (see §9).
- **Death/respawn & save:** `PlayerLife`/`PlayerManager` handle death; per-player state persists via `PlayerSavedata` under `/Players/{steamID}_{characterID}/{levelName}/...` (§10). `[SOURCE: PlayerSavedata.cs:19]`
- **Safe observation points for a local debug overlay:** `Player.LocalPlayer.transform.position` (used already by `PlayerDashboardInformationUI.cs:439`), `Level.info.name`, `LightingManager` time, `ZombieManager.tickingZombies.Count` (used by `CommandDebug.cs`), `Provider.isServer` / `Provider.isClient` / `Dedicator.IsDedicatedServer`. All are read-only statics already consumed by UI/diagnostic code. `[SOURCE]`

## 8. Zombie and AI systems

- **Regions:** `ZombieManager.regions` is an array of `ZombieRegion`, one per navmesh bound; created in `onLevelLoaded` from `LevelNavigation.bounds` (`ZombieManager.cs:1523-1534`). Zombie spawn points come from `LevelZombies` (per-map data loaded in `Level.cs:1823`). `[SOURCE]`
- **Ticking:** `ZombieManager.Update()` (`:1704`) ticks `tickingZombies`; zombies are only simulated server-side (managers early-out otherwise — `[INFER]`, verify at runtime). Zombie count is available cheaply via `ZombieManager.tickingZombies.Count` `[SOURCE: CommandDebug.cs:28]`.
- **Alert/stimulus model (the important seam):**
  - `Zombie.checkAlert(Player)` — can this zombie target that player; `Zombie.alert(Player)` (`Zombie.cs:1073`) — acquire player target; `Zombie.alert(Vector3, bool isStartling)` (`:1225`) — investigate a position. `[SOURCE]`
  - `AlertTool.alert(Player, position, radius, sneak, spotDir, isSpotOn)` — vision/proximity alerts with line-of-sight raycasts (`RayMasks.BLOCK_VISION`), sneak dot-product checks, flashlight detection, `Detect_Radius_Multiplier` game config, and `LevelAsset.minStealthRadius`. `[SOURCE: AlertTool.cs:55-211]`
  - `AlertTool.alert(position, radius)` — **positional noise** (gunshots etc.): alerts all zombies in radius (`zombie.alert(position, true)`) if the position is on navigation (`LevelNavigation.checkNavigation`), and makes animals flee/investigate. `[SOURCE: AlertTool.cs:218-270]`
  - Built-in debug hooks: `LOG_ALERTS`, `ALERT_SPHERE_GIZMOS`, `ALERT_LINE_OF_SIGHT_GIZMOS` defines drawing via `RuntimeGizmos` (`Utils/RuntimeGizmos.cs`). `[SOURCE: AlertTool.cs:5-9]`
- **Movement/pathfinding:** `IUnturnedPathfindingMovementComponentInterface` per zombie, produced by `UnturnedPathfinding.Get().CreateMovementComponentForZombie(zombie)`. In the SDK this is the Empty implementation; `NonPathfindingZombieMovementComponent.cs` exists alongside `Zombie.cs`. **Any zombie chase/wander behavior depending on real navmesh pathing needs runtime verification and will differ from retail.** `[SOURCE: UnturnedPathfinding.cs; behavior RUNTIME]`
- **Damage/death/loot:** zombies take damage through `DamageTool` (see §9); death triggers loot drops via `ItemManager` spawn tables `[INFER — not traced line-by-line]`.
- **Networking:** `ZombieManager_NetMethods.cs` exists in `NetGen/` — zombie state replication is part of the generated protocol. Do not touch. `[SOURCE: NetGen listing]`
- **Unsafe to alter in a first experiment:** zombie tick loops, region arrays, replication, spawn tables. Safe: *observing* alerts (e.g. reading target state), counting zombies, enabling the existing gizmo defines locally.

## 9. Weapons, damage, and noise

Flow for firing a gun (`Unturned/Useable/UseableGun.cs`, equipped via `PlayerEquipment`):

1. Input reaches the equipped `Useable` (client predicts; server validates). `[INFER: standard pattern; exact input path RUNTIME]`
2. Server block (`if (Provider.isServer)`, `UseableGun.cs:912-954`) `[SOURCE]`:
   - `SendPlayShoot.Invoke(GetNetId(), ENetReliability.Unreliable, …)` — generated RPC telling nearby clients to play the shot.
   - Careful shot-count/rechamber bookkeeping (see the in-code comment warning at `:925-928` — respect it).
   - **Noise:** unless a silencer is fitted and functional (`thirdAttachments.barrelAsset.isSilenced && player.equipment.state[16] != 0`), calls `AlertTool.alert(transform.position, equippedGunAsset.alertRadius)` (`:936-939`).
   - Weapon durability wear + `sendUpdateQuality()`.
3. Ballistics/damage resolve through raycast/projectile paths into `DamageTool` (`Unturned/Damage/`, `DamageTool_NetMethods.cs` in NetGen). `[INFER from folder/file names; details RUNTIME]`
4. **Mod hook:** `InvokeModHookShotFiredEvents()` (`UseableGun.cs:922`) feeds `UseableGunEventHook` components (`Unturned/ModHooks/UseableGunEventHook.cs`) — an intentional, asset-driven extension point. The `ModHooks/` folder holds ~45 such components (e.g. `MobAlertSpawner`, `EffectSpawner`, `TimerEventHook`) designed to react to gameplay events without engine changes. `[SOURCE]`

**Answers to the four scoping questions (all from source):**
1. *Existing noise/aggro abstraction?* Yes — `AlertTool` (noise + vision), `Zombie.alert*`, `equippedGunAsset.alertRadius` data on gun assets.
2. *Lowest-risk observation point for firearm discharge?* The `UseableGunEventHook` mod-hook path or a listener around `InvokeModHookShotFiredEvents`; read-only logging near `AlertTool.alert` is also viable. Enabling `LOG_ALERTS` locally gives telemetry with a one-line define change (still a source change — prefer observing via hooks first).
3. *Where would "zombies hear gunshots" live?* It already exists: `UseableGun` (server) → `AlertTool.alert(position, radius)` → `Zombie.alert(position, isStartling)`. Any tuning belongs in that chain, server-side only.
4. *Danger zones:* anything inside `if (Provider.isServer)` blocks changes multiplayer behavior; `SendPlayShoot`/NetGen RPCs are protocol; shot-count/rechamber logic is explicitly fragile; ammo/quality mutations touch save data.

## 10. Persistence and configuration

- **Path root:** `ReadWrite.PATH` = `UnturnedPaths.RootDirectory` = `<project>/Builds/Shared` in editor, the game directory in builds (`UnturnedPaths.cs:17-29`, `ReadWrite.cs:23`). `[SOURCE]`
- **Containers:** custom binary `Block` reader/writer and `River` streams via `ReadWrite` — not Unity serialization, not JSON (JSON is used for a few files e.g. `ConvenientSavedata.json`, `Preferences.json`). `[SOURCE: PlayerSavedata.cs (writeBlock/openRiver), ConvenientSavedata.cs:120]`
- **Layout** `[SOURCE: PlayerSavedata.cs, ServerSavedata.cs:34, Provider.cs:6670]`:
  - Singleplayer world data: `/Worlds/...`
  - Dedicated server data: `/Servers/{serverID}/...` (+ `/Server/Commands.dat`, adminlist/whitelist/blacklist)
  - Per-player per-map: `/Players/{steamID}_{characterID}/{levelName}/{file}`
  - Client conveniences: `/Cloud/ConvenientSavedata.json`
- **SaveManager** (`Unturned/Managers/SaveManager.cs`, 142 lines) exposes static `save()` orchestrating world saves. `[SOURCE: header + :42]`
- **Versioning:** save readers branch on embedded version bytes `[INFER — pattern not exhaustively verified; check the specific file's reader before touching any field]`. There is no migration framework to lean on; **field order in Block streams is the format**. Adding/removing/reordering a written field without bumping and handling the version breaks existing saves. `[INFER, high confidence]`
- **Gameplay config:** `ConfigData`/`ModeConfigData` (difficulty-mode config, e.g. `Provider.modeConfigData.Players.Detect_Radius_Multiplier`), loaded in `Provider.awake` (`LoadGameplayConfig`). `[SOURCE: Provider.cs:6729-6736, AlertTool.cs:62]`

## 11. Networking, dedicated server, plugins, and mod compatibility

- **Transport abstraction:** `SDG.NetTransport` assembly with pluggable implementations compiled into Assembly-CSharp: `NetTransport_SteamNetworkingSockets`, `NetTransport_SteamNetworking`, `NetTransport_SystemSockets`, `NetTransport_Loopback` (singleplayer), `NetTransport_UNetLLAPI`. `[SOURCE: folder listing]`
- **Message layer:** `NetMessaging/NetMessages.cs` + explicit handler classes per message (`ClientMessageHandler_*`, `ServerMessageHandler_*` — Accepted, Banned, ReadyToConnect, Authenticate, InvokeMethod, ValidateAssets, ThirdpartyAntiCheat, etc.). Enum order of message types is wire format. `[SOURCE: folder listing; enum-order claim INFER, high confidence]`
- **RPC layer:** `NetInvokable/` primitives (`ClientMethodHandle`, `ServerMethodHandle`, `NetId`, `NetIdRegistry`, attributes in `NetMethodAttributes.cs`) + **generated** `NetGen/NetInvokable/*_NetMethods.cs` for every replicated class (PlayerLife, ZombieManager, VehicleManager, …). Generated code is committed intentionally (`.gitignore:154-157`) and regenerated via **Window > Unturned > Net Gen** (`Assets/Editor/Assembly-CSharp-Editor/NetGen/NetGenTool.cs:12`). Never hand-edit `NetGen/` output. `[SOURCE]`
- **Rate limiting / anti-abuse:** `RateLimitedAction`, join/bad-packet rate limiters configured from `ConfigData` (`Provider.cs:6738-6740`). `[SOURCE]`
- **Server browser/identity:** server visibility (`ESteamServerVisibility`), `serverID`, host bans (`SDG.HostBans`), server-list curation (`ServerListCuration`). Also `Builds/Shared/Status.json` game version participates in client/server compatibility. Changing any of it risks polluting the public server browser or breaking version checks — off-limits. `[SOURCE: Dedicator.cs, Provider.cs:6744, 6877; compatibility semantics INFER]`
- **Plugins/modules:** `SDG.Framework.Modules.ModuleHook` loads third-party module DLLs from `Modules/`; the public surface of `SDG.Unturned` types **is the plugin API** (see deliberate obsolete-member retention in `Dedicator.cs:40-46`, `Player.cs:241-247`). Renaming/removing public members breaks the module ecosystem. `[SOURCE]`
- **Anti-cheat:** gated behind `WITH_THIRDPARTYAC` + dedicated message handlers; BattlEye executables are whitelisted binaries in `.gitignore:112-114`. Absent/limited in SDK builds — expected. `[SOURCE; runtime behavior RUNTIME]`
- **Safest test conditions** (in order): 1) Editor Play Mode (client, Steam-backed singleplayer); 2) Build Test standalone (development build); 3) editor-as-server via `-DedicatedServerInEditor` editor flag / command-line `+LANServer/{ID}` with a second local client `[SOURCE: Dedicator.cs:159; multi-instance workflow RUNTIME]`.

## 12. UI, debugging, and observability

- **Glazier**: Unturned's UI abstraction. `GlazierFactory.Create()` at boot selects a backend; three implementations live in `Glazier_IMGUI/`, `Glazier_uGUI/`, `Glazier_UIToolkit/`. The widget vocabulary is "Sleek" (`Unturned/Sleek/`, `ISleek*` types). UI code builds widget trees in C#; there are no scene-authored HUD prefabs to break. `[SOURCE: Setup.cs:50, folder listing; INFER on authoring model details]`
- **HUD/menus:** `PlayerUI` (`Unturned/Player/PlayerUI.cs`, MonoBehaviour) owns in-game UI incl. `PlayerLifeUI`, `PlayerDashboardUI` (+ Information/Inventory/Skills/Crafting), `PlayerPauseUI`, `PlayerDeathUI`, NPC UIs (`Unturned/UI/Player/`). Menus under `Unturned/Menu/` (`MenuUI`). `[SOURCE: file listing]`
- **Existing diagnostics:**
  - `UnturnedLog` → editor console / log files (`Logs` component; `Builds/Shared/Logs` locally). `[SOURCE + LOCAL]`
  - Dedicated-server console commands via `Commander`; `CommandDebug` prints UPS/TPS + `ZombieManager.tickingZombies.Count` + `AnimalManager.tickingAnimals.Count`. `[SOURCE: CommandDebug.cs]`
  - `RuntimeGizmos` — runtime line/sphere drawing used by alert debug defines. `[SOURCE]`
  - `CommandLineFlag`/`CommandLineValue` system; in the editor, flags are toggleable via EditorPrefs (`CommandLineFlag.applyEditorPreferencesToAllFlags`, `Setup.cs:27`) and an editor settings window exists (`Assets/Editor/Assembly-CSharp-Editor/Tools/EditorSettingsTool.cs`, menu Window/Unturned/Editor Settings). `[SOURCE]`
  - `SDG.Framework.Debug` inspectable-type helpers. `[SOURCE: file names]`
- **Ranked seams for a future local-only dev overlay** (no implementation yet):

  1. **New MonoBehaviour + IMGUI `OnGUI()` overlay, attached from a new module or a single new file** — reads `Player.LocalPlayer`, `Level.info`, `LightingManager`, `ZombieManager` statics. Safe: purely additive, client-only, no Sleek/Glazier coupling, trivially removable. Runs client-side; no save/multiplayer effect (read-only). Scope: 1 new file, 0–1 existing files modified (only if a bootstrap hook is needed). Perf: negligible if it avoids per-frame allocation. **Inspect first:** `PlayerDashboardInformationUI.cs` (position math), `CommandDebug.cs` (safe stats), `Player.cs:226-247` (LocalPlayer guard).
  2. **Sleek/Glazier widget added under `PlayerUI`** — native look, uses the game's own UI system. Slightly riskier: touches `PlayerUI` lifecycle (1–2 existing files), must follow Sleek idioms; still client-only/read-only. **Inspect first:** `PlayerUI.cs`, `PlayerLifeUI.cs`, `Unturned/Sleek/` basics.
  3. **Module (`Modules/` DLL implementing `IModuleNexus`)** — zero existing files modified; matches how real mods ship; but requires building an external assembly against Assembly-CSharp and understanding module load order. **Inspect first:** `ModuleHook.cs` (`findModules`, `loadModules`, `getRequiredModules`), `UnturnedNexus.cs` as the reference nexus.

  Zombie *alert/target state* is exposed via `Zombie` fields/methods of the region lists (`ZombieManager.regions[nav].zombies`); read-only access is feasible but verify field accessibility at implementation time `[RUNTIME]`.

## 13. Build and release tooling

- **Build Tool** (`Window > Unturned > Build Tool`, `Assets/Editor/Assembly-CSharp-Editor/Tools/BuildTool.cs`): buttons for full standalone platform builds, **Build Test** / **Build Test (Scripts Only)** (`BuildMethods.runBuild(...)` → gitignored `Builds/Test`), code docs, assembly hashing, SteamCmd runs, and dedicated-server SDK update. `[SOURCE]`
- **Bundle tools:** `BundleTool.cs`, `MasterBundleTool.cs` build asset bundles (`core.masterbundle` staging in `Builds/CoreAssetBundle`, per-platform copies — all gitignored). `[SOURCE: file names + .gitignore:22-34]`
- **CI:** `Jenkins.Editor` asmdef + `JenkinsBootstrapper` console app + `Build_Scripts/Jenkinsfile.txt`; `Assets/Editor/Jenkins.asset` is intentionally untracked. SDG-internal; do not run or "fix". `[SOURCE: .gitignore:159-171]`
- **Dev vs release:** development-build-only affordances are gated by `UNITY_EDITOR || DEVELOPMENT_BUILD` defines (e.g. `-ApplicationTargetFrameRate` parsing in `Dedicator.cs:122-129`). Release-only gates use `WITH_NOREDIST`/`!DEVELOPMENT_BUILD` (e.g. the Steam-install requirement applies to editor/dev/redist-free builds, `Provider.cs:6771`). `[SOURCE]`

## 14. Steam-backed and third-party dependency boundaries

See `Docs/U3_RUNTIME_AND_STEAM_DEPENDENCIES.md` for the full map. Summary:

- `SteamApps.GetAppInstallDir(304930)` → `Provider.steamAppInstallDirectory` (`SteamworksProvider.cs:112-120`). Consumers fall back to it for: core asset bundles (`Assets.cs:2323-2325`), vanilla maps (`Level.cs:1137-1141`), English localization (`Localization.cs:220-222`, `Provider.cs:6930-6932`), loading screens (`LoadingUI.cs:540-542`), economy `EconInfo.bin` (`TempSteamworksEconomy.cs:1880-1882`). `[SOURCE]`
- Editor Play Mode requires Steam running **and** Unturned installed, else it exits with a dialog. `[SOURCE: Provider.cs:6754-6783]`
- Workshop content arrives through `provider.workshopService` (client UGC subscription refresh at boot) and server workshop folders. `[SOURCE: Provider.cs:6844-6845, 6719-6727]`
- Intentionally absent proprietary pieces: ASPFP pathfinding (`WITH_ASPFP` off), third-party anti-cheat (`WITH_THIRDPARTYAC` off). `[SOURCE]`

## 15. High-value architecture landmarks

| Landmark | Location |
|---|---|
| Boot sequence | `Unturned/Loading/Setup.cs` |
| Platform/session god-object (client + server) | `Unturned/Provider/Provider.cs` (7,183 lines) |
| Client-vs-server switch | `Unturned/Provider/Dedicator.cs`; `Provider.isServer` |
| Module/plugin loader | `Framework/Modules/ModuleHook.cs`; nexus: `Unturned/UnturnedNexus.cs` |
| Asset database (.dat driven) | `Unturned/Bundles/Assets.cs`, `Bundles.cs`; `UnturnedDat` assembly |
| Level lifecycle + level discovery | `Unturned/Level/Level.cs`; `onLevelLoaded` event |
| Navigation abstraction (stubbed) | `Unturned/Level/UnturnedPathfinding.cs`, `LevelNavigation.cs` |
| Zombie simulation | `Unturned/Managers/ZombieManager.cs`, `Unturned/Zombies/Zombie.cs` |
| Noise/vision stimuli | `Unturned/Tools/AlertTool.cs` |
| Gun fire flow | `Unturned/Useable/UseableGun.cs` (server block at :912) |
| Asset-driven mod extension points | `Unturned/ModHooks/` (~45 components) |
| RPC/codegen | `NetInvokable/`, `NetGen/`, editor `NetGenTool.cs` |
| Save primitives | `Unturned/Files/ReadWrite.cs`, `PlayerSavedata.cs`, `ServerSavedata.cs`, `UnturnedPaths.cs` |
| UI abstraction | `GlazierFactory` + `Glazier_*` folders; widgets `Unturned/Sleek/`; HUD `Unturned/Player/PlayerUI.cs` |
| Build tooling | `Assets/Editor/Assembly-CSharp-Editor/Tools/BuildTool.cs` |

## 16. What not to touch first

1. **`NetGen/` generated files and anything in `NetMessaging`/`NetInvokable`** — wire protocol.
2. **Save read/write code** (`*Savedata`, Block/River field sequences) — corrupts existing saves.
3. **Public API surface of `SDG.Unturned`** (names, signatures, enum orders) — plugin/module ecosystem contract.
4. **`Provider.cs` initialization order** and the `Setup.Awake` sequence — everything depends on it.
5. **Steam identity/config**: `steam_appid.txt`, `Status.json` version block, server browser/visibility code, HostBans, anti-cheat handlers.
6. **`ZombieManager` tick/region internals** and the rechamber/shot-count logic in `UseableGun` (explicitly warned in code).
7. **Asset `.meta` files, GUIDs, and the `Assets/Game/Sources` content tree** — scene/prefab references break invisibly.
8. **Build/CI tooling** (`BuildMethods`, Jenkins, SteamCmd, `Build_Scripts/`).

## 17. Three ranked beginner-safe experiments

### Experiment 1 — Local development debug HUD/overlay (recommended first)
- **Learning goal:** scene lifecycle, manager statics, client/server split, safe read-only observation.
- **Inspect first:** `Player.cs:226-247`, `PlayerDashboardInformationUI.cs`, `CommandDebug.cs`, `Level.cs` (info/name), `ZombieManager.cs` (regions/tickingZombies), `Dedicator.cs`.
- **Shape:** one new MonoBehaviour rendering via `OnGUI()` (IMGUI), created only when `!Dedicator.IsDedicatedServer` and (ideally) only in editor/dev builds; displays player position, `Level.info.name`, time of day, zombie count, network role. Toggle via a `CommandLineFlag` or key.
- **Max existing files to modify:** 1 (a single hook point to instantiate it — or 0 if done as a module). All logic in new file(s).
- **Editor test:** open `GameStartup.unity` → Play → load a map in singleplayer → toggle overlay → verify values against the in-game map/dashboard.
- **Standalone test:** Build Test → run `Builds/Test` exe → same checks.
- **Multiplayer/save/plugin implications:** none if strictly read-only and local-only; never send network messages or write files.
- **Performance traps:** per-frame string allocation/`GUILayout` garbage — cache strings, update text at ~4 Hz, never `FindObjectsOfType` per frame.
- **Rollback:** delete new file(s) + revert the ≤1 touched file (git diff will show exactly two paths).
- **Stop and reassess if:** you need to touch `Provider.cs`, anything in `NetGen/`, or a `.unity`/`.prefab` file; or values require calling methods with side effects.

### Experiment 2 — Local-only firearm/noise telemetry
- **Learning goal:** the Useable/equipment system, server-vs-client execution of the same component, the AlertTool stimulus model.
- **Inspect first:** `UseableGun.cs:905-955`, `AlertTool.cs`, `ModHooks/UseableGunEventHook.cs`, `RuntimeGizmos.cs`, `ItemGunAsset` (`alertRadius` field).
- **Shape:** log/visualize each discharge: position, `alertRadius`, silenced-or-not, zombies-in-radius count *before* the alert resolves. Prefer subscribing via the existing mod-hook path or a small observer invoked next to the overlay from Experiment 1; alternatively enable `LOG_ALERTS`/`ALERT_SPHERE_GIZMOS` defines locally (do not commit).
- **Max existing files to modify:** 2 (`UseableGun.cs` only if a hook is truly unavailable; otherwise 0–1).
- **Editor test:** Play → equip a gun (spawn via cheats/sandbox) → fire silenced vs unsilenced → verify telemetry matches `AlertTool` behavior (silencer suppresses the alert).
- **Standalone test:** Build Test; confirm identical behavior; confirm no output on a `-DedicatedServerInEditor` run without a client.
- **Implications:** must not alter when/whether `AlertTool.alert` is called; read-only. No save/plugin impact. Multiplayer: observe-only; be aware the alert only happens where `Provider.isServer`.
- **Performance traps:** `getZombiesInRadius` allocations — reuse a static list exactly as `AlertTool` does; don't raycast per zombie.
- **Rollback:** delete new files; revert any define/hook change.
- **Stop and reassess if:** you find yourself reordering code inside the `Provider.isServer` block, or touching `SendPlayShoot`/rechamber logic.

### Experiment 3 — Tightly scoped PvE reaction experiment (only after 1 & 2)
- **Learning goal:** server-authoritative gameplay change through the existing stimulus seam.
- **Safe seam identified from source:** `AlertTool.alert(position, radius)` and asset-level `alertRadius` values; or an asset-driven `MobAlertSpawner`/`EffectSpawner` ModHook on a test effect — the engine already supports "make noise here" as data.
- **Shape:** e.g., a local-only test where an additional, clearly-labeled stimulus is emitted (say, on a thrown item impact) and zombies investigate — implemented as a new component or module calling the *existing* `AlertTool.alert(Vector3, float)`, never modifying `Zombie`/`ZombieManager`.
- **Max existing files to modify:** 2, and zero inside `Zombies/`, `Managers/ZombieManager.cs`, or `NetGen/`.
- **Editor test:** singleplayer map with zombies; emit stimulus; watch investigation behavior; confirm normal behavior when feature is off. Note: zombie *movement* fidelity is limited by the stubbed pathfinding — judge "reacted/turned/investigated", not path quality `[SOURCE: UnturnedPathfinding_Empty]`.
- **Standalone test:** Build Test singleplayer; then optional local LAN server + one client to confirm the stimulus only originates server-side.
- **Implications:** gameplay-affecting → keep behind a local flag, default off; no protocol/save changes (positions/radii are transient); plugin-safe if nothing public changes.
- **Performance traps:** emitting alerts every frame (rate-limit), large radii on dense maps.
- **Rollback:** flag off → delete files.
- **Stop and reassess if:** the idea requires new replicated state, new saved fields, or edits to zombie targeting internals — that is a different, much larger project.

## 18. Glossary

| Term | Meaning |
|---|---|
| **Provider** | Central static/session god-class: Steam services, connection state, config, player list. Also the name of the platform-services folder (`SDG.Provider`/`SDG.SteamworksProvider`). |
| **Dedicator** | Determines process role (client vs dedicated server); `IsDedicatedServer`. |
| **Module** | Unturned's mod-DLL plugin unit, loaded by `ModuleHook` from `Modules/`; entry interface `IModuleNexus`. |
| **Nexus** | A module's entry object (`UnturnedNexus` is the game's own). |
| **Useable** | The equipped-item behavior base class (guns, melee, consumables) under `Unturned/Useable/`. |
| **Sleek / Glazier** | Sleek = UI widget vocabulary (`ISleek*`); Glazier = pluggable UI backend (IMGUI/uGUI/UIToolkit). |
| **Block / River** | Custom binary serialization containers used for saves and level files (`ReadWrite`). |
| **.dat assets** | Text-defined game content (items, vehicles, etc.) parsed by `UnturnedDat`, registered by type name via `Assets.assetTypes`. |
| **Masterbundle** | Packed Unity asset bundle (`core.masterbundle`) providing meshes/textures/prefabs; built by editor tools, shipped with the game (Steam copy provides it for the SDK). |
| **NetGen** | Generated RPC marshaling code (`*_NetMethods.cs`), regenerated via Window > Unturned > Net Gen. |
| **NetId** | Network identity handle registered in `NetIdRegistry`, used by generated RPCs. |
| **Nav (byte)** | Index of a navmesh region; `255` = not on any navmesh (`player.movement.nav`, `ZombieManager.regions[nav]`). |
| **ZombieRegion** | Per-navmesh-bound zombie population bucket with difficulty/aggro settings. |
| **ASPFP** | A* Pathfinding Project — licensed navigation plugin, excluded from the SDK (`WITH_ASPFP`). |
| **LiveConfig** | Remotely-fetched configuration assembly/system (`LiveConfig.Refresh()` at boot). |
| **HostBans** | SDG-run server-list ban/curation system (`SDG.HostBans`). |
| **Build Test** | Editor build producing a runnable local Windows copy in gitignored `Builds/Test`. |
