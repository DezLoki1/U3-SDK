# CLAUDE.md — U3-SDK (Unturned source SDK)

Unity **2022.3.62f3** project containing Unturned's C# source. Entry scene: `Assets/GameStartup.unity`. Steam AppID 304930.

**Read before any non-trivial change:** `Docs/U3_ARCHITECTURE_AUDIT.md`, `Docs/U3_RISK_REGISTER.md`, `Docs/U3_RUNTIME_AND_STEAM_DEPENDENCIES.md`.

## Known-good baseline (do not "fix" what isn't broken)

- Project imports in Unity 2022.3.62f3; `GameStartup.unity` runs in Play Mode; `Window > Unturned > Build Tool > Build Test` builds and the exe in gitignored `Builds/Test` runs.
- **Steam must be running and retail Unturned installed** — Play Mode exits otherwise (`Provider.cs`). Core bundles, vanilla maps, localization, loading screens load from the Steam install (`Provider.steamAppInstallDirectory`). This fork tracks no maps; that is normal.
- Zombie pathfinding is intentionally stubbed (`UnturnedPathfinding_Empty`; A* plugin not licensed for the SDK). Degraded AI navigation, missing NavmeshCut, simplified visuals, and absent anti-cheat are **expected**, not setup failures.
- `Build_Scripts/`, `Builds/Test/`, root `*.csproj`/`*.sln`, `Library/`, `Temp/`, `Logs/`, `UserSettings/` are generated and gitignored. Never delete, commit, or "repair" them.

## Layout & landmarks

- `Assets/Runtime/Assembly-CSharp/` — the game (~1,650 files, namespace `SDG.Unturned`). Subfolders: `Unturned/` (gameplay), `Framework/` (modules/devkit), `NetMessaging|NetInvokable|NetGen/` (protocol), `Glazier*/` (UI backends), `SteamworksProvider/`.
- `Assets/Runtime/<X>/` asmdef assemblies: SystemEx, UnityEx, UnturnedDat, SDG.NetPak, SDG.NetTransport, SDG.Glazier, SDG.HostBans, LiveConfig. `Assets/Editor/` — editor tools (BuildTool, NetGenTool, Jenkins, SteamCmd).
- Boot: `Setup.Awake()` (`Unturned/Loading/Setup.cs`) → Dedicator → Logs → ModuleHook (mod DLLs from `Builds/Shared/Modules`) → `Provider.awake/start` (Steam init, 7k-line god-class) → Glazier UI → pathfinding stub. Managers are static singletons on one `DontDestroyOnLoad` GameObject in the boot scene.
- Levels: `Level.load()` → scene "Game" → `Level.onLevelLoaded` event re-initializes managers. Runtime data root `ReadWrite.PATH` = `Builds/Shared` in editor (saves: `/Worlds`, `/Servers/{id}`, `/Players/{steam}_{char}/{level}`, `/Cloud`).
- Roles: `Dedicator.IsDedicatedServer`, `Provider.isServer`. `Player.LocalPlayer` **throws on dedicated server**.
- Noise/AI stimulus seam: `AlertTool.alert(...)` (`Unturned/Tools/AlertTool.cs`); guns already call it server-side (`UseableGun.cs:938`). Asset-driven extension components live in `Unturned/ModHooks/`.

## Build & test routes

1. Editor: open `GameStartup.unity`, press Play (client/singleplayer).
2. Standalone: `Window > Unturned > Build Tool > Build Test` (or Scripts Only) → run from `Builds/Test`.
3. Server-ish testing: editor flag `-DedicatedServerInEditor` (via Window > Unturned > Editor Settings / CommandLineFlag prefs) or build with `+LANServer/{ID}`, optionally `-OfflineOnly`.

Safe commands: `git status`, `git diff`, `git log`, `git ls-files`, ripgrep/reads anywhere in tracked source.
Forbidden without explicit user request: `git reset|clean|restore|rebase|merge|stash|checkout -- .`, `git push`, remote changes, package/Unity upgrades, regenerating the solution, running Net Gen, deleting generated folders.

## Rules for all future sessions

1. **≤5 existing files per feature** — modifying more requires first explaining why every file is necessary. Prefer new files; best of all, a module under `Builds/Shared/Modules`.
2. **Early experiments are local-only**: no network messages, no writes to save files, default-off toggles.
3. **Never change without explicit user approval:** save formats (Block/River field order), network protocol (`NetGen/`, `NetMessaging/`, `NetInvokable/`, enum orders), Steam integration/build identity (`steam_appid.txt`, `Status.json`, server browser/HostBans/anti-cheat), or public plugin-facing APIs (renames/removals/signature changes — keep obsolete shims like the codebase does).
4. **Preserve `.meta` files.** New assets ship with their `.meta`; never edit GUIDs; never hand-edit `.unity`/`.prefab` YAML.
5. **Do not move or rename Unity assets casually** — scene/prefab references break silently.
6. **Label assumptions** (`[INFER]`/`[RUNTIME]`) and validate them in source before acting on them.
7. **Before implementing:** present a patch plan (files, risks per `Docs/U3_RISK_REGISTER.md`, Unity test steps, rollback steps) and wait for approval.
8. **After implementing:** list every changed file and exact Unity test steps (editor sequence + Build Test sequence).
9. **Never `git add -A` blindly** after Unity or an agent session — Unity touches metas and generated files; stage by explicit path only, and inspect `git status` first.
10. Match `.editorconfig` style (tabs, Allman braces, existing comment density). Server-authoritative logic goes inside `Provider.isServer` guards; editor-only APIs inside `#if UNITY_EDITOR`.
