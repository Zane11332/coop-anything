# Hydroneer: co-op notes

Goal: true online co-op with host and join (each player on their own PC and screen),
not the official split-screen / Steam Remote Play Together mode.

## Game identity (rule 5)

| Field | Value |
|---|---|
| Store | Steam, app 1106840 |
| Version / build id | Steam build 23218978 (in-game version string not yet read) |
| `Mining\Binaries\Win64\Mining-Win64-Shipping.exe` SHA-256 | `DF27B86E4564100FA792464EA625F64C96C4D155C6C911486FE3769C8D5570B1` |
| `Mining.exe` (launcher) SHA-256 | `B56C45269DC4B36A86AFBD6C5934D0C30F9DCFDC09D58C07810847B81D4F6B70` |
| Engine | Unreal Engine 4.26 (`++UE4+Release-4.26` in the exe), PhysX 3 |
| OS | Windows 64-bit |
| Install size | 13.8 GB, 30 files, two paks (`pakchunk0`, `pakchunk1001`) |

## Phase 3 session 1 (2026-10-22): UE4SS probe on a test copy

Setup, all actually run:
- Test copy made with robocopy; file count, sizes and both exe hashes match the real install.
- UE4SS **v3.0.1** (latest stable, 2024-02-14), asset `zDEV-UE4SS_v3.0.1.zip`, 23,810,106 bytes,
  SHA-256 `af777d2af33fce4d520de35a8665e417376bcc530fec647778012ce6cdf51b2b`. Upstream publishes no
  hash for v3.0.1 (GitHub digests exist only on the moving `experimental-latest` tag), so this
  hash is the one we computed; size matches the release API.
- Only `ConsoleEnablerMod` (read: creates the UE console on F10), `Keybinds` and our own
  read-only `CoopProbe` Lua mod enabled. `steam_appid.txt` (1106840) next to the shipping exe so
  Steam does not relaunch the real install. Started `Mining-Win64-Shipping.exe` directly with
  `-userdir=<workspace> -windowed -log`. UE4SS found GUObjectArray and ran fine on this build.

### Observations (from the probe log and dumps)

- `GameEngine.NetDriverDefinitions`: `GameNetDriver` = `/Script/OnlineSubsystemUtils.IpNetDriver`,
  `DemoNetDriver`. No `NetDriver` instance at the menu (standalone).
- Framework: GameInstance `BP_GameInstance_C` (C++ base `UCGameInstance`), GameMode
  `GM_Hydroneer_C`, GameState `BP_GameState_C`, PlayerController `BP_GameController_C`,
  pawn `BP_GameCharacter_C` / `BP_Character_HydroGuy_C`. The main menu runs inside
  `/Game/Levels/L_GameWorld` (55 pawns at the menu, mostly `BP_NPC_C`).
- **No replication in game Blueprints**: no `OnRep_`, `Server*`, `Multicast*` or `Client*`
  functions in any game class header. Engine replication alone would sync nothing.
- `GM_Hydroneer_C` has `AddPlayer(FSteamRemotePlaySessionConnected Data, bool isRemote, out
  ABP_GameController_C*)`, `AddRemotePlayer`, `AddLocalPlayer`, `RemovePlayer`, `RemoveLocalPlayer`,
  `SpawnHydroguy(Transform, Controller)`, `MultiplayerStart`, `ContinueGame(Creative, CloudSaving,
  ClearResources, Slot)`, `NewGame`. `FSteamRemotePlaySessionConnected` is just `int32 SessionID`.
- Plugins of interest: `OnlineSubsystemSteam`, **SteamCore** (Blueprint wrappers for Steam
  matchmaking/lobbies, `Networking`, sessions), **EasyMultiSave** (`EMSActorSaveInterface`,
  async save/load), **Voxel** (VoxelPro terrain; a Voxel class exposes `StartServer(IP, Port)` and
  `ConnectToServer(IP, Port)`), `AutoSettings`, `PakLoader`, `EasyFileSystem`.
- Game C++ module `/Script/Mining`: `UCGameEngine`, `UCGameInstance`, `UCHydroneerLibrary`
  (`GetSaveDirectory`, `GetAppdataLocalDir`, `BackupSave`, `RenameSave`, ...),
  `UCModdingSubsystem` (Workshop mods), `ULocalCableComponent`.
- `SaveGame_GameData_C` only holds `CurrentGameType`, `CloudSave`, `CreativeMode`, `CanAchieve`;
  world contents go through EasyMultiSave.
- 7,939 `StaticMeshComponent`s at the menu, 0 simulating physics (menu state, not a loaded world).
- **`-userdir` does not isolate everything.** Engine configs, logs and the asset-registry cache
  went to the redirected dir, but the game still rewrote
  `%LOCALAPPDATA%\Mining\Saved\Config\WindowsNoEditor\Settings.ini` (`r.setres=...w` to `...f`).
  Restored from backup and verified by hash. Saves were not touched in this run (no world loaded).
  `UCHydroneerLibrary.GetAppdataLocalDir` explains the hard-coded path.

### Derived

- Saves very likely ignore `-userdir` too (same library). Do not load or save a world in a test
  copy until saves are isolated.
- Host side can create a real player slot with `GM_Hydroneer_C:AddPlayer(..., isRemote=true)`,
  the same path Remote Play uses. That gives a controller and pawn the host world fully supports
  (tools, inventory, machines), driven by our own network input instead of Remote Play.
- For the joining player to render on their own PC they still need a local copy of the world:
  host save sent on join (EasyMultiSave files), then host-authority state sync for players,
  loose items (physics), voxel edits, machines, shops/money. Voxel's own `StartServer` /
  `ConnectToServer` may cover terrain edits: untested.

## Phase 3 session 2 (2026-10-22): isolating saves, first world load

### Save isolation (all runs below actually ran)

