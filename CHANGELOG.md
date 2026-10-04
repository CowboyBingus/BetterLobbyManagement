# Changelog

## v1.2 (2026-10-04)

- New: Simplified Chinese translation by joyrhyme (pull request #1); it shows when the game's Text Language is Simplified Chinese.
- The update hook is now Bingus Shared Runtime's guard, the error policy every CowboyBingus mod shares; its log lines name the mod, and its status is `BetterLobbyManagement.guard`.
- After an error in the game's update or another mod's, the mod pauses: it cancels a running action and puts the Lobby Region table and the scanner's value back. It resumes after 60 frames without such an error and applies its settings again.
- A pause keeps a CANCEL SOS you made: the first frame after it checks the session, the mission and the beacons again and turns a re-listed SOS off as before.
- The mod's own errors no longer stop it at the first one: each cancels the running action at the start of the next frame.
- 8 errors in a burst stop the mod for the session with everything put back; errors more than a minute apart never add up.
- The shutdown status keeps the first failure (`stopped after: <reason>`), and an error in the mod's own shutdown work can no longer keep the shutdowns of the game and other mods from running.
- The update passes every argument and return value through to the update it wraps, not just the frame time.
- Game module hashes come from the shared runtime's cache, so each module file is read once per session for all mods.
- Every Windows function the mod calls is declared under a private name, so another mod that declared the same functions first can no longer stop it from starting.
- A translation registry that another mod left incomplete, or a malformed translation pack, no longer stops the mod: missing parts are filled in and bad packs are skipped. A pack forces its language only with `force = true`.
- Mod Options Menu options are registered again when Mod Options Menu refused or failed the first attempt: up to 8 more tries in the first four minutes. Before, the options stayed missing for the session.
- With Bingus Shared Loader v19 the Mod Options Menu options are registered once, after every mod has started; with v18 the first-update registration and its retries stay.
- Measured in live play: 0.004 ms per frame in missions and 0.003 on the ship.

## v1.1 (2026-09-30)

- New: CANCEL SOS in the escape menu stops the SOS Beacon you called in as host, and keeps it stopped when a slot opens later.
- CANCEL SOS puts your privacy setting back on the lobby at once, so SOS Quickplay stops finding it; other players need no mod.
- CANCEL SOS gives the SOS Beacon its use back, so you can call in a new one.
- Translatable: buttons, dialogs, Mod Options Menu entries and the DISBAND chat line follow the game's Text Language when a translation is installed (see TRANSLATING.md).
- Player names on the buttons are upper-cased in every script the game's fonts carry.
- Tested in game with a host and one friend: after CANCEL SOS the mission left the other player's Galactic Map; the restored SOS Beacon use is not yet tested in game.
- Measured in live play: 0.005 ms per frame on the ship and in missions.

## v1.0 (2026-09-29)

First public release, as Better Lobby Management (the unpublished prototypes v0.1 to v0.4 were called Lobby Manager).

- DISBAND SQUAD and PROMOTE on the ship, in the escape menu's GAME tab, each confirmed in the game's own dialog. Both kick with the game's own player-menu KICK.
- PROMOTE announces the new host with the game's own line "*name* is the new squad leader", kicks them home, finds their new lobby and moves the squad there with the game's squad-Quickplay party join; other players need no mod. The escape menu closes by itself once the new host has left.
- New: 15 seconds after a move the log says how many of the squad reached the new host's session.
- DISBAND posts a chat line from you first. Squad Messages (Mod Options Menu) turns both messages off.
- Faster lobby scanner: the Galactic Map's lobby scanner recharges in 5 seconds instead of 20 (Scanner Recharge, 5 to 20 seconds, never longer than the game's own). Formerly the unpublished Fast Lobby Scanner.
- Lobby Region: My Continent Only for the Galactic Map scanner and quickplay.
- Measured in real play (the scanner still as its own mod): 0.003 ms per frame for the lobby tools and 0.001 ms for the scanner, on the ship and in missions.
- Tested live with a host and one friend; squads of three or four have not been tested.

## v0.4 (prototype, 2026-09-29, not published)

- Fixed: DISBAND SQUAD and PROMOTE crashed the host. The mod now makes the game run its own player-menu KICK (it opens the player's menu and completes the KICK hold), so the game kicks in its own update: the player sees the kick message and their Helldiver leaves with them. Calling the kick code directly, as v0.1 and v0.3 did, left the kicked player's Helldiver on the host's ship, and the game crashed when it unloaded their gear. Keep the escape menu open until the players are gone.
- Fixed: PROMOTE never found the successor's new lobby. The search sent a broken player id, because the game's Lua prints 64-bit numbers as `[cdata (deleted)]`.
- New: squad messages. Before PROMOTE's kick, every player sees the game's own line "*name* is the new squad leader"; before DISBAND, the squad gets a chat line from you. Mod Options Menu > Lobby Manager > Squad Messages turns them off.
- PROMOTE is faster: the kick follows the message after half a second, and the successor's lobby is searched for from half a second after they leave, every half second for 15 seconds, then every 4 seconds, for up to a minute (before: first search after 8 seconds, then every 4 seconds). In the live test the move took 6.9 seconds from the confirm, 5 of them the game's own join. The log says where the time went.
- Fixed: after PROMOTE the old host arrived on the new ship stuck in the escape menu's pose until Esc was pressed twice. The escape menu now closes by itself once the new host has left.
- Removed: HAND OVER. On the ship it was PROMOTE followed by leaving, and in a mission the game's own host migration already handles a host who leaves.
- Much shorter confirm dialog texts: the old ones spilled out of the box.
- Tested live with a host and one friend: DISBAND SQUAD and PROMOTE work without crashes, and the squad leader line appears before the kick.

## v0.3 (prototype, 2026-09-29)

- Fixed: confirming DISBAND SQUAD, PROMOTE or HAND OVER did nothing. The game hides its hold-to-confirm dialog in the very frame it is answered, and the mod dropped the answer when it saw the dialog hidden; it now reads the answer first.
- Fixed: a button could open the previous button's dialog (e.g. PROMOTE showing DISBAND SQUAD) when clicked while the last dialog was still fading out; the dialog now always shows the clicked button's title and text.
- Long button labels (long player names) scroll inside the button with the game's own marquee instead of spilling out.
- When nobody was picked, the PROMOTE / HAND OVER dialog says the player was chosen automatically and how to pick another.

## v0.2 (prototype, 2026-09-28)

- The actions moved from hotkeys into the escape menu: DISBAND SQUAD, PROMOTE *name* and HAND OVER TO *name* are native buttons on the GAME tab, each confirmed in the game's own dialog. The player whose menu you opened last is the one promoted. Mod Bindings Menu is no longer needed.
- Fixed a crash of v0.1's Promote: players are now removed only with the game's own kick (the removed player sees the kick message), one per frame. v0.1 used an internal removal the game only performs during host migration.
- Disband and Promote no longer have a "quiet" variant; the Successor and Disband Method options are gone (the menu picks the player).

## v0.1 (prototype, 2026-09-28)

- Disband Squad (ship): removes every other player the way the game does when a host leaves; option: the game's kick.
- Promote Successor and Hand Over And Leave (ship): sends the successor home, finds their new lobby by host id and moves the squad there with the game's squad-Quickplay party join; other players need no mod.
- Hand over in a mission: PlayFab lobby ownership to the successor, then leave; the game's host migration takes over.
- Lobby Region option: My Continent Only, written into the game's online configuration and restored byte for byte.
- Not yet tested in live multiplayer. Known crash on Promote (fixed in v0.2).
