# Sekiro: Shadows Die Twice (2019)

| | |
|---|---|
| Engine | FromSoftware proprietary (x64, Windows), shares lineage with Dark Souls III |
| Versions tested | Steam 1.06 (`sekiro.exe` ProductVersion 1.6.0.0, PE timestamp `0x5fa3e066`, SHA-256 `637aca527538c0ec6e1f136c8ed66046e95dfbdbb1f51926e134d9916398b856`) |
| Projects | [Sekiro Online](https://www.nexusmods.com/sekiro/mods/577) (Souls-style summons, abandoned 2021); [sekiro-coop](https://github.com/mstampfli/sekiro-coop) (seamless 2-player prototype, Rust) |
| QA level | none (one real game client plus the project's peer simulator; no two-client test yet) |
| Last updated | 2026-10-28 |

## Where things stand

Nobody can play the Sekiro story seamlessly with a friend yet. Sekiro Online gives Souls-style
co-op through summon signs and Steam, but it has been unmaintained since 2021 and we did not test
it. sekiro-coop is the only seamless attempt. On a public branch ([vano112211-man/sekiro-coop `coop/puppet-probe`](https://github.com/vano112211-man/sekiro-coop/tree/coop/puppet-probe)) the second player
now shows up as a stand-in enemy (the Owl) that follows the remote player and plays matching
attacks, blocks and dodges, tested on one PC against the peer simulator. A Wolf-looking second
player exists but does not walk yet. A Steam P2P transport and a native remote-player packet pipe
are written but not yet run.

## Existing work

- **Sekiro Online** (Nexus #577): v2.05, 2021-06-13. Summon signs, invasions (cannot be turned
  off), 1 host plus up to 5 phantoms over Steam P2P, level scaling. Must reach the Dilapidated
  Temple first. Page says it is no longer developed and mentions a new version "as of 09/03/2025";
  nothing newer was found. Closed source.
- **sekiro-coop** ([mstampfli/sekiro-coop](https://github.com/mstampfli/sekiro-coop), 6 commits,
  last 2026-06-15, AGPL-3.0-or-later): Rust DLL loaded by [me3](https://github.com/garyttierney/me3),
  MinHook detours, raw UDP direct IP (Steam P2P is an empty stub), host-authoritative enemy HP.
  Its `docs/HANDOFF.md` plans to drive a puppet NPC by patching `common.emevd`; not started.
- No Seamless Co-op (LukeYui) release for Sekiro was found.

## How the game works (for co-op)

- The Steam exe on disk is packed (`.bind` section, encrypted `.text`): AOB patterns only match the
  decrypted image in memory, so static scans of the file find nothing.
- me3 0.11.0 starts the game with Arxan neutralised (`--disable-arxan true`), blocks matchmaking
  (`--online false`) and can redirect the save to a separate file (`--savefile`).
- Live character state is reached from a `ChrIns` (character instance):
  - handle `+0x08`, character id `+0x68`, team byte `+0x74`;
  - HP/posture `+0x1ff8 → +0x18 → +0x130/+0x134/+0x148/+0x14C`;
  - animation id `+0x1ff8 → +0x10 → +0x20`;
  - physics-module position `+0x1ff8 → +0x68 → +0x80`.
- The game's global debug flag "all no update AI" is at `debug_flags + 13` (1.06 RVA of
  `debug_flags`: `0x3d7a369`).

## What we changed

Nothing is released. All changes are local experiments on top of sekiro-coop `1ef5fa8`, described in
[NOTES.md](NOTES.md):

- Build fixes: MinHook rebuilt from the vendored (byte-identical to upstream `d94c64d`) source
  instead of the prebuilt `.lib`.
- A puppet probe (`SEKIRO_COOP_PUPPET_PROBE=1`) that pins one enemy and holds its physics position
  next to the player. Findings:
  1. Writing the physics position **from the game thread** (inside the SetFlag detour) moves the
     character on screen. sekiro-coop's earlier conclusion that the render path ignores these
     writes came from writing on a separate 60 Hz thread.
  2. The team byte (6 → 2) does not stop the puppet attacking; freezing all AI does.
  3. The game keeps pulling the character back to a stored home position: during a 1 s write pause
     it snapped back to its spawn point, and ~25-80% of writes were reverted. Also writing the
     `ChrIns+0x1060` copy did not help; `+0x1050` is not maintained by the game. Position poking
     fights the engine; a puppet needs the native move/warp, animation and per-character AI calls.
  4. Writing the player's height into slopes made physics push the puppet out every frame
     (FPS fell to ~15); keeping the game's own height let it fall through the world.
- Offline analysis of a 1.06 memory image (kept local) found the script command natives
  (`EventEntityCommandUtil::ChrCmd_*`): per-character EnableLogic (bit 0 of `[ChrIns+0x50]+0x190`),
  SetDisableGravity, SetDisableMapHit and the native warp `ChrIns->vt[+0x6A0](pos, rot, 1, 0)`.
  RVAs are in [NOTES.md](NOTES.md).
- Puppet v3 uses those: AI off for the puppet only, native warp every 500 ms (or when more than
  2 m off) plus small physics steps in between. Result on the owner's PC: the puppet follows
  smoothly 3 m ahead, does not attack, FPS normal, almost no flicker. Warping every frame flickered.
- Driven by the network: the puppet follows the remote player's position from PlayerSnapshot
  (tested with the peer simulator replaying the player's own path). Rotation: physics +0x70 holds
  `[0, yaw, 0, 0]` (radians), yaw = atan2(-dx, -dz). With the native animation reset
  (`0x9f4c60`) every 2 s and a native warp every 50 ms, the puppet walks and runs with its own
  locomotion animations and faces its direction of travel.
- Fixed in sekiro-coop: freed characters stayed in the registry and crashed the host's enemy
  broadcast after a death/reload; the simulator now answers handshake retries.

- Actions (2026-10-27/28): the player's action ring (TimeAct `[sm+0x10]+0x34+k*0x28`) is sent as
  `action_id/action_seq`; the puppet plays them through the native ForcePlayAnimation
  (`0xbd84c0(module=[[chr+0x1ff8]+0x58], id)`). The Owl (c5060, NpcParam 50600000) has nearly the
  player's move set; map used: combo hits to 3000/3001/3005/3006, block and parry to 3093, dodge
  to 3064. The owner saw matching attacks, block and parry, and a dodge that looked right; FPS
  normal.
- Wolf-looking second player: a real `PlayerIns` created with the game's net-player path
  `0xa49460(WorldChrMan, id, data)` plus `0x9f5170(chr, 0)` and `vt+0x90`, is visible with the
  player's own appearance and equipment. Driving it through a swapped PadManipulator replays
  actions, but it never plays locomotion (it floats or stands). Ruled out on the owner's PC: input
  fields, manipulator sub-objects, IsLocalPlayer, omission level, prosthetic/save state.
- Offline (not yet run): the engine's own remote-player pipe. Every PlayerIns has a
  PlayerSessionHolder (`+0x2020`, session pointer at `+0x18`, normally the PlayerNullSession
  singleton). The local PadManipulator serialises a DP_REGULAR packet and calls
  `session->Send(buf, size, 1)`; a kind-2 NetworkManipulator calls `session->Receive(...)` and
  applies it. The branch swaps in fake sessions to capture the player's packets and feed them to
  the clone. Details and RVAs in [NOTES.md](NOTES.md).
- Steam (not yet run): Sekiro 1.06 ships an old `steam_api64.dll` with no flat API, so the
  transport calls `ISteamNetworking` (SteamNetworking005) through its C++ vtable, using the game's
  own AppID; see the new section in
  [playbook/networking-steam-spacewar.md](../../playbook/networking-steam-spacewar.md).

## QA

- sekiro-coop's own data plane, reproduced on one PC against its `peer-simulator` over loopback:
  session established, remote snapshots and flag/effect events received, host enemy-state echo
  resolved for every enemy (`matched=2 unmatched_count=0`). No desync kill, no crash.
- The Owl puppet was checked by eye by the owner with the simulator mirroring his own moves
  (runs 40-56 in NOTES.md).
- No two-client test and no bot test yet, so the QA level stays `none`.

## How to play

Not playable yet. The branch has a launcher for two players over Steam (`play-steam.cmd
host|client <friend SteamID64>`), untested between two accounts.

## Known issues

- The second player is an enemy model (the Owl) standing in for the player; the Wolf-looking
  clone does not walk yet.
- Enemies, story and world state are not shared beyond sekiro-coop's flag/effect events.
- Steam connection untested; the upstream project is direct IP only.

## Next steps

1. Run the fake-session test (`run-wolf-dp.cmd`): does the game call the player's session Send
   offline, and does a kind-2 clone walk on the replayed packets?
2. First two-account Steam test with the Owl puppet; then a bot-driven two-client test
   (playbook/automated-qa.md).
