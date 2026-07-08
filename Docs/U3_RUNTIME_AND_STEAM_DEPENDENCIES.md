# U3-SDK Runtime & Steam Dependencies

Audit date: 2026-07-07. Companion to `Docs/U3_ARCHITECTURE_AUDIT.md` (same confidence labels: `[SOURCE]`, `[LOCAL]`, `[INFER]`, `[RUNTIME]`).

**Core fact:** the SDK repository deliberately does **not** contain everything the game needs. The installed Steam copy of Unturned (AppID 304930) is a first-class runtime dependency. Treat "content loaded from Steam install" as designed architecture, never as a broken clone.

## 1. Source-controlled vs external content

| Content | Where it comes from | Confidence |
|---|---|---|
| All C# gameplay/editor source | This repo (`Assets/Runtime`, `Assets/Editor`) | `[SOURCE]` |
| Scenes, shaders, models, textures, animations | This repo (`Assets/Game/Sources`, `Assets/Runtime/...`) | `[LOCAL ls]` |
| Game version + achievement metadata | `Builds/Shared/Status.json` (tracked) | `[SOURCE]` |
| Mod identity | `Builds/Shared/ModInfo.json` (tracked) | `[SOURCE]` |
| Core asset bundles (`core.masterbundle` etc.) | Local `Bundles/` if built, else **Steam install** `<install>/Bundles` | `[SOURCE: Assets.cs:2323-2325]` |
| Vanilla maps (PEI, Washington, …) | `Builds/Shared/Maps` if present (this fork: absent), else **Steam install** `<install>/Maps` | `[SOURCE: Level.cs:1137-1141]`, `[LOCAL]` |
| English localization | Repo `Localization/` path if present, else **Steam install** | `[SOURCE: Localization.cs:220-222, Provider.cs:6930-6932]` |
| Loading screen images | Local path, else **Steam install** `<install>/LoadingScreens` | `[SOURCE: LoadingUI.cs:540-542]` |
| Economy metadata (`EconInfo.bin`) | Local file, else **Steam install** | `[SOURCE: TempSteamworksEconomy.cs:1880-1882]` |
| Workshop mods/maps | Steam Workshop via `provider.workshopService` (client) and `/Workshop` folders (server) | `[SOURCE: Provider.cs:6844-6845, 6719-6727]` |
| Player/world saves | Generated at runtime under `Builds/Shared` (editor) — gitignored | `[SOURCE: UnturnedPaths.cs:24, .gitignore:53-57]` |
| ASPFP pathfinding, third-party anti-cheat | **Intentionally absent** (license) — compile-time defines `WITH_ASPFP`, `WITH_THIRDPARTYAC` are off | `[SOURCE: UnturnedPathfinding.cs, Dedicator.cs:65-68]` |

## 2. How the Steam install is discovered