- `SaveRedirect` Lua mod: `RegisterHook` post-callbacks on `/Script/Mining.CHydroneerLibrary:
  GetAppdataLocalDir` and `:GetSaveDirectory` rewrite any path under the real `%LOCALAPPDATA%` to a
  workspace root (from env `HYDRO_COOP_APPDATA`); it also checks AutoSettings'
  `USettingsManager.IniFilename`. A post-callback's return value overrides a native UFunction's
  return in UE4SS v3.0.1 (changelog, [UE4SS #366](https://github.com/UE4SS-RE/RE-UE4SS/pull/366)).
  Verified: `GetAppdataLocalDir()` returns the workspace root after the hook.
- Launch with `-userdir=<workspace root>\Mining` so the engine's own paths and the hooked ones
  agree. A wrapper script snapshots the real folder (SHA-256 plus mtime) before each run and
  compares it after, restoring changed files from the backup.
- Observed without any hook: `GetSaveDirectory` (native and `BP_GameController`),
  `USettingsManager.IniFilename` and EasyMultiSave all follow `-userdir`. Only `GetAppdataLocalDir`
  returns the real `%LOCALAPPDATA%`. The game never called `GetAppdataLocalDir` through reflection at
  the menu or during a new game in these runs (only our probe did).
- EasyMultiSave `GetSortedSaveSlots` returned 0 slots in the test copy (the owner's real saves did
  not show up).
- Real folder unchanged after 5 runs (3 menu-only with the redirect, 1 menu-only without the
  redirect and with the dumpers on, 1 new game with save).
- **Unexplained:** in session 1's first run the real `Settings.ini` was rewritten once
  (`r.setres` `w` to `f`). Not reproduced in any later run, including an exact repro of that run's
  setup. Shipping build writes no engine log, so the cause is `unknown`. Keep the hash guard on.

### First world (new game, test copy)

- `GM_Hydroneer_C:NewGame()` called from Lua at the main menu starts a new world. It autosaved slot
  `MySaveGame` into the workspace: `Level.sav` 21,881,873 bytes, `Player.sav` 573, `SaveData.sav` 21.
  Played-in saves on a real install are about the same size (~21 MB), so the level file is mostly
  fixed-size (voxel data is the guess).
- Fresh world: 7,782 actors in 249 classes (5,570 `StaticMeshActor`, 316 `BP_OreVein_C`, 56
  `BP_Water_C`, 16 `BP_WorldStore_C`, 9 `BP_VehicleStore_C`, ...), 12,618 primitive components,
  7 `VoxelWorld`, 0 `VoxelWorldNetworkInfo`.
- 0 components reported `IsSimulatingPhysics() == true` in a fresh world. Unverified method: a
  failing call is counted as not simulating. Recheck with items the player has dropped.
- **`GM_Hydroneer_C:AddLocalPlayer()` from Lua works**: LocalPlayer 1 to 2, a second
  `BP_GameController_C` with its own `BP_Character_HydroGuy_C`, spawned about 10 m from player 1.
  Engine split-screen follows. This is the slot a remote player would drive.

## Phase 3 session 3 (2026-10-22): driving player 2, hiding split screen

Run under the hash guard (real folder unchanged afterwards). New game, `AddLocalPlayer()`, then
from Lua, every ~16 ms on the game thread (`LoopAsync` + `ExecuteInGameThread`):

| Input sent to P2 (the game's own BP input events) | Split screen on | Split screen off |
|---|---|---|
| `BP_GameCharacter:InpAxisEvt_MoveForward_K2Node_InputAxisEvent_0(1.0)` for 2 s | P2 moved 767 cm (116 ticks), P1 0 cm | P2 moved 791 cm (117 ticks), P1 0 cm |
| `BP_GameCharacter:HandleYawInput(1.0)` for 1 s | P2 control yaw -50 to 95, P1 unchanged (-75) | 95 to 240, P1 unchanged |
| `Character:Jump()` | P2 Z 239 to 362 after 300 ms | 274 to 397 |

- `/Script/Engine.Default__GameplayStatics:SetForceDisableSplitscreen(world, true)` works:
  `IsSplitscreenForceDisabled` true. Screenshots (`shot` console command via
  `KismetSystemLibrary:ExecuteConsoleCommand`, written into the redirected userdir): before, two
  vertical halves (P1 left, P2 right); after, P1's view fills the window. P1's two after-shots are
  identical while P2 moved, turned and jumped, so P2 keeps simulating with no viewport on the host.
- Movement direction follows P2's own control rotation (the second trial walked a different
  heading after the yaw), so the game's input BP ran for P2, not a raw position write.
- `GameplayStatics:IsSplitScreen` call failed from Lua (returned a function value); not needed.

### Derived

- The host-slot model works at the input level: a remote player's intents (move axes, look,
  jump, and later the controller's action events) can be replayed on a host-side P2 that the host
  does not see on screen. The host world simulates P2 with all the game's own rules.
- What the remote player sees still has to come from their own client world (state sync), or from
  a per-player stream (Remote Play again, which the owner rejected).

## Phase 3 session 4 (2026-10-22): player 2 pickup, drop, dig (split screen off)

Five runs, all under the hash guard (real folder unchanged after each). Calls were logged with
UE4SS `RegisterHook` pre-hooks on the game's own functions, with `self` recorded.

Input mappings (read from `Default__InputSettings.ActionMappings`): Pickup = E, LMB = left mouse,
RMB = right mouse, Jump = Space, Sprint = LeftShift, Crouch = C, Rotate = R/T/Y, VehicleMode = F,
VehicleKey = Q, Wiki = B, EscMenu = Escape (plus gamepad keys).

Generated input functions on `BP_GameController_C` (which one is key-down, observed):
- `InpActEvt_Pickup_K2Node_InputActionEvent_12` = E down (alone it calls `StartFreeplacing`,
  the hold-E placement mode), `_13` = E up. A tap is down then up 100 ms later.
- `InpActEvt_LMB_K2Node_InputActionEvent_5` = LMB down (calls the carried tool's
  `LMBDownWhileCarrying`), `_6` = LMB up.
- Sending up before down, or `PlaceDownItem()` directly, left P2 in a broken state (controller
  `CarryingItem` cleared but item still `BeingCarriedBy` P2). Send real key order only.

Results, P2 driven only by these input calls, split screen force-disabled:

| Action | Call chain logged | State after |
|---|---|---|
| Aim at shovel (SetControlRotation) | — | P2 `LastThingLookingAt` = the shovel, so P2's crosshair trace runs without a viewport |
| E tap (pickup) | `TryPickupItem`, `PickUpItem` on P2's controller | P2 `CarryingItem` = shovel, shovel `BeingCarriedBy` = P2's pawn, P1 carries nothing; the shovel moved with P2 while walking |
| E tap (drop), looking at ground | `ScanForDropCollision`, `ItemDropped`, `DropItem` (P2) | nothing carried, shovel on the ground |
| LMB tap at Ember Cradle voxel dirt (`BP_HydroVoxelWorldEmberCradle`) | `MoveDirt` (P2 controller), `HitActorItem`, `SetDirt`, `LMBDownWhileCarrying` | shovel `Dirt` mesh visible; surface at the spot Z 160 to 146 (trace now hits the landscape under the voxel) |
| P2 drops the full shovel, **P1** picks it up and clicks the same spot (control) | same chain on P1's controller | shovel empty, surface Z 160 to 171 (dirt dumped) |
| LMB at Mildews voxel world | only `LMBDownWhileCarrying` | nothing happens |
| LMB at landscape near spawn | only `LMBDownWhileCarrying` | nothing happens |

- P2 `IsMainController?` = false, P1 = true.
- 7 voxel worlds found: Dreck, EmberCradle, Icehelm, Mildews, Snowrune, SouthHope, plus a
  `BP_VoxelNoDig_C`. Surface found by `KismetSystemLibrary:LineTraceSingle` from Lua.
- Screenshot after the handover: host view full screen, P1 holding the empty shovel over dirt.

### Derived

- Pickup, drop and digging run for P2 through the same Blueprint code as for P1, triggered by the
  same input events. A network client can send key events (down/up per action) plus look/move axes,
  and the host replays them on P2.
- Mildews refusing to dig in a new game is probably the per-area dig licence, not a P2 problem:
  the dig chain stopped before `HitActorItem` there and on plain landscape alike. Not checked with
  P1 at Mildews.

## Phase 4 session 1 (2026-10-22): network input prototype, two copies on one PC

Goal: a second game copy sends its player's inputs over the network and drives P2 on the host.

### Setup (all actually run)

- Two copies: the host test copy and a client copy built from **hard links** to it (28 links:
  paks and engine files) with a **private** `Mining\Binaries\Win64` (17 files: exe, UE4SS, mods;
  UE4SS writes its log there). Client exe SHA-256 matches. The second copy started fine: no
  single-instance check seen.
- `tools/gameslot` lease for 2 slots around every two-copy run; every started pid registered.
- Transport (prototype only): UE4SS Lua has no sockets, so
  `client game -(Lua appends a line per ~16 ms to outbox file)-> client_relay.py -UDP-> host_relay.py -(appends to inbox file)-> host game Lua`.
  Relays are Python standard library. The host Lua keeps the inbox open and reads new bytes each
  tick (Lua 5.4 `read` clears EOF, so appended data shows up).
- Client mod `CoopNet` hooks its own input events with `RegisterHook` (BP functions):
  `BP_GameCharacter_C:InpAxisEvt_MoveForward_*_0` / `MoveRight_*_1` (largest-magnitude value per
  tick, because the engine also fires them with 0 every frame), and both generated events (down and up)
  of Pickup, LMB, RMB, Rotate*, VehicleKey, RotationMod, BuildPlace*, Jump, Crouch, Sprint. It sends
  absolute control rotation (pitch, yaw), not look deltas. Each line repeats the last 8 events with ids,
  so a lost datagram does not lose a key press.
- Host replays: the latest axes every host tick while packets keep coming (stop after ~300 ms
  of silence), `SetControlRotation` from the client's rotation, and each new event by calling the
  same function name on P2's controller or pawn with the matching `FKey`.
- Test: a bot on the client (rule 11), no human input. Markers in the event stream let both
  sides log positions at the same points.

### Results (two clean runs; second run shown, first run within 2 percent)

| | client P1 (bot input) | host P2 (replayed) |
|---|---|---|
| walk forward 2 s | 785 cm, yaw -75 | 784 cm, yaw -75 |
| turn (`HandleYawInput` on the client) | yaw to 67.5 | yaw to 67.5 |
| diagonal 1.5 s (fwd 0.7, right -0.7) | delta (496, 205) | delta (530, 219) |
| jump (Jump events down/up) | +122 cm | +121 cm |
| E tap, LMB tap | sent | replayed in order; game called `TryPickupItem` on P2 |
| transport | 630 lines sent | 630 received, 0 lost, UDP one-way p99 0.5 ms (loopback), longest gap between lines 3 host ticks |

The diagonal leg differs by about 7 percent: the host applies the held axis once per host tick, and
host and client tick counts differ. Movement will drift without position correction.

### Real `Settings.ini` writes: cause narrowed down

- The guard caught the real `%LOCALAPPDATA%\Mining\Saved\Config\WindowsNoEditor\Settings.ini` being
  rewritten in the first two-copy run (`r.setres=<width>x<height>w` to `...f`), the same change as session 1
  run 1. Restored from backup.
- Reproduced on purpose: a brand-new hard-linked copy, menu only, nobody touching it, rewrote it
  on its **first launch**; the **second launch** of the same copy did not. Session 1 run 1 was the
  test copy's first launch; the failing two-copy run was the client copy's first launch.
- The redirected `Settings.ini` is never created, and `SettingsManager.IniFilename` points at the
  redirect when mods load, so the writer uses another path, probably resolved before UE4SS mods
  load. Nothing is written inside the install copies. Writer: `unknown`.
- Minimizing and restoring the game window did not trigger a write.
- Hooking `/Script/AutoSettings.SettingsManager:SaveSettingStatic` and
  `ConsoleUtils:SetStringCVar`/`SetIntCVar` with UE4SS v3.0.1 `RegisterHook` crashed the game at
  startup ("Tried to execute UFunction::FuncPtr hook but there was no function map entry"). Do not
  hook static native AutoSettings functions this way.
- Rule until the writer is found: every new copy gets one warm-up launch under the guard (which
  restores the file and keeps the changed one in `quarantine-*`), before any real test.

### Derived

- The network input path works: the client's real input events, captured by hooks, drive the host's
  P2 through the game's own Blueprint input code, with movement within 1 to 7 percent and actions in
  order. The same capture works for human input, because it hooks the events, not the bot.
- Still missing for playable co-op: the client sees its own unsynced world. Next: send host state
  back (P1 and P2 transforms first, then items, voxel edits, money), correct the client's own player
  from the host, and replace the file+relay transport with in-process sockets (a UE4SS C++ mod, or
  SteamCore P2P for internet play).

## Phase 4 session 2 (2026-10-22): host state back to the client

Both runs under the gameslot lease and the hash guard; real folder unchanged after each.

### What was built

- Relays are two-way. The host relay learns the client's address from its first datagram and
  sends the host's outbox lines back. The client relay writes them to a client inbox file.
- Host sends (prototype rates, host ticks of ~16 ms):
  - `S`: P1 and P2 location and control yaw every 3 ticks (~20 Hz).
  - `I`: items (`BP_ParentItem_C`) that moved more than 2 cm or turned more than 2 degrees, in
    lines of 10. Only items within 50 m of a player are scanned (every 6 ticks), the near list is
    rebuilt every 120 ticks, all items are swept every 300, and the near list is resent in full
    every 186 ticks to cover loss.
  - `M`: test markers with P1, P2 and the tracked items' positions, each sent 3 times.
- Client:
  - Puppet for host P1: the client calls `AddLocalPlayer()` too, sets split screen off, sets the
    puppet's `CharacterMovement` mode to none, and moves it 35 percent of the way to the host
    position every tick, with host yaw.
  - Own player correction against host P2: snap above 400 cm; blend 10 percent per update above
    150 cm while moving, 25 percent above 40 cm while still.
  - Items matched by object name (`GetFName`). The client turns off physics on an item's root
    when it applies a host transform.
- Test harness: a debug teleport event (`D`, `TP_x_y_z_yaw`) moves P2 on the host and the client's
  player locally at the same time.

### Results (second run; first run the same within a few cm)

| Check | Result |
|---|---|
| Host P1 seen on the client (puppet vs host P1) | 0 cm at 8 of 9 markers, 17 cm right after the host's walk |
| Client's own player vs host P2 | 0 to 4 cm when still; 16 avg / 45 max cm while walking (run 1); 29 to 34 cm right after walking stops |
| Local-only 5 m desync | detected (498 to 510 cm), snapped back, 0 cm after 100 ms |
| Lantern carried by the host's P1 | client copy 0 to 1 cm from the host's at every marker |
| Shovel picked up, carried and dropped by the client (through host P2) | 0 to 1 cm on both sides; the host's logs show `TryPickupItem` and `DropItem` on P2 |
| Transport | 1,754 client lines, 0 lost; 2,258 state lines, 3,453 item updates in 400 lines back |
| Screenshots | client view: the host's character standing at the lamp post where host P1 is; host view: P2 (the client's player) walking by the post |

### Problems found

- Item updates with names the client does not have (37 per run): created when an item is picked
  up (`BP_Lantern_C_2147478289`, `BP_ShovelT1_C_2147478271`), very likely the held proxy copies
  (`CreateProxyItem`, `ToggleOnlyOwnerSee` on `BP_ParentItem_C`). The real items stayed in sync.
  Runtime-spawned items (bought, dug, crafted) have no cross-game identity yet either.
- Item scan costs up to 22 to 23 ms on the host game thread (a visible hitch). Needs a cheaper change
  test or spreading the scan over ticks.
- After a debug teleport the client snapped back twice to stale host state before the host's
  teleport arrived (643 cm). Needs a sequence check; real play does not teleport.
- The still threshold (40 cm) leaves about 30 cm of residual error after walking. Lower it.
- The puppet has movement off, so it likely slides in its idle pose (screenshot shows it standing).
  Animation state is not sent yet.
- The client's own action events also run locally (it picks up its own copy of the shovel). The
  host transform overrides it, but the client's local game state (carrying, dirt) is not
  host-controlled.

## Phase 4 session 3 (2026-10-22): fixes for session 2's problems

Three runs under the gameslot lease and the hash guard; real folder unchanged after each.

### Changes

- **Proxy items**: `BP_ParentItem_C` has `ProxyItem` (the held copy) and `CreateProxy?`. The host
  records every item's `ProxyItem` name and never sends those.
- **Item scan cost**:
  - No full scan on one tick. Each tick checks 6 items round robin (a full sweep of 557 in ~1.5 s),
    plus "hot" items (carried, or moved in the last ~1 s) every tick. Items that ever moved are
    resent every 125 ticks.
  - No periodic `FindAllOf`: one at start, then `NotifyOnNewObject` on `BP_ParentItem_C` appends
    new items, and the sweep drops destroyed ones.
  - `collectgarbage("generational")` in the host mod (it makes many short strings each tick).
- **Correction with acks**: the host's `S` line now carries the last client input line and event
  it applied (`ack_seq`, `ack_ev`). The client keeps its own position for each line it sent and
  compares host P2 with its position *at the acked line*, so latency no longer looks like
  divergence. Corrections: snap above 400 cm, else 30 percent of the difference above 10 cm, and the
  recorded positions after the ack are shifted by the applied amount. This also fixed the teleport
  race (the client's teleport and the host's are compared at the same line).
- **Puppet animation**: the host sends P1's `GetVelocity()`. The client writes it into the
  puppet's `CharacterMovement.Velocity` every tick while its movement mode stays none. The anim
  instance (`ABP_HydroGuy_C`) reads it: `Speed` matched host P1's speed every sample
  (for example 239/239, 425/425, 312/312). An automatic fallback to walking movement mode was
  never needed.

### Results

| Check | Session 2 | Run 1 (anim, acks, spread scan) | Run 2 (+ notify list) | Run 3 (+ generational GC, 6 per tick) |
|---|---|---|---|---|
| Unknown items on the client | 37 | 0 (2 proxies skipped) | 0 | 0 |
| Host tick work avg / max | scan alone up to 23 ms | 0.78 / 16 ms | 0.78 / 11 ms (10 ticks over 4 ms) | **0.56 / 5 ms** (1 of 1,761 ticks over 4 ms) |
| Snaps at the debug teleport | 2 (643 cm) | 0 | 0 | 0 |
| Client player vs host P2 when still | 30 to 34 cm | 0 to 1 cm | 0 to 1 cm | 0 to 10 cm |
| Reconciled error while walking (avg / max) | — | 1.6 to 14 / 30 cm | 5 / 21 cm | 3 to 7 / 21 cm |
| Puppet anim `Speed` vs host P1 speed | not sent | equal | equal | (not logged) |
| Puppet vs host P1, lantern, shovel at markers | 0 to 17 cm, 0 to 1 cm | 0 to 12, 0 to 1 cm | 0 to 10, 0 to 1 cm | 0 to 14, 0 to 1 cm |
| Local-only 5 m desync | snapped back | snapped back, 0 cm at 100 ms | same | same |

- Client screenshot one second into the host's walk, enlarged: the puppet is mid-stride (legs
  apart, arms swinging), not standing.
