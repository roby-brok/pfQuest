> ## This is a fork
>
> [pfQuest](https://github.com/The-Kludge-Bureau/pfQuest) maintained for the
> [OctoWoW](https://octowow.st) server (WoW 1.12.1), by **Roby_Brok**.
> Part of my [OctoWoW addon setup](https://github.com/roby-brok/octowow-addons).
>
> It tracks [The Kludge Bureau's build](https://github.com/The-Kludge-Bureau/pfQuest) and adds
> two changes. Neither is OctoWoW-specific — both apply to any server, and if an upstream
> maintainer wants either one, it's theirs to take.
>
> **Configurable map icon size.** Two new options in the pfQuest settings, *World Map Node Scale*
> and *Minimap Node Scale*, both defaulting to `1.0`. They are multipliers rather than pixel
> sizes, so the larger cluster icons keep their proportion to the regular ones. A value that is
> missing, non-numeric, zero or negative falls back to `1.0`.
>
> **The quest tracker defaults to Current Zone.** Upstream defaults to *All Quests*, which puts
> every active quest in the tracker at once. This fork defaults to *Current Zone*, which shows
> only the quests relevant to where you are standing, plus anything you have explicitly watched.
> The mode itself is upstream's, only the default changed. Switch it any time from the dropdown
> at the top right of the world map. Existing installs are untouched — the config seeding never
> overwrites a value you already have saved.
>
> **Fix: the map could stop following the quest log.** pfQuest locks its scan loop for ten
> seconds at login so the burst of events on load can settle. Every `QUEST_LOG_UPDATE` arriving
> while that lock was active pushed it out another 1.5 seconds, and nothing ever cleared it — so
> any stream of events arriving faster than one per 1.5 seconds held the lock open indefinitely.
> `QUEST_LOG_UPDATE` fires in exactly that pattern while you accept and complete quests, which is
> when you most want the map to react. Once stuck, the map stopped updating until you forced it
> with `/db query`. The lock now has a hard ceiling and always releases.
> ([reported by ReikerEQ](https://github.com/roby-brok/pfQuest/issues/1))
>
> **Fix: the `[Translate]` button never worked.** Two independent reasons. Its `OnClick` passed
> the global `self` to `UIDropDownMenu_Initialize` and `ToggleDropDownMenu` — a 1.12 script
> handler has no `self`, the frame is `this` — so both got nil and the menu never opened,
> silently, because `scriptErrors` is off. And even repaired it would have shown nothing: the
> locale-freeing loop nils out every non-active locale table at load, so the quest text the
> button reads was already gone. The freeing loop now spares the `quests` locale tables (the
> only ones the button reads), which keeps 19.8 MB and still frees the 10.7 MB of `items`,
> `units` and `objects` locales that genuinely are unreachable. A new **Quest Text
> Translations** option, on by default, frees the rest and hides the button — a visible button
> that does nothing is the bug, not the feature.
>
> **Fix: seven quests that drew no objective pins.** The shipped database has no `["obj"]`
> data for them, so nothing was ever plotted — most visibly Un'Goro's three crystal pylons
> (*The Northern / Eastern / Western Pylon*), whose entire objective is to find the pylon and
> which pointed at nothing at all. Also *Lonebrow's Journal*, *The Torch of Retribution*,
> *Catalogue of the Wayward* and *A Bijou for Zanza*. The corrections live in
> `corrections.lua` and apply only where the field is absent, so a future database that
> ships real data takes precedence automatically.
>
> Worth recording how these were picked: a scan for quests whose objective text names a known
> game object produced 20 candidates, and **13 of them were wrong** on inspection — *Master
> Ryson's All Seeing Eye* resolves to an object sitting in Alterac Valley for a quest that
> happens in the Hinterlands, and several matched the object that *starts* the quest rather
> than the one you are sent to find. Only hand-verified entries are in the file.
>
> **New: `/db checkdb`.** Reports any quest in your log that has no objective data. Such a quest
> draws no map pins and says nothing about it, so the only way to notice was to stare at an
> empty map. A pure delivery quest legitimately has none; anything asking you to kill or collect
> should never be listed, and if it is, that is a database bug worth reporting.
>
> **Fix: five hardening fixes from the 2026-08-12 deep audit.** Defensive guards on paths
> that error on imperfect data, found by sweeping the whole addon: the quest tracker erred
> on custom-server objective rows that come back empty (three sites — tooltip, progress
> pass, cached draw); the database browser crashed drawing a favourited quest whose id the
> current database no longer names, and on vendor tooltips for units without a localized
> name; the minimap-arrow probe at login called `strlower` on model-less frames; and the
> pfUI url-copy path assumed a chat module that can be disabled. None of them changes
> behaviour on good data.
>
> **Quest database website is settable.** Each database pack assigns `pfQuest.dburl` in its own
> patchtable, so with more than one installed the winner is whichever folder sorts last —
> `pfQuest-turtle` silently overrides `pfQuest-octo`, and you get the wrong server's site when
> you click through to a quest. There is now a **Quest Database Website** box in the settings that
> overrides all of them, and defaults to OctoWoW's database. Clear the box to defer to the pack.
>
> ### Installing this fork
>
> **The download links further down this page point at the upstream repository, not at this
> fork.** They are upstream's own instructions, left intact. To get the version with the changes
> above, download from here instead:
>
> 1. **[Download this fork](https://github.com/roby-brok/pfQuest/archive/refs/heads/master.zip)**
> 2. Unpack the zip
> 3. **Rename the folder `pfQuest-master` to `pfQuest`** — this step is not optional
> 4. Move `pfQuest` into `Wow-Directory\Interface\AddOns`
> 5. Restart WoW
>
> Step 3 matters because WoW only loads an addon when the folder name matches the `.toc` inside
> it. A folder called `pfQuest-master` containing `pfQuest.toc` is skipped silently — no error,
> no entry in the addon list, it simply never runs.
>
> ### You probably also want a database pack
>
> pfQuest ships the standard Vanilla / TBC / WotLK database and works on its own. Servers with
> custom content need an extra pack on top, downloaded separately — **this repository does not
> include one**:
>
> | For | Pack | Rename the unpacked folder to |
> |---|---|---|
> | [OctoWoW](https://octowow.st) | **[pfQuest-octo](https://github.com/roby-brok/pfQuest-octo)** | `pfQuest-octo` |
>
> Same trap as above — the zip unpacks as `pfQuest-octo-master` and does not load until renamed.
>
> **Install one pack, not two.** Packs assign the database namespace outright rather than
> merging into it, so with two installed the one whose folder sorts last simply replaces the
> other — `pfQuest-turtle` wins on the letter *t*, and everything the Octo pack parsed at
> login is thrown away. Memory and load time spent on data that never gets used, plus quest
> links pointing at the wrong server.
>
> That is why the pack above is a merged one: the TurtleWoW database as the base with the
> Octo pack folded in on top, so there is nothing to choose between. It supersedes both
> [paokkerkir/pfQuest-octo](https://github.com/paokkerkir/pfQuest-octo) (the original Octo
> pack, and the source of the corrections) and
> [pfQuest-turtle](https://github.com/The-Kludge-Bureau/pfQuest-turtle) — **remove
> `pfQuest-turtle` if you have it.**

> ### Credits
>
> Everything below this box, and effectively all of the addon, is other people's work:
>
> * **[Shagu](https://github.com/shagu)** — wrote pfQuest, and [ShaguQuest](https://shagu.org/ShaguQuest/) before it.
> * **[The Kludge Bureau](https://github.com/The-Kludge-Bureau/pfQuest)** — maintains the continuation this fork is built on.
> * **[VMaNGOS](https://github.com/vmangos)**, **[CMaNGOS](https://github.com/cmangos)** and **[MaNGOS Extras](https://github.com/MangosExtras)** — the database behind it.
> * **[paokkerkir](https://github.com/paokkerkir/pfQuest-octo)** — the Octo database pack.
>
> Use the upstream repository unless you specifically want the two changes above.

---

# pfQuest

<img src="https://raw.githubusercontent.com/The-Kludge-Bureau/pfQuest/main/_img/mode.png" float="right" align="right" width="25%">

pfQuest is a quest helper and database browser for World of Warcraft Vanilla (1.12), The Burning Crusade (2.4.3), and Wrath of the Lich King (3.3.5a). Accept a quest and the relevant NPCs, monsters, and objects are automatically pinned on your world map and minimap. Open the database browser to look up any unit, item, or game object in the game — or use the chat commands to build macros for tracking gathering nodes.

The goal is to provide an accurate in-game equivalent of [AoWoW](http://db.vanillagaming.org/) or [Wowhead](http://www.wowhead.com/), not a quest guide or turn-by-turn assistant. The Vanilla database is powered by [VMaNGOS](https://github.com/vmangos). The Burning Crusade version uses data from [CMaNGOS](https://github.com/cmangos) with translations from [MaNGOS Extras](https://github.com/MangosExtras).

pfQuest is the successor of [ShaguQuest](https://shagu.org/ShaguQuest/), written from scratch with no dependency on any specific map or questlog addon. It is designed to work alongside the default UI and any other addon. If you run into a conflict, please open an issue on the bugtracker.

You can check the [Latest Changes](https://github.com/The-Kludge-Bureau/pfQuest/commits/main) page to see what has changed recently.

## Before You Install

pfQuest ships a complete database of all spawns, objects, items, and quests. The full package is approximately 80 MB and is loaded into memory once at login — memory usage is stable after that and does not grow during play.

On Vanilla clients, WoW will show a warning if an addon exceeds the default memory limit. **Before installing, set Script Memory to `0` (no limit)** in the AddOns panel of the character selection screen. This is a one-time step. A [screenshot showing where to find the setting](https://raw.githubusercontent.com/The-Kludge-Bureau/pfQuest/main/_img/addons-memory.png) is available if you are unsure where to look.

## Downloads

### World of Warcraft: Vanilla

1. **[Download pfQuest (full)](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-full.zip)**
2. Unpack the zip file
3. Move the `pfQuest` folder into `Wow-Directory\Interface\AddOns`
4. Restart WoW

Slim packages (single language): [English](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-enUS.zip) · [German](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-deDE.zip) · [French](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-frFR.zip) · [Spanish](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-esES.zip) · [Korean](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-koKR.zip) · [Chinese](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-zhCN.zip) · [Russian](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-ruRU.zip)

### World of Warcraft: The Burning Crusade

1. **[Download pfQuest (full)](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-full-tbc.zip)**
2. Unpack the zip file
3. Move the `pfQuest-tbc` folder into `Wow-Directory\Interface\AddOns`
4. Restart WoW

Slim packages (single language): [English](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-enUS-tbc.zip) · [German](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-deDE-tbc.zip) · [French](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-frFR-tbc.zip) · [Spanish](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-esES-tbc.zip) · [Korean](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-koKR-tbc.zip) · [Chinese](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-zhCN-tbc.zip) · [Russian](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-ruRU-tbc.zip)

### World of Warcraft: Wrath of the Lich King

> [!IMPORTANT]
>
> **This is a BETA version of pfQuest**
>
> It is able to run on a WotLK (3.3.5a) client, but does not yet ship a WotLK database.
> All available content is limited to Vanilla & TBC as of now.

1. **[Download pfQuest (full)](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-full-wotlk.zip)**
2. Unpack the zip file
3. Move the `pfQuest-wotlk` folder into `Wow-Directory\Interface\AddOns`
4. Restart WoW

Slim packages (single language): [English](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-enUS-wotlk.zip) · [German](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-deDE-wotlk.zip) · [French](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-frFR-wotlk.zip) · [Spanish](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-esES-wotlk.zip) · [Korean](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-koKR-wotlk.zip) · [Chinese](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-zhCN-wotlk.zip) · [Russian](https://github.com/The-Kludge-Bureau/pfQuest/releases/latest/download/pfQuest-ruRU-wotlk.zip)

### Development Version

The development version includes databases for all languages and all client expansions in a single folder. It will work in both Vanilla and TBC mode depending on the folder name. Due to the amount of included data, expect higher RAM and disk usage and slightly longer load times compared to the release packages.

- Clone via Git: [`https://github.com/The-Kludge-Bureau/pfQuest.git`](https://github.com/The-Kludge-Bureau/pfQuest.git)
- Download as zip: **[main.zip](https://github.com/The-Kludge-Bureau/pfQuest/archive/main.zip)**

## Controls

Nodes on the world map can be **clicked** to cycle through display colors, making it easy to mark progress visually.

When multiple spawn points are close together they are grouped into a single **cluster** icon to reduce clutter. Holding **\<ctrl\>** on the world map temporarily breaks clusters apart so you can see individual locations. Hovering the minimap and holding **\<ctrl\>** hides minimap nodes entirely.

| Action                                             | Result                                  |
| -------------------------------------------------- | --------------------------------------- |
| **Click** a node on the world map                  | Cycle node color                        |
| **\<Shift\>-click** a quest giver on the world map | Remove completed quest from the map     |
| Hold **\<Ctrl\>** on the world map                 | Temporarily expand clusters / hide them |
| Hover minimap + hold **\<Ctrl\>**                  | Temporarily hide all minimap nodes      |
| **\<Shift\>-drag** the minimap button              | Move the minimap button                 |
| **\<Shift\>-drag** the arrow frame                 | Move the quest arrow                    |

## Map & Minimap Nodes

<img src="https://raw.githubusercontent.com/The-Kludge-Bureau/pfQuest/main/_img/arrow.png" width="35.8%" align="left">
<img src="https://raw.githubusercontent.com/The-Kludge-Bureau/pfQuest/main/_img/minimap-nodes.png" width="59.25%">
<img src="https://raw.githubusercontent.com/The-Kludge-Bureau/pfQuest/main/_img/map-quests.png" width="55.35%" align="left">
<img src="https://raw.githubusercontent.com/The-Kludge-Bureau/pfQuest/main/_img/map-spawnpoints.png" width="39.65%">

Quest objectives, spawn points, and points of interest are plotted directly on the world map and minimap. The directional arrow (top left) points toward your nearest active objective and updates as you move. Nodes are color-coded by type and quest state — available quests, objectives in progress, and turn-in locations each use distinct icons.

## Auto-Tracking

<img src="https://raw.githubusercontent.com/The-Kludge-Bureau/pfQuest/main/_img/map-autotrack.png" float="right" align="right" width="30%">

pfQuest supports four tracking modes that control how quest objectives are shown on the map. The active mode is selected from the dropdown menu in the top-right corner of the world map.

#### All Quests

Every quest in your log is automatically shown and updated on the map. This is the default mode.

#### Tracked Quests

Only quests you have manually tracked via Shift-Click in the questlog are shown and updated.

#### Manual Selection

Only quests you have explicitly shown using the **Show** button in the questlog are displayed. Completed objectives are still automatically removed from the map.

#### Hide Quests

Same as Manual Selection, but quest givers are also hidden and completed objectives remain on the map. This mode makes no changes to existing map nodes.

## Database Browser

<img src="https://raw.githubusercontent.com/The-Kludge-Bureau/pfQuest/main/_img/browser-spawn.png" align="left" width="30%">
<img src="https://raw.githubusercontent.com/The-Kludge-Bureau/pfQuest/main/_img/browser-quests.png" align="left" width="30%">
<img src="https://raw.githubusercontent.com/The-Kludge-Bureau/pfQuest/main/_img/browser-items.png" align="center" width="33%">

The database browser lets you search and bookmark units, game objects, items, and quests from the full pfQuest database. Open it by clicking the pfQuest minimap icon or with `/db show`. Each tab shows up to 100 results — use the scroll wheel or the up/down arrows to navigate. If an entry is marked with `[?]`, that object or unit is not currently available on your realm.

## Questlog Integration

### Questlinks

<img src="https://raw.githubusercontent.com/The-Kludge-Bureau/pfQuest/main/_img/questlink.png" float="right" align="right" width="30%">

On servers that support questlinks, Shift-clicking a selected quest in the questlog inserts a clickable questlink into chat. These links are compatible with those produced by [ShaguQuest](https://shagu.org/ShaguQuest/), [Questie](https://github.com/AeroScripts/QuestieDev), and [QuestLink](http://addons.us.to/addon/questlink-0). Links sent between pfQuest users are locale-independent and use the Quest ID directly.

Some servers (e.g. Kronos) block questlinks entirely. In that case, disable the questlink feature in the pfQuest settings and the quest name will be inserted as plain text instead.

Hovering a questlink displays a tooltip showing your current progress, the objective text, the full quest description, the suggested level, and the minimum required level.

### Questlog Buttons

<img src="https://raw.githubusercontent.com/The-Kludge-Bureau/pfQuest/main/_img/questlog-integration.png" align="left" width="300">

Each quest in the questlog has four additional buttons for manual map control. These buttons only affect nodes placed by pfQuest's quest tracking — they do not touch anything you have placed manually through the database browser.

**Show** — Adds the quest objectives for the selected quest to the map.

**Hide** — Removes the selected quest from the map.

**Clean** — Removes all nodes placed by pfQuest from the map.

**Reset** — Restores the default node visibility to match the current auto-tracking mode (e.g. re-shows all quests if the mode is set to "All Quests").

## Chat / Macro CLI

<img src="https://raw.githubusercontent.com/The-Kludge-Bureau/pfQuest/main/_img/chat-cli.png">

All pfQuest features are accessible from chat or macros using `/db`. For example, `/db object Iron Deposit` plots all Iron Deposit locations on the map, and `/db track mines 150 225` shows only mines that require a Mining skill between 150 and 225. The commands `/shagu`, `/pfquest`, and `/pfdb` are all aliases for `/db`.

### General

```
/db lock                Lock/unlock the map tracker position
/db tracker             Show the map tracker
/db journal             Show the quest journal
/db arrow               Toggle the quest arrow
/db show                Open the database browser
/db config              Open the settings panel
/db locale              Display the active addon locales
/db scan                Scan the server for custom items
/db debug               Toggle debug output
```

### Questing

```
/db reset               Reload all quest nodes on the map
/db query               Query the server for completed quests
/db clean               Remove all database search results from the map
/db checkdb             List quests in your log that have no objective data
```

### Database Search

```
/db unit <name>         Find spawn locations for a unit (e.g. Thrall)
/db object <name>       Find locations for a game object (e.g. Iron Deposit)
/db item <name>         Find units and objects that drop an item (e.g. Runecloth)
/db vendor <name>       Find vendors that sell a specific item (e.g. Jagged Arrow)
/db quest <name>        Search for a quest by name
```

### Tracking Lists

Tracking lists let you pin all instances of a category on the map at once.

```
/db track               Show all available tracking lists
/db track clean         Remove all tracked list nodes from the map
/db track <list>        Show all objects in <list> on the map
/db track <list> clean  Remove all objects in <list> from the map
```

The `mines` and `herbs` lists support an optional skill range and an `auto` shortcut that uses your current skill level:

```
/db track mines         Show all mines
/db track mines auto    Show mines within your current skill range
/db track mines 50 150  Show mines requiring skill 50–150
/db track mines clean   Remove all mine nodes from the map
```

Available tracking lists: `auctioneer`, `banker`, `battlemaster`, `chests`, `fish`, `flight`, `herbs`, `innkeeper`, `mailbox`, `meetingstone`, `mines`, `rares`, `repair`, `spirithealer`, `stablemaster`, `vendor`
