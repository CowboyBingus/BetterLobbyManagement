# Better Lobby Management: how it works

Steam build 25480438 (game.dll SHA-256 `2E2C3B7C...F51E`, EXE 1.8.46015.0 `F5FEE03D...5F06`). Every address below is a game.dll RVA unless marked exe. The exact bytes the mod verifies are in [src/game.lua](../src/game.lua) (`G.CODE`, `G.NATIVES`, `G.EXE_CODE`, `G.ENGINE_SLOTS`, `G.PACKAGE_SLOTS`), [src/menu.lua](../src/menu.lua) (`M.CODE`, `M.NATIVES`), [src/chat.lua](../src/chat.lua) (`C.SEND`, `C.RPC`, `C.CODE`), [src/scanner.lua](../src/scanner.lua) (`S.CODE`), [src/sos.lua](../src/sos.lua) (`B.DEACTIVATE`, `B.SET_KEY`, `B.CODE`) and [src/region.lua](../src/region.lua) (`R.CODE`). A mismatch in `G` keeps the whole mod inactive; a mismatch in `M` disables only the menu buttons (and with them the actions), one in `C` only the squad messages, one in `S` only the scanner, one in `B` only CANCEL SOS, and one in `R` only Lobby Region.

The mod only reads memory, writes a few data fields (its buttons in the escape menu, a player menu's KICK hold timer, the escape menu's pending close, the lobby's send countdown, the scanner's recharge seconds, the Lobby Region table) and calls the game's own functions. It changes no game code.

## Escape menu buttons

The escape screen is `[[0x347CE38]+200]` (0 while the menu is closed; the screen and every widget in it are freed on close). Its GAME tab content is `screen+281480`:

| Offset | What |
| --- | --- |
| +1371024 + 85552·i | player card i (peer id +85504, state +85512: 1 focused, 2 player menu open; player menu +7144; KICK hold timer +85520, float seconds) |
| +1714272 | the container the buttons are attached to |
| +1714544 + 14920·i | button i (label text widget at +1928) |
| +1789912 / +1789913 | button count / one type byte per button |
| +1970000 | focus: 0-3 cards, 4 + i button i, 255 none |
| +1970288 | the shared confirm dialog (title widget +1496, body +2192, state +19372) |
| +1989648 / +1989649 / +1989650 / +1989651 | dialog inactive / opening / answered / confirmed |
| +1989812 | dialog fade (float; the tab takes no input while it is not 0) |
| +1832636 | squad panel state (2 while it takes input) |
| +1989808 | content hidden (another tab) |

The game uses button types 0-4 (0x18FB330); on the ship a host has types 1 and 3, in a mission 2 and 3 (and 0 with a squad), leaving two or three of the five slots free. Its select handler (0x18FAD50) and the dialog result dispatch (in 0x18FA5D0) act only on types 0-4; for any other type the select handler still runs its common tail, which shows the shared dialog. The rebuild 0x18FBF50 (host or mode changes) drops extra buttons.

- **Adding** a button: type 5 + action and the new count in one checked write, `add_child(list, button)` (0x144C5C0), `set_enabled(button, 1)` (0x14508E0), and the label through the `#COUNT` template (0xC67C7FAF with a string `COUNT` argument, `set_label` 0x143BF90 or 0x1441720 for wrapped text, `set_string_arg` 0x143C950), as Mod Options Menu does. Added again whenever missing; a refused write is retried only when the menu opens again. An action no longer offered is removed by calling the game's rebuild.
- **The dialog** is filled while hidden, when the focus lands on one of the mod's buttons: `0x179FDF0(dialog, #COUNT, #COUNT, 0xD94B7608, 0x8A36D40A, 1, 0)` (the game's own confirm and cancel labels and hold-to-confirm), string arguments on the title and body, `measure_text` (0x144E1A0). While showing, the setup would queue instead (state +19372), so it is never called then. When the focus leaves the mod's buttons, `clear_args` (0x143A0F0) takes the mod's text off.
- **The answer**: on the frame answered turns 1 with the focus on the mod's button, confirmed runs the action. The game hides a hold-to-confirm dialog in that same frame (0x13FB1A0: inactive at once, state 3 while it fades out), so the answer is read before the inactive flag. While it fades out the setup would queue, so a click on another of the mod's buttons meanwhile only retexts the dialog (title and body labels); when the dialog opens from a mod button, its title and body are always set for that button.
- **Long labels** scroll inside the 420 px button with the game's own marquee: `0x143C400(text, width)` sets text flag 0x20 at +672 and the visible width at +652 (button width less the 16 px inset on both sides, 388 px); the text layout (0x143BB20 → 0x143C5A0) then creates the scroller at +624 when the text is wider.
- **The successor** is the peer of the last card whose player menu was open (state 2), else the automatic choice.