- `os.clock` on this build has 1 ms resolution, so phase times are whole milliseconds. The last
  over-4 ms tick was in the item scan.
- Residual: the client's player can rest up to 10 cm off the host (the blend threshold); lower it
  if that shows.

## Phase 4 session 4 (2026-10-22): host decides actions, client mirrors

Two runs under the gameslot lease and the hash guard; real folder unchanged after each.

### Changes

- **Client actions no longer run locally.** World-changing actions (Pickup, LMB, RMB, Rotate,
  Rotate2, Rotate3, VehicleKey, VehicleMode, RotationMod, BuildPlaceCloser/Further) lose their key
  mappings on the client.
  - `InputSettings:RemoveActionMapping` from Lua removed nothing (22 before, 22 after), even with
    the modifier flags copied from each mapping.
  - What worked: setting each of those mappings' `Key.KeyName` to `None` in
    `Default__InputSettings.ActionMappings`, then `ForceRebuildKeymaps()` (22 to 0).
  - Jump, crouch and sprint stay local (they only move the client's own player, which is reconciled).
- **Keys are polled instead:** `PlayerController:IsInputKeyDown` for E, LMB, RMB, R, T, Y, Q,
  LeftControl and the gamepad equivalents. Edges become the same down/up events (lower generated
  index = down), which are only sent.
- **Host `S` line also carries** what P1 and P2 carry (item name or `-`) and the carried item's
  dirt flag (`Dirt` mesh visible, 1/0, -1 if none).
- **Client mirrors carrying:** host P2 carries X and the client does not, so the client calls
  `PickUpItem(X)` (the game's own carry logic). Host P2 carries nothing, so the client calls
  `DropItem`. At most two attempts per second. The client does not apply host transforms to the
  item it carries itself.
- **Client mirrors shovel dirt:** `item.Dirt:SetVisibility(...)` to the host's flag (own carried
  item and host P1's).
- **Digging:** the host hooks `BP_GameController_C:MoveDirt(PickUp?, Radius)` (args readable in
  the post hook) and sends `V|id|pickup|x|y|z|radius`, with the location from that controller's
  `CrosshairTraceHit`, sent 3 times.
  - Scoop on the client: `RemoveDirtByLocation(L, R)` at the host's point.
  - Dump on the client: the controller has no add-at-location function (only `MoveDirt`,
    `RemoveDirtByLocation`, `RemoveDirtLine`, `RemoveDirtVehicle`, `Scan for Items in Voxel Dirt`),
    so the client calls its own `MoveDirt(false, R)`; its aim equals host P2's aim.
  - Both sides trace the ground at the dig point 500 ms later; the host sends `Z|id|z` to compare.
- **Real key test:** the orchestrator brings the client test window to the front, checks
  `GetForegroundWindow` equals it, then sends E down/up with `keybd_event`. It refuses and presses
  nothing when the window is not in front.

### Results (second run)

| Step | Host (authority) | Client |
|---|---|---|
| E (bot event) at the shovel | `TryPickupItem` on P2 | mirror `PickUpItem`; both carry the shovel |
| LMB on Ember Cradle dirt | `MoveDirt(true, 30)` on P2; ground 160.2 to **145.7** | `RemoveDirtByLocation`; ground **145.7**; shovel dirt full |
| LMB 1.5 m to the side | `MoveDirt(false, 28)`; ground **194.5** | `MoveDirt(false)`; ground **194.5**; shovel dirt empty |
| E (drop) | `DropItem` on P2 | mirror `DropItem`; shovel 0 cm apart |
| **Real E key pressed on the client window** | `TryPickupItem` on P2 | key polled (down and up); mirror `PickUpItem`; **no local `TryPickupItem`** |
| Local game calls on the client, whole run | — | `PickUpItem`=2, `DropItem`=1, `RemoveDirtByLocation`=1, `MoveDirt`=1: exactly the mirrors (pickups 2, drops 1, digs 2) |

- First run of this session: mappings were not removed, and the real E press also ran
  `TryPickupItem` locally (count 1). Fixed by blanking the keys.
- While carried, the shovel is 60 cm apart between client and host: the client's first-person
  carry pose versus the host's third-person one. Expected; the client skips host transforms for
  its own carried item.

## Phase 4 session 5 (2026-10-22): runtime item identity (blocked on client spawning)

All runs under the hash guard (one two-copy run with the gameslot lease, three single-copy
experiments); real folder unchanged after each.

### Host side: works

- Every new `BP_ParentItem_C` (`NotifyOnNewObject`) is registered ~10 ticks after construction.
  Proxies are excluded by checking the carried items' `ProxyItem` first (1 new proxy skipped in
  the run). Others get a net id `N<k>`. The host sends
  `N|id|class path|transform` (3 times) and uses the id everywhere on the wire: `I` lines, the
  `S` carry fields, and markers.
- Game spawn path used for the test: `GM_Hydroneer_C:SpawnInitialItems()` created 8 items, all
  registered: `N1` `BP_WoodenCart_C`, `N2` `BP_WorldMap_C`, `N3` `BP_TreasureChest_C`,
  `N4` `BP_Lantern_C`, `N5` `BP_Bucket_C`, `N6` `BP_Brush_C`, `N7` `BP_Pan_C`, `N8` `BP_ShovelT1_C`,
  with class paths such as `/Game/WorldObj/Items/Tools/BP_Pan.BP_Pan_C`. Host P1 picked up `N1`
  through its input events (`P1 carries N1`) and dropped it.
- **Destroy detection:** after `K2_DestroyActor`, UE4SS 3.0.1 still reports `IsValid() == true`
  and `FindAllOf` still lists the actor; its location reads 0,0,0 and **its root component is
  invalid at once**. The first run therefore sent no `X` for the destroyed `N2`. The host now
  treats "root component invalid" as gone. **Not yet run in a two-copy test.**

### Client side: spawning from Lua fails on UE4SS 3.0.1

| Attempt | Result |
|---|---|
| `GameplayStatics:BeginDeferredActorSpawnFromClass(ctx, class, FTransform-table, 1, nil/pawn)` | pcall fails; the error value is a Lua function, no message in the log; 3 variants |
| `KismetMathLibrary:MakeTransform(...)` then pass its result | MakeTransform returns a plain Lua table; passing it fails the same way. Inferred: FTransform parameters do not convert in this UE4SS build |
| `GameplayStatics:SpawnObject(actor class, outer)` (only to test class-param conversion) | Class parameter converts; it **constructed a `BP_Bucket_C` object owned by the controller**, not placed in the level (test world only). Do not use for actors |
| `GameplayTask_SpawnActor:SpawnActor(owner = BP_AIC_C, location, rotation, class, false)` (no FTransform) | returned nil (no task); the `TScriptInterface` owner probably does not convert |
| Game functions that spawn from a class | `BP_ParentMasterwork_C:SpawnItem(Class)` (crafting output; same graph has `CheckForAchievements`) and `C_Task*:SpawnReward(...)` (quests). **Not used**: they may touch Steam achievements or quest state on the owner's account. No other game or engine function takes class + location for an ordinary actor |

### Options (owner decision)

1. Try a newer UE4SS build (`experimental-latest`, `UE4SS_v3.0.1-1181-g83330635`, published with
   a GitHub SHA-256 digest; uploaded 2026-10-17) and repeat the spawn tests. Cheap; unknown whether
   it fixes FTransform or interface parameters.
2. A UE4SS **C++** mod: real `SpawnActor`, and in-process UDP sockets (no files and relays). Needs a
   C++ toolchain (Visual Studio Build Tools, CMake) and the UE4SS C++ build, which per its docs
   needs a GitHub account linked to the Epic Games organization for the Unreal headers.
3. Replay the cause, not the result: call the same game function on the client (for example
   `SpawnInitialItems`) and pair new items by class and order. Duplicates side effects (money,
   quests, achievements), so only for side-effect-free spawns. Not recommended as the general path.

## Phase 4 session 6 (2026-10-22): newer UE4SS, client spawning works

### UE4SS experimental build (test copies only)

- Release tag `experimental-latest`, asset `UE4SS_v3.0.1-1181-g83330635.zip`, 8,748,315 bytes,
  uploaded 2026-10-17. SHA-256 `b9a166a756605b21058a64cde93bcaab1a855dba3553e5c7e6ce0fa8fdfad7b9`
  **matches the GitHub release digest**. Log header: `UE4SS - v3.0.1 Beta #0 - Git SHA #83330635`.
  The tag moves; this exact asset is the pin.
- Layout change: `dwmapi.dll` stays next to the exe; everything else is in `Win64\ue4ss\`
  (`UE4SS.dll`, settings, `Mods\` with `mods.txt` and `mods.json`, `UE4SS.log`). The 3.0.1 files
  were moved, not deleted, to a backup folder per copy. Only our mods plus `ConsoleEnablerMod`
  and `Keybinds` are enabled (both read).
- New Lua API in this build (strings in `UE4SS.dll`): `UWorld:SpawnActor(UClass, table Location,
  table Rotation)`, `LoopInGameThreadWithDelay`, `ExecuteInGameThreadWithDelay`,
  `LoopInGameThreadAfterFrames`, `CancelDelayedAction`, `PauseDelayedAction`,
  `RegisterLoadMapPre/PostCallback`.
- Single-copy test: `pc:GetWorld():SpawnActor(BP_Bucket_C, loc, rot)` and a `BP_Pan_C` both spawned
  into `PersistentLevel` at the requested positions; moving them afterwards works.
- `InputSettings:RemoveActionMapping` works on this build (22 to 0); on 3.0.1 it did nothing.

### Crash on the new build, and the fix

- First two-copy run: the host exited (code 3) right when its 16 ms loop started. The game's crash
  report (`<userdir>\Saved\Crashes\UE4CC-Windows-...\CrashContext.runtime-xml`; it lands in the
  redirected userdir) says `EXCEPTION_ACCESS_VIOLATION reading address 0x0`, call stack inside
  `UE4SS` from the engine tick, 95 s after start. A single-copy host-only rerun logged
  `[Lua::Registry::get_function_ref] Ref was not function` after tick 4, and the loop stopped.
- That loop queued a new `ExecuteInGameThread` callback every 16 ms from `LoopAsync`, and the host
  had switched Lua to `collectgarbage("generational")`. Both were removed: all loops and delays now
  use `LoopInGameThreadWithDelay` / `ExecuteInGameThreadWithDelay` when present (the old pattern
  stays as a fallback for 3.0.1). Which of the two caused the crash is **unknown**.
- After the change: host-only run past tick 100 without errors, and the full two-copy run below
  finished with no crash report.
- The crash reporter (`CrashReportClient`) was not left running.

### Results: two copies, both on the new build

| Check | Result |
|---|---|
| Runtime items from `GM_Hydroneer_C:SpawnInitialItems()` on the host | 8 registered (`N1`..`N8`); 2 new proxies skipped |
| Copies spawned on the client (`World:SpawnActor` with the host's class path, location and rotation) | **8 of 8**, at 0 to 1 cm from the host's items |
| Host destroys `N2` (stand-in for sold or consumed) | host: `item gone: N2` (root component invalid); client: copy destroyed, still gone at every later marker |
| Client's player carries `N8` through the host | host `TryPickupItem` on P2 (carry field `N8`); client mirrors `PickUpItem` on its spawned copy; carried, dropped; `N8` 0 to 7 cm apart |
| Digging and dumping (repeat) | ground 145.7 / 194.5 on both sides |
| Unknown items on the client | 0 |

### Problems found

- **Brush `N6` was 47 cm off on the client** from `h_moved` to the end. Likely cause (not
  verified): the client's spawned copies overlap the client's own starter items at the same spots,
  and client physics pushed it. The host never moved it, so it never resent it (the periodic resend
  only covers items that moved on the host). Fix options: keep physics off on all client copies, or
  a slow full resend of all items.
- Host P1's bot picked up the original starter cart instead of `N1` (both carts at the same spot).
  Harmless here.

## Phase 4 session 7 (2026-10-22): client copies stay where the host has them

One two-copy run (both copies on UE4SS v3.0.1-1181) under the gameslot lease and the hash guard;
real folder unchanged, no crash report.

### Changes

- **Client physics guard:** each tick, 8 item copies (round robin over the client's starting items
  plus every spawned copy) get their root's physics turned off if it simulates. The client's own
  carried item is skipped.
- **Host slow full resend:** besides change-based updates, 2 items per tick are sent forced, so
  every item goes out again every ~4.5 s (564 items at ~60 ticks/s).
- **Whole-world check (test harness):** at the end, host and client each write every live item
  (wire name and location) to a file in the run folder; `local/tools/coopnet/compare_items.py`
  compares them.

### Results

| Check | Result |
|---|---|
| All items, host vs client, at the end | **564 vs 564, all matched, 0 missing, 0 extra, max error 0.0 cm** |
| Brush `N6` (47 cm off in session 6) | 1 cm at markers, 0.0 cm in the final comparison |
| Physics guard | turned physics off **0 times**: no copy simulated physics this run, so the drift did not happen and the guard did not act. The fix that shows its effect is the full resend; session 6's drift cause stays unverified |
| Runtime items, destroy, carry, digs | as session 6 (8 copies, `N2` removed, `N8` carried, ground 145.7/194.5) |
| Item traffic | 15,656 item updates in 7,850 lines (session 6: 927 in 437). About 50 datagrams/s; a forced resend line holds only 2 items |

### Problems found

- Batch the forced resend better (it flushes every tick with 2 items).
- Reconciled error while the client carried `N8` reached 67 cm max (22 avg) on one stretch; other
  runs stayed under ~20 cm. Host tick cost was not measured in this run; check whether the resend
  adds load.

## Phase 4 session 8 (2026-10-22): resend batching, host cost, the correction spike

Six two-copy runs under the gameslot lease and the hash guard; real folder unchanged after each,
no crash reports. All on UE4SS v3.0.1-1181.

### Resend batching and host cost

- Forced resends now go in their own batch, sent at 12 items (~every 6 ticks): 15,057 to 15,556
  resent items in 1,254 to 1,296 full lines per run (session 7: ~7,500 half-empty lines).
- Host tick work: avg 0.46 to 0.50 ms, max 3 to 5 ms, 0 or 1 tick over 4 ms per ~3,000 ticks.
  Phase maxima: item scan 3 to 5 ms, axes+S up to 5 ms, resend 1 to 3 ms, inbox+replay 1 to 3 ms.
- **Observed:** the game-thread loop (`LoopInGameThreadWithDelay(16)`) runs about every **34 ms**
  (~30 Hz), not 16 ms. One ~520 ms gap per run at start (world load).

### The correction spike

- Client logging (every reconciliation step over 30 cm) showed the error growing steadily in the
  walking direction to ~70 cm: host P2 walked ahead of the client's own position at the acked
  line, with the acked line 33 to 34 ms old. The client's **own** movement (position change per
  tick, corrections excluded) was **11 to 18 cm/s** against `MaxWalkSpeed` 425; host P2 moved at
  399 to 425. The client was being dragged by the blends, not walking.
- Isolation runs (three walks each: nothing carried, after picking the original shovel, carrying
  spawned copy `N8`):

| Run | Client physics guard | Host full resend | Own speed, cm/s | Max reconciled error | All 564 items max error |
|---|---|---|---|---|---|
| A | on | on | 18 / 11 / 17 | 70 cm | 0.0 cm |
| B | off | off | 338 / 76 / 331 | 35 cm | 4.3 cm |
| C | on | off | 17 / 12 / 17 | 69 cm | 25.1 cm (2 items over 5 cm) |
| D | off | on | 139 / 49 / 239 | 42 cm | 0.0 cm |
| final | off | on | 233 / 96 / 112 | 54 cm | 0.0 cm |

- **Observed:** with the physics guard on, the client's player barely moves (A, C), although the
  guard only read component state (`K2_GetRootComponent`, `IsValid`, `IsSimulatingPhysics` on 8
  items per tick) and changed nothing in any run. **Mechanism unknown.** Guess (unverified): the
  extra UE4SS calls in the client's game-thread callback delay the bot's movement calls, which
  also run as delayed game-thread actions.
- **Observed:** the full resend is what keeps items right (C without it: 25 cm drift; D and final
  with it: 0.0 cm).
- The guard is now off (only with env `HYDRO_COOP_GUARD=1`). Applying a host transform still turns
  physics off on that item.
- Still open: the client's own walking speed varies a lot between runs (the "after pickup" walk is
  always slow: 49 to 96 cm/s), so 35 to 54 cm corrections remain during walks. The bot drives
  movement by calling the axis event from a ~30 Hz timer, not every frame as real input does. Next
  test: hold a real key (W) on the client window and measure own speed, to see whether real input
  shows the same slowdown.

## Phase 4 session 9 (2026-10-22): real held W on the client

Two two-copy runs under the gameslot lease and the hash guard; real folder unchanged after each.

### Method

- The orchestrator now takes `HOLD <key> <ms>` requests (max 5000 ms). It brings the client test
  window to the front, checks `GetForegroundWindow`, presses W with `keybd_event`, keeps checking
  every 20 ms that the window is still in front (releases at once if not), and releases after the
  time. Both holds lasted 2,004 to 2,033 ms.
- The client logs its own speed (corrections excluded), distance moved and the worst reconciled
  error during each walk; spike lines give per-update detail.

### Run 1: real input versus bot input (P2 replay still on the 30 Hz timer)

| Walk | Client own speed | Worst correction |
|---|---|---|
| bot (axis event from a ~30 Hz timer), nothing carried | 182 cm/s | 29 cm |
| **real W held, nothing carried** | **400 to 405 cm/s** (MaxWalkSpeed 425) | 52 to 74 cm |

- **Observed:** the slow client walks of session 8 come from the bot's input method. Real input
  walks at full speed.
- With real input the error is the reverse: host P2 trails the client's acked position by a steady
  34 to 53 cm along the walk, peaking at 74 cm when the client stops. Host P2 replays axes once per
  host timer tick (~34 ms) while real input arrives every frame, so P2 starts later and moves a
  little slower.

### Fix: replay P2's axes every frame

- The host replays the latest movement axes and control rotation with
  `LoopInGameThreadAfterFrames(1, ...)` (UE4SS v3.0.1-1181): **70 to 81 calls per second**
  observed. The rest of the host work stays on the 34 ms timer. The stale-input timeout is now
  300 ms of clock time (it was 18 timer ticks, ~600 ms at the real tick rate).

### Run 2: with per-frame replay

| Walk | Client own speed | Worst correction | Host P2 |
|---|---|---|---|
| **real W held, nothing carried** | **409 cm/s**, 800 cm in 2 s | **19 cm** | 425 cm/s |
| real W held, carrying shovel `N8` (picked with a real E) | 39 cm/s, 79 cm | 13 cm | did not move faster either |
| bot walk, nothing carried | 162 cm/s | 75 cm | 425 (now ahead of the bot) |

- The real carrying walk was slow on both sides alike. The player was looking steeply down at the
  shovel when W was held; whether that or an obstacle slowed it is **unknown**. The sides agreed
  (13 cm), so it is not a sync problem.
- Host tick work avg 0.44 ms, max 3 ms, 0 of 3,383 ticks over 4 ms.
- All items: host 564, client 565, 564 matched, max error 0.2 cm. The extra client item
  `BP_ShovelT1_C_2147478441` is very likely the client's own held proxy from its local carry; the
  client's dump does not leave out its own proxies yet.

### Run 3 (same session): bot movement every frame, client proxy left out of its dump

- The bot's `drive()` now calls the axis event every frame with `LoopInGameThreadAfterFrames(1)`
  (timer fallback on 3.0.1). The client's item dump leaves out the `ProxyItem` of the item it
  carries (logged: `leaving out our own held proxy BP_ShovelT1_C_2147478377`).

| Walk | Client own speed | Worst correction |
|---|---|---|
| bot, nothing carried | **405 cm/s** (was 162 to 182) | 18 cm |
| real W held, nothing carried | 409 cm/s | 18 cm |
| bot, carrying spawned copy `N8` | **395 cm/s** | 13 to 19 cm |
| bot right after picking a shovel (looking steeply down) | 56 cm/s | 13 cm |
| real W right after a real E pickup (looking steeply down) | 43 cm/s | 13 cm |

- Worst reconciled error over the whole run (after the start snap): **19.4 cm**.
- The two slow walks both start while the player looks steeply down at the shovel just picked up,
  and host P2 was equally slow (32 cm/s logged). **Likely** the downward look direction; not
  verified. Not a sync issue (sides within 13 cm).
- All items: host 564, client 564, all matched, max error 0.1 cm, nothing extra.
- Host tick work avg 0.44 ms, max 3 ms, 0 of 3,392 ticks over 4 ms after the client connected.
  One item-scan phase maximum of 10 ms falls outside those counted ticks, most likely before the
  client connected (world load).
- Real folder unchanged.

### Still open

- The dump mirror relies on the client's aim matching host P2's. Not tested with lag or movement
  during the click.
- `MoveDirt(true)` on the host may spawn items found in the dirt (`Scan for Items in Voxel Dirt`);
  items spawned at runtime still have no cross-game identity.
- Only the E key was tested as a real key press. LMB/RMB/gamepad polling only ran via bot events.
- Transport is still files plus relays.

### Unknowns

- Other tools: pickaxe on `BP_OreVein_C`, pan, bucket, building; shop and other UI widgets for P2
  (they would need to show on the remote client's screen instead).
- Whether P2's HUD/UI widgets are created and where they render with split screen disabled.
- P2 `CrosshairTraceHit` did not change in one run after a second `SetControlRotation`
  (session 4, v4 run); in the next run it was right each time. When it refreshes is unknown.
- Save behaviour with P2 present (whose `Player.sav`, does P2's inventory persist).
- Physics item counts and behaviour in a played-in save.
- Whether the Voxel network functions are compiled in and working in shipping.
- How many physics-simulated items a mid-game save has (needs a loaded world).
- Exact save path logic in `GetSaveDirectory`.

## Observations (sourced)

- The official multiplayer (2.0 update) is split-screen; online play goes through Steam Remote
  Play Together, so the remote player sees a stream of the host's split screen.
  [TechRaptor guide](https://techraptor.net/gaming/guides/hydroneer-multiplayer-guide),
  [2.0 update news](https://techraptor.net/gaming/news/hydroneer-20-multiplayer-update-is-out-now)
- The developer team says it "tried all avenues" for true multiplayer and blames physics load;
  a moderator calls a full multiplayer mod "highly unlikely" (2022-08-03/04).
  [Steam thread "Any multiplayer mod?"](https://steamcommunity.com/app/1106840/discussions/0/3458220484817654192/)
- Community mods load through HML (Hydroneer Mod Loader); Workshop mods work in split-screen.
  [Steam thread](https://steamcommunity.com/app/1106840/discussions/0/3729575504107129027)
- A community UE editor modkit exists for game version 3.0.8 and used UE4SS's UHT generator
  for dummy headers, so UE4SS runs on at least that build.
  [Hydroneer-Modkit](https://github.com/Buckminsterfullerene02/Hydroneer-Modkit)
- [Hydroneer-Template](https://github.com/Buckminsterfullerene02/Hydroneer-Template) describes
  itself as "A template UE4.26 project for BP modding".
- Related but not host/join co-op:
  [Hydroneer-P2-Keyboard](https://github.com/codeopen0-cpu/Hydroneer-P2-Keyboard) (keyboard for
  split-screen P2, pushed 2026-06-07);
  [HydroneerAPMod](https://github.com/Thedragon005/HydroneerAPMod) (Archipelago randomizer,
  item sync only, pushed 2026-09-26).

## Derived

- Engine 4.26 comes from the modding template, not from the exe. Check the exe version info.
- Split-screen means the game already handles several player pawns and controllers in one
  world. A host/join mod could reuse that: the host keeps simulating everything (physics,
  water, machines, saves) and a remote client drives a host-side player slot as a puppet,
  with its own camera on its own PC.

## Guesses (untested)

- The UE4 net driver may still be in the shipping exe (`open <ip>` / `?listen`), but the
  game's Blueprints are almost certainly not replicated, so engine replication alone would
  not sync the world.
- Full physics state sync (many loose ore, pans, buckets) is the hard part the developer hit.
  A host-authority design with snapshot interpolation for nearby objects is the likely route.

## Unknowns

- Exact build, exe hash, UE version.
- Whether `IpNetDriver` / `SteamNetDriver` are in the shipping build.
- Save format and where saves live.

## Searches run (2026-10-22)

- This repo: DIRECTORY.md, `games/`, issues and PRs for "hydroneer": nothing found.
- Web: "Hydroneer multiplayer co-op how to host join", "Hydroneer online multiplayer mod host
  join", "Hydroneer modding UE4SS ...": no host/join mod found.
- GitHub repo search: "hydroneer", "hydroneer multiplayer", "hydroneer coop", "hydroneer mod":
  no host/join mod found. Forks and branches of hits not yet checked.
- Not yet checked: Nexus, Hydroneer modding Discord, Reddit, Steam Workshop.
