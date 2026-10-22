# Hydroneer (2020)

| | |
|---|---|
| Engine | Unreal Engine 4.26 (`4.26.2-15973114+++UE4+Release-4.26`), PhysX 3, Voxel Plugin Pro, EasyMultiSave |
| Versions tested | Steam app 1106840, build 23218978; `Mining-Win64-Shipping.exe` SHA-256 `DF27B86E4564100FA792464EA625F64C96C4D155C6C911486FE3769C8D5570B1` |
| Projects | Official split-screen / Remote Play co-op; CoopNet, a host/join prototype (this write-up, not published yet). Both in [DIRECTORY.md](../../DIRECTORY.md) |
| QA level | official mode: community-reports; CoopNet: automated (two real game clients on one PC, bot-driven, loopback only) |
| Last updated | 2026-10-22 |

## Where things stand

Hydroneer ships co-op only as split screen: one game, several local players, and online play by
streaming that screen with Steam Remote Play Together. No host/join mod existed (searched
2026-10-22). CoopNet is a working prototype of real host/join co-op: two copies of the game, each
player on their own screen, the host's world authoritative. It is not something players can
install yet: it runs only through the test harness, over loopback, and syncs players, items,
carrying, digging and runtime-spawned items, but not money, shops, machines, quests or saves.

## Existing work