## Session

The network context is `[0x347CEF0]`:

| Offset | What |
| --- | --- |
| +0xB398 | local peer id |
| +0xB3A8 | host peer id |
| +0x159B0 | host synchronizer (+24 hosting state, 1 = hosting; +28 transition step) |
| +0x162C8 | client synchronizer (+0x521 party join in progress) |
| +0x171C8 | join component (+0 state, 0 = idle) |
| +0x1D470 | lobby wrapper (+0 engine lobby; its +0x10 is the exe PlayfabLobby: +0x100 member count, +0x118 state, +0x120 PFLobbyHandle) |
| +0xC418 | the text chat (+0 enabled; 64-line history, first line +0x9590, count +0x9594) |
| +0x1F028 | the lobby browser handle every game search uses |
| +0x16390 | peer count |
| +0x16398 | 32-byte peer entries (player index at +20) |

Game mode is `[0x3326340]+0xAC21C`: 3 = ship, 4 = mission.

Peer ids are PlayFab `title_player_account` ids. They exceed 2^53, so the mod keeps them as 32-bit halves, formats them as decimal for lobby searches, and passes them to the game as `uint64_t` built from the halves.

## Why the ship cannot migrate in place

When the host goes, each client's synchronizer update (0x1083150) checks the game mode at 0x10834ED. On the ship it disconnects with HostConnectionBroken and goes home. In a mission it starts host migration toward the PlayFab lobby owner. That check runs on the clients, so a host-only mod cannot change it, and workspace mods never patch code.

## Removing players: the game's own player-menu KICK

Every kick is the game's own KICK, run by the game in its own update: `menu.game_kick` does
what a click on the player's card and a completed hold do.

1. Wait until the GAME tab takes input: shown tab 0, content not hidden, the dialog inactive and
   not opening, its fade (+1989812) at 0, the squad panel (+1832636) in state 2. After a confirm
   this is a fraction of a second, while the dialog fades out.