1. `Provider.awake()` constructs `SDG.SteamworksProvider.SteamworksProvider` and calls `intialize()` (sic). `[SOURCE: Provider.cs:6756-6758]`
2. `WriteSteamAppIdFileAndEnvironmentVariables()` ensures `steam_appid.txt` (304930) is visible to the Steam API. `[SOURCE: Provider.cs:6580, 6756; file contents LOCAL]`
3. `SteamworksProvider` calls `SteamApps.GetAppInstallDir(appId)`; result stored in `SDG.Unturned.Provider.steamAppInstallDirectory` (nulled with a logged warning if the directory doesn't exist). `[SOURCE: SteamworksProvider.cs:112-120]`
4. Consumers use the pattern *"if local path missing and `steamAppInstallDirectory != null`, use Steam path"* (see table above).

### What happens when Steam is not running
- Steam init throws → logged (`"Steam init exception:"`) → `QuitGame`. In the editor, a dialog appears ("Please ensure you have Steam running!") and Play Mode exits. `[SOURCE: Provider.cs:6760-6768]`

### What happens when Unturned is not installed
- In editor / development / non-`WITH_NOREDIST` builds: error dialog ("Please ensure you have Unturned installed on Steam!") and Play Mode exit / `QuitGame`. `[SOURCE: Provider.cs:6771-6783]`
- Dedicated servers write their own appid/env vars and init Steam GameServer APIs instead; `-OfflineOnly` (`Dedicator.offlineOnly`) skips backend/workshop queries for LAN use. `[SOURCE: Provider.cs:6576-6589, Dedicator.cs:63]`

## 3. Dependency timing

| Phase | Needs Steam running? | Needs Unturned installed? |
|---|---|---|
| Opening the Unity project / editing code | No | No `[INFER — no editor-time Steam calls found in import path; LOCAL experience agrees]` |
| **Entering Play Mode** | **Yes** | **Yes** (client path) `[SOURCE: Provider.cs:6754-6783]` |
| Build Test (producing the build) | No for compilation `[INFER]`; SteamCmd tooling exists for SDG release flows only | No |
| **Running the built exe** | **Yes** (dev builds) | **Yes** (dev builds) `[SOURCE: same gate, WITH_NOREDIST off]` |
| Dedicated server run | Steam GameServer init required (LAN mode with `-OfflineOnly` reduces backend use) | Maps/bundles must come from somewhere: local folders or Steam install `[SOURCE + RUNTIME for exact server fallback behavior]` |

## 4. Workshop / mod loading assumptions

- Client: `provider.workshopService.refreshUGC()` + `refreshPublished()` at boot enumerate subscribed items; localization workshop items can even supply the UI language. `[SOURCE: Provider.cs:6844-6845, 6903-6907]`
- Server: expects `/Workshop/Content` and `/Workshop/Maps` folders under `/Servers/{id}` (created at boot). `[SOURCE: Provider.cs:6719-6727]`
- Code modules (plugins) load from `<ReadWrite.PATH>/Modules` (`Builds/Shared/Modules` in editor — exists locally, `[LOCAL]`) via `ModuleHook.findModules()` → `.module` configs + DLLs. `[SOURCE: ModuleHook.cs:494-597]`

## 5. Known/likely proprietary or removed pieces

| Piece | Evidence | Symptom in SDK |
|---|---|---|
| A* Pathfinding Project (ASPFP) | `UnturnedPathfinding.cs` header comment: "ASPFP implementation not included in SDK release" | Zombie/animal navigation degraded; `NavmeshCut`-style behavior absent; navmesh baking tools inert `[RUNTIME for exact visuals]` |
| Third-party anti-cheat (BattlEye) | `WITH_THIRDPARTYAC` define; `ClientMessageHandler_ThirdpartyAntiCheat`; `BEService*.exe` exceptions in `.gitignore:112-114` | No anti-cheat in local builds; connecting to protected retail servers is out of scope |
| SDG release/CI infrastructure | Jenkins assemblies, SteamCmd tools, `Build_Scripts/` | Buttons/menus exist but are for SDG's pipeline; ignore |
| Pre-rendered item icons / econ previews | `.gitignore:65-77` | Icons render on demand instead of shipping as PNGs |

## 6. Do not panic if…

- …zombie pathfinding looks dumb, zombies walk through odd routes, or navmesh-cut behavior is missing — ASPFP is intentionally stubbed (`UnturnedPathfinding_Empty`). `[SOURCE]`
- …the console warns about missing optional components, simplified water, or broad/tinted selection highlighting — visual third-party niceties are not all in the SDK. `[INFER from task briefing + license-driven removals; specific warnings RUNTIME]`
- …a `Build_Scripts/` folder, `Builds/Test/`, `Library/`, `Temp/`, `Logs/`, `UserSettings/`, or root `*.csproj`/`*.sln` files appear or churn — all generated, all gitignored. `[SOURCE: .gitignore]`
- …`Builds/Shared` accumulates `Worlds/`, `Cloud/`, `Logs/`, `Sandbox/` content after playing — that's your local save data root. `[SOURCE: UnturnedPaths.cs:24]`
- …there are no maps inside the repo — vanilla maps come from the Steam install in this fork. `[LOCAL + SOURCE: Level.cs:1137-1141]`
- …anti-cheat–related warnings appear or third-party AC features no-op — `WITH_THIRDPARTYAC` is off. `[SOURCE]`

## 7. Actually investigate if…

- …Play Mode exits immediately **with Steam running and Unturned installed** — check the editor log for the `SteamworksProvider` warning about a non-existent install directory. `[SOURCE: SteamworksProvider.cs:117-120]`
- …`Assets.cs` logs that core bundles can't be found in either location — the Steam copy may be corrupt/moved (verify integrity in Steam; do not copy files into the repo).
- …scripts fail to compile on a clean clone — that is not expected; suspect Unity version mismatch (must be 2022.3.62f3) before anything else.
- …`git status` shows modified `.meta` files or scene/asset churn you didn't cause — stop, review, and revert-by-intent before any commit (an agent or editor action likely touched assets).
- …the game version in `Builds/Shared/Status.json` differs from the installed Steam game in a way that breaks content parsing — SDK/retail drift is possible; verify against upstream. `[INFER]`

## 8. Never commit these local/generated items

- `Library/`, `Temp/`, `Logs/`, `obj/`, `UserSettings/`, `.vs/`
- Root `*.csproj`, `*.sln` (Unity-generated; gitignored on purpose)
- `Builds/Test/`, `Builds/CoreAssetBundle/`, per-platform `Builds/*/Bundles`, `Builds/*/Unturned_Data`
- `Builds/Shared/{Worlds,Servers,Cloud,Logs,Sandbox,Screenshots}` (save data / local testing content)
- `Build_Scripts/*` except the whitelisted `Jenkinsfile.txt` / certificate / `JenkinsBootstrapper.exe`
- `Preferences.json`, `Execute.config`, `sysinfo.txt`, `TestInventory.dat`
- `Assets/TextMesh Pro/` (auto-imported package resources)
- Anything copied out of `steamapps/` — Steam-owned content must never enter the repo (license + repo hygiene).

## 9. Safe troubleshooting steps (no source edits, no binary copying)

1. Confirm Unity version `2022.3.62f3` (Unity Hub).
2. Confirm Steam is running and logged in; confirm Unturned installed; use Steam's "Verify integrity of game files".
3. Read `Logs/` (Unity editor log links exist at repo root) and `Builds/Shared/Logs` for `UnturnedLog` output.
4. Re-open the project so Unity regenerates `Library/` if imports look wrong (slow but safe; do not delete anything else).
5. For dedicated-server experiments use `-DedicatedServerInEditor` (editor flag) or `+LANServer/{ID}` and `-OfflineOnly` on a build. `[SOURCE: Dedicator.cs:63,159]`
6. If a doubt remains about whether something is a real defect: reproduce it in an untouched clone/branch before concluding anything.
