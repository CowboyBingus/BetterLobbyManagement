# Better Lobby Management v1.0

Host tools that Helldivers 2 does not have, in the game's own escape menu:

- **DISBAND SQUAD.** On the ship, every other player is kicked back to their own ship in one confirmation, instead of kicking them one by one or quitting.
- **PROMOTE *name*.** On the ship, another player becomes the squad's host and the whole squad moves to their ship. You stay as a member.
- **Squad messages.** Before PROMOTE's kick, every player sees the game's own line "*name* is the new squad leader", the one it shows after a host migration. Before DISBAND, the squad gets a chat line from you. Option: *Squad Messages*.
- **Faster lobby scanner.** The Galactic Map's lobby scanner recharges in 5 seconds instead of the game's 20, so the squads on the globe refresh sooner. Option: *Scanner Recharge (Seconds)*, 5 to 20; never longer than the game's own. (Formerly the unpublished Fast Lobby Scanner.)
- **Lobby Region.** An option: *My Continent Only* limits the Galactic Map scanner and quickplay to lobbies hosted on your continent.

**Only the host needs the mod.** The other players need no mod. The squad moves with the game's own squad-Quickplay party join.

> Tested live with a host and one friend on 2026-09-29: DISBAND SQUAD and PROMOTE work without crashes, the friend sees "*name* is the new squad leader" before the kick, and the host arrives on the new ship ready to play. Squads of three or four have not been tested: the other members should follow through the game's own party join, and the log says how many arrived. Steam build **25480438** / EXE 1.8.46015.0 only. The mod checks the game.dll and EXE hashes and every piece of game code it depends on before it does anything; on any other build it stays inactive and says why in its log.

## Install

Close the game, import `Better-Lobby-Management-v1.0.zip` and [Bingus Shared Loader v18 or newer](https://github.com/CowboyBingus/BingusSharedLoader/releases/latest) into Arsenal or HD2MM, enable both and deploy. Keep the shared loader as the winning Wwise startup replacement. [Mod Options Menu](https://github.com/CowboyBingus/ModOptionsMenu/releases/latest) is optional (Squad Messages, Scanner Recharge, Lobby Region). The mod is also an option of [Vanilla Plus Megapack](https://github.com/CowboyBingus/VanillaPlusMegapack/releases/latest); enable only one copy.

## Use

Host a squad on your ship and open the escape menu. The GAME tab lists your squad and, under the game's own buttons, the mod's two. Each opens the game's own confirm dialog (hold to confirm).

**Keep the escape menu open until the kicked players are gone.** The mod kicks with the game's own player-menu KICK, so you may see a player's menu open for a moment. After PROMOTE the escape menu closes by itself once the new host has left.

*name* is the player whose card you selected last. Before that, it is a friend of yours if there is one, otherwise the player with the lowest id. The button shows who.

## How it works

**Removing players.** The mod makes the game run its own KICK: it opens the player's menu on the GAME tab and completes the KICK hold, and the game kicks in its own update, exactly as when you hold KICK yourself. The player sees the game's kick message, and their Helldiver leaves your ship with them. One player at a time. (Prototypes that called the kick code directly left the kicked player's Helldiver on the host's ship and crashed the game when it unloaded their gear. See [how it works](docs/TECHNICAL.md).)

**Promote on the ship.** The game cannot move a ship to a new host: when the host leaves, every client goes home. So the mod moves the squad instead:

1. It tells the squad. The game's own notice names the successor as the new squad leader; every player's game shows it as one of its own system lines (no colon), like its join and kick lines.
2. Half a second later it kicks the successor. Their game returns to their own ship and hosts a new, joinable lobby. The escape menu then closes, as if you pressed Esc.
3. It finds that lobby. Every lobby publishes its host's id, and the mod searches for it through the game's own lobby browser: from half a second after the successor left, every half second for 15 seconds, then every 4 seconds, for up to a minute. It never overlaps the game's own searches.
4. It starts the game's squad-Quickplay join into that lobby. The rest of the squad follows through the game's own party protocol. 15 seconds after the move, the log says how many of the squad reached the new host's session.

In the live tests the successor was kicked 0.5 seconds after the confirm, their new lobby was found 1.4 to 2.8 seconds after they left, and the game's join took 0.6 to 5 seconds.

A successor whose lobby is Friends Only or Invite Only refuses a squad led by someone who is not their friend. The squad then stays with you, and the log says so. A member whose game does not follow gets the game's usual "host left" handling and returns to their own ship.

**Disband.** A chat line from you ("The host disbanded the squad."), then the kicks, one player at a time.

**Faster lobby scanner.** When a scan's results arrive, the scanner copies its recharge seconds from the game's online configuration into its countdown ("SCANNER RECHARGING... N"). The mod keeps that one configuration value at your setting, never above the game's own, and writes it again whenever the game downloads its configuration (at login and about every 15 minutes). Every scan is one lobby search, so 5 seconds searches at most about four times as often as the game does; that is also the shortest setting.

**Lobby Region.** The game already excludes some continent pairs on every lobby search (for example, North America excludes Africa, Asia and Oceania). The mod marks every other continent as excluded for your own continent, in the game's online configuration. There are no extra searches; fewer lobbies may be listed. The table is put back exactly as the server sent it when you switch the option off, when the game closes, or after any error.

**Not covered.** A host who crashes or quits the game: the squad disbands as it does today. In a mission the game's own host migration already handles a host who leaves, so the mod adds nothing there.

## Cost

Measured in real play (one 10-minute session with a PROMOTE, a DISBAND and a joined mission, when the scanner was still its own mod): 0.003 ms per frame for the lobby tools and 0.001 ms for the scanner, on the ship and in missions. The worst frames, 0.3 ms, were the memory protection checks on action frames. Per frame: 3 to 6 memory loads when not hosting a squad (2 of them the scanner's), 10 while hosting one with the escape menu closed, and no Windows calls. With the escape menu open while hosting, the mod reads the menu (45 loads) and calls the game only when something changes. The memory protection check (about 0.3 ms in game) happens when the mod adds its buttons (once per menu opening), once per kicked player, once per promote (closing the menu), when the scanner writes its value (at login, a setting change and about every 15 minutes), and when Lobby Region writes its table. Tests pin these counts exactly; see [how it works](docs/TECHNICAL.md).

`%LOCALAPPDATA%/CowboyBingus/Helldivers2/Logs/BetterLobbyManagement.log` records the startup check, each action step, why an action stopped, where a promote's time went, how many of the squad arrived and each change of the scanner's value.

## Build and test

Clone [Bingus Shared Loader](https://github.com/CowboyBingus/BingusSharedLoader) beside this repository (or set `BINGUS_SHARED_LOADER` to its path). `python -B scripts/build.py` assembles `build/better_lobby_management.lua` from `src/` (`scripts/entry.py`), runs every test inside the installed game's `lua51.dll` (the game is not started) and writes `releases/Better-Lobby-Management-v1.0.zip`. `--diag` builds the diagnostic test build instead: it adds a read-only recorder of the kicks and the Kick Test and Promote Notice options.

[Changes](CHANGELOG.md) · [How it works](docs/TECHNICAL.md)

**AI disclosure:** Claude Opus 5.5 assisted with research, implementation, tests and documentation.