2. Find the card whose peer id (+85504, set by the tab's refresh through 0x18F7FC0) is the
   player; its player menu (+7144) must exist.
3. Focus the card (0x18FB240, the game's focus setter) and open its player menu
   (0x18F7E00 with state 2).
4. Write 100.0 into the card's KICK hold timer (+85520): one checked write.
5. The game's next update of the tab (0x18FA5D0 → 0x18F7000 → 0x18F7110 → KICK 0x18F7830) finds
   the timer past 3 s and kicks: log, `rpc_from_host_kicked`, `remove_peer(hs, peer, 1)`, card
   reset. The mod's update runs before it in the same frame.

One player at a time; the next waits until the previous one has left the session (5 s at most
per step). **The escape menu must stay open** until everyone is gone. Without the menu
integration, the actions refuse.

Why not call the kick code directly: v0.1 (`remove_peer`) and v0.3 (`kick_peer`, the same code the
KICK runs) crashed the host. In-game timelines recorded with the test build (2026-09-29) showed:

- Every removal releases the kicked player's loadout at once (PlayerHistory, `[0x347CE50]`), and
  the engine unloads one queued package per frame (exe package queue at `[[exe+0x1A10208]+0x400]`).
- The game's KICK despawns the kicked player's Helldiver in the same update: 24 in-use unit
  entries gone by the next frame, so the unloads find nothing in use.
- The same calls made from the mod's Lua update never reached the client (it stayed, with its
  Helldiver). From the render callback the message arrived and the client left, but the
  Helldiver still stayed. Either way the engine then unloaded a package the Helldiver still used
  (a weapon attachment) and crashed on purpose (exe 0x5F5C33, `mov [0], 0x7ED` in the unit
  unloader 0x5F5730).

## Disband

On the ship, every other player is kicked through the player menu, one at a time; the job ends
when they have all left the session (10 s at most). Guards: you are hosting, at least one other
player is present, no transition or join is running, you are on the ship, and the escape menu
integration is available. Verified live on 2026-09-29: the kick message, the player and their
Helldiver gone, no crash.

## Promote on the ship (party handover)

1. The squad is told (see Squad messages): the game's new squad leader notice names the
   successor T. The kick waits 0.5 s so the line shows first.
2. T is kicked through the player menu (T sees the game's kick message). T's game starts hosting
   a fresh lobby: owner migration Automatic, access Public while joinable. When T has left the
   session, the mod closes the escape menu (see Squad messages).
3. The first search runs 0.5 s after T has left the session, then 0.5 s after each result for
   15 s, then every 4 s; the mod gives up after 60 s. A search only starts when the browser is idle
   (0x1094780) and neither game request is searching (`[0x347CE80]` +0x688 quickplay and
   +0x56E0 scanner; state at +4). Each search clears the browser filters (engine lobby API
   `[0x3326308]+0xF8`, slot 0x198), adds `string_key2 eq <T in decimal>` (0x1094DF0) and
   starts (0x1094710). The mod then reads the count (0x1094820) and result 0 (0x1094890), whose
   connection string is at +72. Verified live: a host's lobby publishes its peer id as
   `string_key2` (logical key 3) with access policy 0 (Public).
4. `start_join(ctx+0x171C8, info, 2, 0, 5)` (0x108FF40) is the squad-Quickplay party join. The
   members' party-leader-join handler (0xBA2FC0 → 0x1090530) does not check the sender. T's join
   handler (0x108BC70) checks T's lobby privacy against the party leader, and checks for room.
5. Success is when the local host peer becomes T. If the host returns to hosting with an idle
   join, T refused (privacy or room). The join is abandoned after 60 s. The log line says where
   the time went (kick, finding the lobby, the join).
6. The other members follow the party join on their own. 15 s after the move the mod reads the
   session once and logs how many of the squad are in T's session ("3 of 4 players ..."), or that
   the host is no longer in it; nothing is read before then.

The successor is the player whose card was selected last, else a friend (0x13EFEA0), else the
lowest peer id. Verified live on 2026-09-29 (host and one friend, the friend's lobby Public):
T was kicked 0.5 s after the confirm, T's lobby was found by the second search 1.4 s after T
left, and the game's join took 5.0 s (0.6 to 5.1 s across three tests). Promote is ship only: in
a mission the game's own host migration already handles a host who leaves.

Peer ids go into the search as decimal built by arithmetic on the two 32-bit halves: the game
replaces Lua's `tostring`, which then prints 64-bit FFI numbers as `[cdata (deleted)]`.

## Squad messages

- **Promote: the new squad leader notice.** `rpc_from_host_notify_new_host` (0xB9E77C36), the
  message a host that took over after a mission migration sends (game.dll 0x108A2DC). The mod
  sends it with the game's RPC send 0xBDE430(hash, -1 = every other peer, args, 1); the one
  argument is {type 9, 8 bytes, &T's peer id}. A client accepts it only from its current host
  (0xB925A0); it shows T as the new leader in its HUD and adds event 5 {T} to the session's event
  ring (ctx+0x1F038), and the chat shows `<i=1>#NAME</i> is the new squad leader`. It does not
  change the client's host, so the kick that follows is accepted (verified live with one friend;
  in larger squads the other clients also send T their migration state, RPC 0xE15F07B2, which
  has not been seen live).
- **Disband: a chat line.** The chat box (0x186025D) hands the context's chat (ctx+0xC418) and
  the text to 0x1097560, which caps the text at 512 bytes, packs it into
  `rpc_ingame_chat_message` (0x9FDDB88E, one u32 array) for every other peer the host has not
  muted, and shows it in the host's own chat. The mod calls the same function. A chat line has no
  sender or type field, so every client shows it as "*host*: text".
- **Closing the escape menu after the kick.** The GAME tab closes on Esc with
  `PresenterManager set_pending_close(Main)`: byte +0x10 of `[[0x347CE28]+0x4388]`, the Main
  presenter (0 while the menu is closed; 0x18FA514..0x18FA54F). The mod sets it once T has left,
  and the game closes the menu in its next update. With the menu left open, the old host arrived
  on T's ship in the menu's wrist-device pose (v0.4-diag7).

## Texts and translations

Every text the mod shows is a key in `locales/en.lua`. `src/bingus_text.lua` (shared byte-for-byte
with the other CowboyBingus mods that show text) resolves a key for the current language: English,
then a translation shipped in `locales/<tag>.lua` (checked with `scripts/translations.py` by the
build and embedded in the entry), then translation packs registered in `_G.BingusTranslations`.
A translated text with invalid UTF-8, control characters or other `{placeholders}` than the English
one is refused alone and logged. Sentences are whole per variant (PROMOTE picked or automatic, a
Public or another lobby for CANCEL SOS), never assembled from pieces, so translators can order them.

The language is the game's Text Language: the game-state object `[0x3326340]` (the one `G.MODE`
is read from) holds at +705712 an index into the table of language records at `0x37C5650`, whose
code string is at +8 (`us` for English). The mod reads it through `api.read_bytes` (guarded,
16 bytes at most) at the first update, before it registers its Mod Options Menu entries, and each
time the escape menu opens with a button to show, since the menu's own OPTIONS tab is where the
language changes. If the read fails it uses Steam's language for the game, then English.

Mod Options Menu v1.1 and later (`ModOptionsMenu.version >= 2`) take each option text as a function
and call it when they build the MODS page, so option texts follow the language. v1.0 takes strings
with byte limits (label 64, mod name 40, choice 48, description 400): the mod passes the
translation if it fits, else the English text. ON and OFF stay the game's own words. The DISBAND
chat line is sent in the host's language: a chat line carries no language, so the squad reads it as
if the host had typed it.

## CANCEL SOS

The SOS system is `S = [0x3326BF0]`, the component manager of the SOS beacon (`SosComponent`): `S+8` is the SOS's
on byte, `S+24` the number of enabled SOS beacon components, `S+0` the engine time it went on (telemetry only).

| Function | What it does |
| --- | --- |
| `0x67A9A0(S)` SOS on | when hosting: lobby key 8 = 1, then the privacy setter with 0 (Public); then `S+0` = now, `S+8` = 1 |
| `0x67AA20(S)` SOS off | when hosting: lobby key 8 = 0, then the privacy setter with the privacy setting `[[0x3326340]+0xAC550]` (0 Public, 1 Friends Only, 2 Invite Only, 3 Friends and Clan); then `S+0` = 0, `S+8` = 0 |
| `0x134FCA0(ctx, 19, value)` privacy setter | key 19 = 0 while PlayFab's copy of the lobby still has key 8 on, else `value` |
| `0x10925D0(lobby, key, value)` key setter | the value as text in the lobby wrapper's cache (`+56 + 257·key`), and the key's bits in `+0x18` and `+0x20` when it changed (0x10924D0) |

The game calls them when the beacon activates (0x517FD0: SOS on while the session has fewer than 4 peers), when a
player joins (0xB5EA90: SOS off once the session holds 4), and when a player leaves (0xB5F140) or a new host takes
over (0x1087010): SOS on again whenever `S+24 > 0` and `S+8 == 0`. The SOS Beacon stratagem is available only to the
host, only while `S+8 == 0` and with fewer than 4 players (0x66C920, stratagem type 0x91).

Lobby key 8 is PlayFab `number_key8`: SOS Quickplay (a server-set share of Quickplay searches) requires it, and
Quickplay's result scoring reads it. Key 19 is `number_key6`: every lobby search requires 0 (Public). The lobby
wrapper (`ctx+0x1D470`) sends the keys whose `+0x20` bit is set when its countdown (`+0x1B80`, float) reaches 0,
then restarts the countdown from the online configuration (`[0x347CEE0]+0x3CE68`, 30 s live; 0x109411D). The
privacy setter and the host's join check (0x108BC70) read keys through the engine lobby (`PFLobbyGetLobbyProperty`),
so they see what has been sent, not the cache. So the game's own SOS off leaves key 19 at 0 while PlayFab still
shows the SOS on, which it does until the next send: after an SOS a Friends Only lobby stays Public.

What the mod does:

1. **Offer.** Hosting (host sync state 1, no transition) in a mission with `S+8 == 1`: CANCEL SOS (button type
   7). The dialog names the privacy setting the lobby returns to.
2. **Cancel.** SOS off (0x67AA20), key 19 = the privacy setting (0x10925D0), countdown = 0 (one checked write): the
   game sends keys 8 and 19 together in its update of the same frame.
3. **Keep it off.** Every frame, 6 direct loads. It ends with a new session or SOS object, a mode other than
   mission, `S+24 == 0` (no beacon), `S+24` growing (the host called in a new beacon, which the game lists), or the
   host changing. `S+8 == 1` with the same count is the game's re-arm, which ran in its update after the mod's:
   SOS off and key 19 again. If key 8's `+0x20` bit is clear, the lobby already sent the re-arm, and the countdown is
   set to 0 again (one checked write); otherwise the pending send already carries the cancel.

4. **The SOS Beacon's use back.** The SOS Beacon stratagem (type 0x91) has one use per mission (settings
   `[0x37CB600 + 8·0x91]`: `+0x50` uses per mission, -1 for none; `+0x94` shared by the squad; live: 1 and 0), and the
   availability check (0x66C880 -> 0x66D3D0) needs uses left in the player's slot. The slots live in the per-player
   records `[0x347CE50]` (32 records of 0x1690 bytes, their count at `+0x2D200`, the peer id at `+0`): up to 16 slots
   of 48 bytes from `+0x1C0` (type `+0`, uses left `+4`), their count at `+0x7C0` (0x66F060, 0x66F0F8). On a cancel the
   mod adds one use to the host's own SOS Beacon slot, never above the uses per mission, and not for a stratagem
   without a limit or with shared uses (one checked write). Calling in a new beacon then spends it as usual; the
   first beacon stays, and `S+24` grows, which ends the kept cancel.

Alone, CANCEL SOS is the only action, so the mod reads the game mode (2 loads), in a mission the escape menu (2
loads), and with the menu open the SOS (2 loads); the menu is followed until CANCEL SOS has left it, because the game
rebuilds its list when players come and go, not when the SOS stops.

## Galactic Map scanner

Formerly the standalone Fast Lobby Scanner ([src/scanner.lua](../src/scanner.lua)). Selecting a planet on
the Galactic Map runs the lobby scanner in the matchmaking object `M = [0x347CE80]`: a scan starts a lobby
search; when its results arrive (0.25 to 0.4 s later in live play) the scanner enters its recharging
state and copies a whole number of seconds from the online configuration `C = [0x347CEE0]`, field
`C+0x3CE4C`, into its countdown `M+0x10BD74` (float; the screen shows `SCANNER RECHARGING... N`):

```
0x1337fa8  mov   rax, [game.dll+0x347cee0]
0x1337fb2  movd  xmm0, [rax+0x3ce4c]
0x1337fd6  cvtdq2ps xmm0, xmm0
0x1337fd9  movss [rsi+0x10bd74], xmm0
```

Every frame the countdown drops by the frame time (0x133752E). `C+0x3CE4C` has no other reader; its
built-in default is 10, and the live value, 20, arrives with the configuration the game downloads at login
and then every 900 s.

Settings run from 5 to 20 s (default 5). Once per frame: load `[0x347CEE0]` (none yet: wait); a new object
is read once with ReadProcessMemory before any direct load; load `C+0x3CE4C`, and if it still holds the
value the mod settled on and the setting did not change, stop (two direct loads). Otherwise the game wrote
it (login, a configuration download) or the setting changed: the game's value is the one it wrote, and the
mod writes `min(setting, game value)` (one memory protection check, a direct store, a read back) and
shortens a running countdown that is longer. Game values outside 1 to 3600 are left alone; one below the
setting is kept. The code checks (`S.CODE`: the reader, the countdown update, and a scanner setter that ties
`[0x347CE80]` to the scanner fields) disable only the scanner when they do not match. The scanner stops,
putting the game's value back if the field still holds the mod's, after a failed write, an error, or more
than 500 rewrites in a session (another writer). The game rewriting its usual value every 15 minutes is
counted, not logged.

## Lobby Region

The search builder (0x133B1A3) adds `string_key6 ne <continent>` for each continent X where `lookup(config, 1, combine(0x673BC524, 0x5DF5D36B, own id, X id))` returns type 7 with value byte 0. The lookup is 0x12E7990. The combine is murmur-style (K = 0x5BD1E995): a child key is its parent key combined with its name. The continent ids are at 0x21D45B0. The player's continent comes from engine lobby API slot 0x40.

The config object `[0x347CDF8]` holds four inline tables of 0x6028 bytes. The lookup checks the peer-synced table (+0x12078) first, then OnlineOverrideData (+0xC050).

- **Table layout:** +0 entries pointer (table + 0x20), +8 capacity 512, +12 empty key 0, +16 probe multiplier 2, +20 root key, +24 entry count. A checksum of the 0x6000-byte entry block sits at +0x6020.
- **Entry layout (48 bytes):** key, name, parent key, key, type, value, previous and next keys, and a weight of 100.0f.

The server ships 16 false pairs; for NA, these are AF, AS and OC. With the option on, the mod plans every other pair of the player's row:

- An existing true flag in +0xC050 is flipped to false.
- A missing pair is added as an entry shaped like the server's, key written last, and the count is raised.
- A pair present in the peer-synced table is left alone, and the log says so.

All of this is one `write_words` with one page check. The mod never writes +0x12078: the host sends that table to its clients (0x12E6F60, 0x12E72E0, 0x12E7440).

The config download (0x103EF00) replaces +0xC050 with a whole new table only when the stored checksum (+0x12070) differs. Nothing inserts into it one entry at a time. So:

- The mod leaves the stored checksum alone, and identical downloads keep its flags.
- A changed download shows up as a new checksum. The check every 120 frames (config pointer, both checksums, the continent, and each written entry's key and value) then plans and writes again.
- Restoring puts flipped values back and removes added entries. That gives back the server's table byte for byte: no server entry's probe path crosses a slot that was empty when the table was built.

Restoring happens when the option is switched off, at shutdown and after any error.

## Per-frame cost

Pinned exactly by the tests with [tests/frame_budget.lua](../tests/frame_budget.lua):

| Path | Calls per frame |
| --- | --- |
| No session | 3 direct loads (2 of them the scanner's check, which every row below includes) |
| Alone on the ship | 6 direct loads (v1.0: 4; since v1.1 the game mode, for CANCEL SOS) |
| Alone in a mission | 8 direct loads; 10 with the escape menu open; 44 with it open and an SOS on |
| Client of another host | 6 direct loads |
| Hosting a squad, escape menu closed | 10 direct loads |
| Hosting a squad, escape menu open, nothing to do | 45 direct loads (one guarded read per new screen); in a mission 47, 49 with an SOS on |
| An SOS kept off (after CANCEL SOS) | 6 more direct loads |
| CANCEL SOS (once) | about 30 direct loads, up to 8 guarded reads, 2 native calls (the game's SOS off and key setter), 2 checked writes (2 VirtualQuery: the send countdown and the SOS Beacon's use) |
| A re-armed SOS switched off again (after a player leaves) | 13 direct loads, 2 native calls; 1 checked write only if the lobby already sent it |
| The scanner writing its value (login, a setting change, about every 15 min) | 1 VirtualQuery, a store and a read back; a longer running countdown is shortened with 1 guarded read and 1 more VirtualQuery |
| Adding the buttons (menu opened, after a rebuild) | 1 VirtualQuery for both + 3 native calls each |
| Focus onto a mod button | 1 dialog setup, 2 string arguments, 2 measures (once per change) |
| Lobby Region check (every 120 frames) | 12 direct loads + the engine's continent getter |
| Lobby Region write or restore | 1 VirtualQuery (about 0.29 ms in game), on change only |
| Promote: the squad leader notice (once) | 8 to 12 direct loads (2 to 4 players), 1 native call (the game's RPC send); no Windows calls |
| Disband: the chat line (once) | 8 to 12 direct loads, 1 guarded read (ReadProcessMemory), 1 native call (the game's chat send) |
| A kick (per player, action frames only) | about 10 direct loads, 2 native calls (focus, open the player menu), 1 checked write (1 VirtualQuery) |
| Promote: closing the escape menu (once, when T has left) | 2 direct loads, 2 guarded reads, 1 checked write (1 VirtualQuery) |
| Promote: the arrival check (once, 15 s after the move) | one session snapshot (about 20 direct loads); nothing before it is due |
| Promote waiting | 18 direct loads; searching adds 1 browser call per frame and 4 calls per attempt (1 PlayFab request every half second for 15 s, then every 4 s) |
| The game's Text Language (first update, and each time the escape menu opens with a button to show) | 5 guarded reads (ReadProcessMemory), no allocation beyond the texts of a rebuild |

Idle frames make no Windows calls and allocate nothing (measured over millions of hooks in the game's `lua51.dll`, interpreted and compiled). Compiled code in those tests, with the scanner's check: 2.6 to 5.0 KB (no session) and 5.2 to 5.6 KB (hosting a squad, menu closed), 2 to 5 traces per path (trace formation varies between runs); the action paths run once per action and stay interpreted. v1.1 against v1.0 in the same session (five runs each): the main loop trace alone on the ship grows by about 0.55 KB (the mode check) and is unchanged hosting a squad (4519 and 4536 bytes); an SOS kept off compiles 8.7 to 10.2 KB in 5 to 8 traces, only while one is kept. Measured in real play (study `real-play-lm04`, 2026-09-29, one 10-minute session): 0.0026 ms per frame on the ship and 0.0025 ms in client missions; the worst frames were the protection checks on action frames (0.30 and 0.33 ms). The mod's own compiled code was not measured in game.

## Tests

`python -B scripts/build.py` runs these inside the installed game's `lua51.dll`:

- **`test_game`:** binding refusals and session readers.
- **`test_lobby`:** every action and failure path against a simulated game.
- **`test_region`:** live keys, writes, restores and downloads.
- **`test_windows_api`:** the FFI layer, plus the update hook and region check on real memory.
- **`test_menu`:** the escape-menu buttons, dialog, answer, rebuilds, the Esc close and refusals.
- **`test_chat`:** the chat line and the new squad leader notice against simulated game natives.
- **`test_scanner`:** the scanner's writes, re-applies, limits and stops against fake memory, with exact per-frame calls.
- **`test_sos`:** CANCEL SOS against a simulated SOS system, lobby wrapper and stratagem records: the offer, the cancel's gates and calls, the lobby sent in the same frame, the SOS Beacon's use back (and when not), re-arms caught (also one already sent), what ends a cancel, exact per-frame calls.
- **`test_addon`:** menu wiring, Mod Options Menu, idle budgets and error containment.
- **`test_diag`:** the test build's recorder and kick modes against a simulated engine that crashes like the game (a direct kick crashes it; the game's own KICK does not).
- **`test_entry`:** the generated entry.
- **`test_package`:** the release ZIP.