| Project | What it is | Source |
|---|---|---|
| Official multiplayer (2.0 update; existed by 2022-08) | Split screen; online via Steam Remote Play Together (the remote player sees a stream of the host's split screen, needs no copy of the game). Developers say they "tried all avenues" for true multiplayer and blame the physics load | [TechRaptor guide](https://techraptor.net/gaming/guides/hydroneer-multiplayer-guide), [2.0 update](https://techraptor.net/gaming/news/hydroneer-20-multiplayer-update-is-out-now), [Steam thread](https://steamcommunity.com/app/1106840/discussions/0/3458220484817654192/) |
| [Hydroneer-P2-Keyboard](https://github.com/codeopen0-cpu/Hydroneer-P2-Keyboard) | Keyboard and mouse for split-screen player 2 (pushed 2026-06-07). Not host/join | GitHub |
| [HydroneerAPMod](https://github.com/Thedragon005/HydroneerAPMod) | Archipelago randomizer: item sync only (pushed 2026-09-26). Not host/join | GitHub |
| Modding base | HML (Hydroneer Mod Loader) for Workshop mods; [Hydroneer-Modkit](https://github.com/Buckminsterfullerene02/Hydroneer-Modkit) and [Hydroneer-Template](https://github.com/Buckminsterfullerene02/Hydroneer-Template) (UE 4.26 editor projects) | GitHub, [Steam](https://steamcommunity.com/app/1106840/discussions/0/3729575504107129027) |

Players report up to 2 or up to 4 local players on PC; sources disagree
([Steam thread](https://steamcommunity.com/app/1106840/discussions/0/6756004301608937453)).

## How the game works (for co-op)

- **Framework:** `GM_Hydroneer_C` (game mode), `BP_GameController_C`, `BP_Character_HydroGuy_C`,
  game C++ module `/Script/Mining` (`UCHydroneerLibrary`, `UCModdingSubsystem`). The main menu runs
  inside the world level `L_GameWorld`.
- **Netcode:** `GameNetDriver` = `IpNetDriver` is configured, but no game Blueprint has a replicated
  property or RPC. Engine replication alone syncs nothing.
- **Players:** `GM_Hydroneer_C:AddLocalPlayer()` (and `AddPlayer(..., isRemote)`, used by Remote
  Play) adds a full second player. `GameplayStatics:SetForceDisableSplitscreen(true)` hides that
  player's view on the host; it keeps working. This is the slot a remote player drives.
- **Input:** movement is `BP_GameCharacter_C:InpAxisEvt_MoveForward_*` / `MoveRight_*`; actions
  are generated `InpActEvt_<Action>_*` pairs on the controller (lower index = key down: Pickup
  12/13, LMB 5/6). Calling them from a mod runs the game's own logic for that player (pickup,
  drop, digging, tools) with no screen needed.
- **Items:** `BP_ParentItem_C` with `BeingCarriedBy` and `ProxyItem` (the held copy, created on
  pickup). Shovels dig through `BP_GameController_C:MoveDirt(PickUp?, Radius)` on voxel worlds
  (`BP_HydroVoxelWorld_C`; a new game can dig in Ember Cradle). There is no "add dirt at a point"
  function; `RemoveDirtByLocation(Location, Radius)` exists.
- **Saves and settings:** EasyMultiSave slots (`Level.sav` ~21 MB, mostly voxel data, plus
  `Player.sav`, `SaveData.sav`). `-userdir=` moves saves and engine configs. The game's
  `GetAppdataLocalDir` still returns the real `%LOCALAPPDATA%`.
- **Trap for testers:** the **first launch of every new copy** of the game rewrites the real
  `%LOCALAPPDATA%\Mining\Saved\Config\WindowsNoEditor\Settings.ini` once (`r.setres` windowed to
  fullscreen), even with `-userdir`; later launches of the same copy do not. The writer is unknown.
  Hash the real folder before and after every run and restore from a backup.
- **Tools used:** UE4SS. 3.0.1 (stable) works for probing, but its Lua cannot spawn actors
  (`FTransform` parameters do not convert). The experimental build `v3.0.1-1181-g83330635`
  (SHA-256 of `UE4SS_v3.0.1-1181-g83330635.zip`:
  `b9a166a756605b21058a64cde93bcaab1a855dba3553e5c7e6ce0fa8fdfad7b9`, matches the GitHub digest)
  adds `UWorld:SpawnActor`, `LoopInGameThreadWithDelay` and `LoopInGameThreadAfterFrames`. CoopNet
  needs it.

## What we changed

CoopNet is a UE4SS Lua mod running in both copies. Transport for now: the game appends lines to a
file and a small Python relay sends them by UDP (UE4SS Lua has no sockets).

- **Host authority:** the client's world-changing key mappings (Pickup, LMB, RMB, Rotate*,
  VehicleKey, BuildPlace*, ...) are blanked. Keys are polled with `IsInputKeyDown` and only sent.
  The host replays them on its hidden player 2, so pickups, drops and digging run only on the host.
- **Movement:** the client sends move axes and its look rotation. The host replays the axes
  **every frame** (`LoopInGameThreadAfterFrames`; on the ~30 Hz game-thread timer, player 2 walked
  slower than real input and the client was pulled back up to 74 cm). The host acks the last input
  line it applied; the client compares host P2 with its own position at that line and blends
  (snap over 4 m), so latency does not count as error.
- **Host player on the client:** the client's own second local player, movement off, glides to
  the host's position; the host's velocity is written into it so the walk animation plays.
- **Items:** host sends moved items (6 per tick round robin plus carried and recently moved items
  every tick), all items again every ~4.5 s in full 12-item lines, and skips held proxies. The
  client turns physics off on items it places.
- **Runtime items:** every new non-proxy item on the host gets a net id (`N1`, ...) and is spawned
  on the client by class path with `UWorld:SpawnActor`. An item whose root component becomes
  invalid is sent as gone; the client destroys its copy. (After `K2_DestroyActor`, UE4SS still
  reports `IsValid()` and `FindAllOf` lists the actor; the root component is the reliable signal.)
- **Carrying, shovel dirt, digging:** the client mirrors the host's decisions with the game's own
  functions: `PickUpItem` / `DropItem` for its player, the shovel's `Dirt` mesh visibility, and
  `RemoveDirtByLocation` at the host's dig point (`MoveDirt(false)` from the same aim for dumps).
- **Fixed on the way:** Lua `LoopAsync` + `ExecuteInGameThread` every tick crashed the
  experimental UE4SS ("Ref was not function", then an access violation in UE4SS); its own timers
  work. A client-side physics guard that only read component state left the client's player
  nearly unable to walk (17 cm/s); it is off. Mechanism unknown.

## QA

Harness (two copies on one PC, both with saves redirected and the owner's real data hashed
before and after every run, slots taken with [tools/gameslot](../../tools/gameslot/README.md)):
bots in both games walk, pick up, carry, dig, dump and drop. The host spawns items through the
game's own `SpawnInitialItems()` and destroys one. Real key presses (E, a held W) go to the client
window after a foreground-window check. At the end both games write every item's position, and a
script compares them.

Last run, 2026-10-22 (Steam build 23218978, UE4SS v3.0.1-1181):

| Check | Result |
|---|---|
| Items, host vs client | 564 vs 564, all matched, max error 0.1 cm |
| Runtime items (8 from `SpawnInitialItems`) | all 8 spawned on the client within 1 cm; the destroyed one removed |
| Client player vs host P2 (reconciled), whole run | worst 19.4 cm; real held W: 409 cm/s, worst 18 cm |
| Digging and dumping | ground height equal on both sides (145.7 / 194.5) |
| Client local game calls | only the mirrors (pickups, drops, digs); a real E press ran no local `TryPickupItem` |
| Host work per tick | avg 0.44 ms, max 3 ms, 0 of 3,392 ticks over 4 ms |
| Transport | 0 input lines lost (loopback UDP, under 1 ms) |

Session-by-session numbers, failed attempts and dead ends are in [NOTES.md](NOTES.md).

## How to play

Not yet. There is no release, no installer and no internet transport; the mod and harness live in
the author's workspace and are not published.

## Known issues

- Only tested with both copies on one PC over loopback. No real network, latency or packet loss.
- Not synced: money, shops, machines, quests, vehicles, inventories inside containers, saves,
  late join. The client starts its own new game, so both worlds only match at the start because
  both are new games.
- Walking right after a pickup, while looking steeply down, is slow on both sides (not a sync
  error; cause unverified).
- The client can rest up to 10 cm from the host's position (the correction threshold).
- Needs the experimental UE4SS build; the stable 3.0.1 cannot spawn items from Lua.
- The first launch of a new game copy rewrites the real `Settings.ini` (see the trap above).

## Next steps

1. Late join and saves: send the host's EasyMultiSave slot to the client at join, instead of two
   new games.
2. Money, shops and machines (host-side state the client must mirror).
3. A real transport: a UE4SS C++ mod with in-process UDP, or Steam P2P through the game's SteamCore
   plugin, plus network chaos tests ([playbook/network-chaos-testing.md](../../playbook/network-chaos-testing.md)).
4. Publish the mod and harness in their own repo, then link it here and in the directory.
5. Find the first-launch `Settings.ini` writer.
