# Find My Stuff! — ESO AddOn

> **Title:** |cFF0000Find|r |cffff00My|r |c00ffffStuff!|r
> **Author:** Vixen Hunny
> **Version:** 1.0.0
> **Description:** Keeping track of all your stuff from other characters!
> **API Version:** 101049 101050
> **Depends On:** LibAddonMenu-2.0
> **Saved Variables:** `FindMyStuffSV`

`FindMyStuff` is an Elder Scrolls Online AddOn with two complementary jobs:

1. **Cross-character inventory tracking** — it silently records what every character on your account is carrying (bags, bank, subscriber bank, craft bag) so you can ask *"which of my characters has the Night Mother's Gaze?"* without logging into each one.
2. **Full item-database search** — it can brute-force scan the entire ESO item ID space (IDs `1`–`184000`) to find *any* item in the game by name, trait, equipment slot, weapon type, or armor weight, even items you don't currently own. This is useful for looking up item links for guides, trading, or verifying exact trait/weight combinations.

This README documents what the AddOn does and walks through `FindMyStuff.lua` section by section.

---

## Table of Contents

- [Installation](#installation)
- [File Overview](#file-overview)
- [Slash Commands](#slash-commands)
- [How It Works — Conceptual Overview](#how-it-works--conceptual-overview)
- [Code Walkthrough](#code-walkthrough)
  - [1. Manifest (`FindMyStuff.addon`)](#1-manifest-findmystuffaddon)
  - [2. Module Bootstrap & State](#2-module-bootstrap--state)
  - [3. Lookup Tables (Traits, Equip Slots, Weapons, Armor)](#3-lookup-tables-traits-equip-slots-weapons-armor)
  - [4. Equipment Synonym Table](#4-equipment-synonym-table)
  - [5. The Item-Link Suffix & Default Quality/Level](#5-the-item-link-suffix--default-qualitylevel)
  - [6. `bridge()` — Preparing a Database Search](#6-bridge--preparing-a-database-search)
  - [7. `split_str()` — Tokenizing Search Text](#7-split_str--tokenizing-search-text)
  - [8. `FindMyStuff.UseName()` — The Search Loop](#8-findmystuffusename--the-search-loop)
  - [9. Dead/Legacy Helpers: `ShowItem`, `BreakItem`](#9-deadlegacy-helpers-showitem-breakitem)
  - [10. `TableBuilder()` / `FindMyStuff.BuildTable()` — Caching Item Names](#10-tablebuilder--findmystuffbuildtable--caching-item-names)
  - [11. `RangeCheck()` — Parsing Flags Out of User Input](#11-rangecheck--parsing-flags-out-of-user-input)
  - [12. The Dump System (`Dump`, `DumpBridge`, `FindMyStuff.DumpBuilder`)](#12-the-dump-system-dump-dumpbridge-findmystuffdumpbuilder)
  - [13. `GradeItem()` / `AssembleItem()` — Quality Tier Guessing](#13-gradeitem--assembleitem--quality-tier-guessing)
  - [14. `SetQuality()` — Calibrating Searches to a Real Item](#14-setquality--calibrating-searches-to-a-real-item)
  - [15. `IFHelp()` — In-Game Help Text](#15-ifhelp--in-game-help-text)
  - [16. Inventory Tracking System](#16-inventory-tracking-system)
  - [17. `FindMyStuff:Initialize()` and AddOn Bootstrapping](#17-findmystuffinitialize-and-addon-bootstrapping)
- [Saved Variables Structure](#saved-variables-structure)
- [Known Quirks & Observations](#known-quirks--observations)
- [Glossary of ESO Concepts Used](#glossary-of-eso-concepts-used)

---

## Installation

1. Copy the `FindMyStuff` folder into your ESO AddOns directory, e.g.:
   `Documents\Elder Scrolls Online\live\AddOns\FindMyStuff\`
2. Ensure the dependency **LibAddonMenu-2.0** is also installed (declared in `## DependsOn:` in the manifest, though no settings panel is implemented yet in this version).
3. Enable the AddOn from the in-game AddOns menu and reload the UI (`/reloadui`).
4. Log in once per character you want tracked — the inventory scanner records data **per character**, so alts you haven't logged into yet won't have data until you do.

## File Overview

| File | Purpose |
|---|---|
| [FindMyStuff.addon](<c:/Users/Admin/AppData/Local/Elder Scrolls Online/pccert/CachedData/AddOnsManaged/FindMyStuff/FindMyStuff.addon>) | The ESO AddOn manifest: title, author, description, supported API versions, saved-variable table name, and dependencies. |
| [FindMyStuff.lua](<c:/Users/Admin/AppData/Local/Elder Scrolls Online/pccert/CachedData/AddOnsManaged/FindMyStuff/FindMyStuff.lua>) | The entire implementation — lookup tables, the brute-force item database search engine, the recipe/furnishing "dump" tool, the quality-grading helper, and the cross-character inventory scanner. |

## Slash Commands

These are the commands that are **actually registered** (`SLASH_COMMANDS[...]`) near the bottom of the file, inside `FindMyStuff:Initialize()`:

| Command | Handler | Purpose |
|---|---|---|
| `/finditem <name>` | `FindInInventory` | Searches the **saved cross-character inventory** for an item by (partial, case-insensitive) name and lists which character(s) have it, with quantity, quality color, trait, bind status, and known/unknown status. |
| `/fmsscan` | `FindMyStuff.ScanInventory` | Manually forces a rescan/save of the **current character's** bags, bank, subscriber bank, and craft bag. |
| `/fmsdump` | `Dump` | Scans the entire item ID space and dumps all furniture-recipe, provisioning-recipe, or reagent item IDs into `FindMyStuffSV` for inspection. |
| `/fmssetquality` | `SetQuality` | Takes an item link and adopts its quality/level as the default used when assembling test item links for `/findindb`-style database searches. |
| `/fmshelp` | `IFHelp` | Prints detailed in-game usage help (see below) to the chat window. |

> **Note:** The in-game help text (see [`IFHelp()`](#15-ifhelp--in-game-help-text)) also describes commands named `/findindb`, `/findid`, `/findbyname`, `/showitem`, `/findname`, `/findbyid`, `/breakitem`, `/gradeitem`, and `/ifdump` / `/ifsetquality`. **None of these are currently bound** via `SLASH_COMMANDS` in this version of the script — only the five commands in the table above are live. The underlying functions they'd call (`RangeCheck`, `ShowItem`, `BreakItem`, `GradeItem`) still exist in the code (and `RangeCheck` is still invoked internally by the table-building flow), but the help text appears to be left over from an earlier iteration of the AddOn where the full item-database search was exposed under different command names. See [Known Quirks & Observations](#known-quirks--observations).

## How It Works — Conceptual Overview

FindMyStuff is really **two separate subsystems** sharing one file and one saved-variables table:

### A. Cross-Character Inventory Tracker (the "live" feature)
- On `EVENT_PLAYER_ACTIVATED` (i.e., whenever a character finishes loading into the world) and on `EVENT_LOOT_UPDATED` (whenever you loot something), the AddOn waits 3 seconds (`zo_callLater`) and then calls `FindMyStuff.ScanInventory()`.
- `ScanInventory()` iterates every slot in the character's backpack and worn-equipment bags, plus the bank/subscriber bank/craft bag (which are account-shared, so they're stored under a synthetic `"[Bank/Craft Bag]"` pseudo-character), and records item name, quantity, quality, trait, bind status, and "known" status (for recipes/motifs/collectibles) into `FindMyStuffSV.charInventory[characterName]`.
- `/finditem <text>` then does a simple substring search across every stored character's inventory table and prints a nicely colored report.

### B. Full Item-Database Brute-Force Search (the "archaeology" feature)
ESO identifies items by a numeric ID inside an item link string, e.g. `|H1:item:61225:364:50:0:...|h|h`. There is no built-in API to search "all items by name" — you can only ask the game "what is the display name of item ID *N*?" via `GetItemLinkName`. So this AddOn works by:
1. **Building a cache**: looping from item ID `1` to `184000` (the `pmax` constant) and asking the client for each item's name via a synthetically constructed item link, storing the lowercase name string in `FindMyStuff.ItemTable.items[id]`. Invalid/non-existent IDs come back as `"h"` and are stored as an empty string instead so later filters can skip them cheaply.
2. **Searching the cache**: once built, name/trait/equipment-type/armor-weight filters are applied to every cached name to find candidate IDs, resolving traits/types/weights on demand via the real ESO APIs (`GetItemLinkTraitInfo`, `GetItemLinkEquipType`, `GetItemLinkWeaponType`, `GetItemLinkArmorType`).

Because scanning 184,000 IDs synchronously would freeze the game, **both the cache-building pass and the search pass are chunked across several game-engine frames** using `EVENT_MANAGER:RegisterForUpdate(...)`, which repeatedly calls a worker function roughly every 100ms until the whole ID range has been processed, printing progress messages as it goes.

---

## Code Walkthrough

### 1. Manifest (`FindMyStuff.addon`)

```
## Title: |cFF0000Find|r |cffff00My|r |c00ffffStuff!|r
## Author: |cff00ffVixen Hunny|r
## Description: Keeping track of all your stuff from other characters!
## APIVersion: 101049 101050
## Version: 1.0.0
## SavedVariables: FindMyStuffSV
## DependsOn: LibAddonMenu-2.0

FindMyStuff.lua
```

This is the standard ESO AddOn manifest format. Key lines:
- `## APIVersion` lists the client API versions this AddOn claims compatibility with (ESO bumps this number with most updates; an AddOn Manager will warn/disable the AddOn if the running game's API isn't listed here).
- `## SavedVariables: FindMyStuffSV` tells the client to persist the global Lua table named `FindMyStuffSV` to disk between sessions — this is what `ZO_SavedVars` wraps in code (see [§17](#17-findmystuffinitialize-and-addon-bootstrapping)).
- `## DependsOn: LibAddonMenu-2.0` declares a library dependency, though the current code doesn't appear to build an LAM settings panel — likely reserved for a future options UI.
- The final line lists which Lua file(s) to load — just `FindMyStuff.lua`.

### 2. Module Bootstrap & State

```lua
FindMyStuff = FindMyStuff or {}
FindMyStuff.Name = "FindMyStuff"
FindMyStuff.Author = "Vixen Hunny"
FindMyStuff.Version = "1.0.0"
FindMyStuff.SettingsVersion = "1.0"
local pmax = 184000
local gmin = 1
local gmax = pmax
local total_results = 0
local max_results = 100
local current_cycle = 1
local cycles = 5
local cycle_size = 0
local base_suffix = ":0:0:0:0:0:0:0:0:0:0:0:0:0:0:0:0:10000:0|h|h"
local link_suffix = ""
local dump_output = {}
local dump_position = 1
```

- `FindMyStuff = FindMyStuff or {}` creates (or reuses) a single global namespace table — the standard ESO AddOn idiom that avoids clobbering the table if the file is somehow re-executed.
- `pmax = 184000` is the upper bound of the brute-force item ID scan — effectively "the largest item ID the author has observed existing in the game".
- `max_results = 100` caps how many matches a single search pass will report before stopping early (to avoid spamming chat).
- `cycles = 5` controls how many `EVENT_MANAGER` update ticks a *search* pass is divided into (see [§6](#6-bridge--preparing-a-database-search)); the table-build pass uses its own local `bcycles = 3` (see [§10](#10-tablebuilder--findmystuffbuildtable--caching-item-names)).
- `base_suffix` is the tail of a synthetic ESO item link: `quality:level` is prepended to this (as `link_suffix`) to form a complete, parseable item link like `|H1:item:61225:364:50:0:0:...|h|h`. The zeros represent enchantment/style/crafted-ability/etc. sub-fields that are irrelevant for lookups; `10000` is a durability/charge placeholder.
- `dump_output` / `dump_position` are scratch state used by the recipe/furnishing "dump" feature ([§12](#12-the-dump-system-dump-dumpbridge-findmystuffdumpbuilder)).

```lua
local function FMS_BuildTable(list)
	local t = {}
	for i = 1, #list do
		if list[i][1] ~= nil then
			t[list[i][1]] = list[i][2]
		end
	end
	return t
end
```

A small helper that converts an array of `{ESO_CONSTANT, "display string"}` pairs into a lookup table keyed by the constant's **value**. It is "nil-safe": if `ESO_CONSTANT` doesn't exist in the current game API version (Bethesda sometimes renames or removes global constants between updates), `list[i][1]` is `nil` and that entry is simply skipped rather than crashing with "attempted to index a nil value" when used as a table key.

### 3. Lookup Tables (Traits, Equip Slots, Weapons, Armor)

```lua
FindMyStuff.TraitTable = FMS_BuildTable({ ... })   -- maps ITEM_TRAIT_TYPE_* -> "Divines", "Nirnhoned", etc.
FindMyStuff.EquipTable  = FMS_BuildTable({ ... })   -- maps EQUIP_TYPE_* -> "head", "chest", "weapon", etc.
FindMyStuff.WeaponTable = FMS_BuildTable({ ... })   -- maps WEAPONTYPE_* -> "sword", "bow", "firestaff", etc.
FindMyStuff.ArmorTable  = FMS_BuildTable({ ... })   -- maps ARMORTYPE_* -> "light", "medium", "heavy"
```

These four tables translate the raw numeric enums returned by ESO's item-link introspection APIs into human-readable strings used both for filtering searches and for labeling results (e.g., appending `" Nirnhoned"` to a found item in search output).

Two manual patches follow the `EquipTable` build:
```lua
FindMyStuff.EquipTable[16] = "robe"   -- Handle robe/shirts
FindMyStuff.EquipTable[17] = "shirt"
```
ESO doesn't expose separate `EQUIP_TYPE_*` constants that distinguish robes/shirts from regular chestpieces at the API level (all three normally map to `EQUIP_TYPE_CHEST`), so the code reserves two synthetic index numbers (`16`/`17`, chosen because they don't collide with real equip-type IDs) purely as internal sentinel values used by [`UseName()`](#8-findmystuffusename--the-search-loop) to special-case the "is this chest item actually a robe/shirt?" check (by testing for the substring `"robe"` in the item's name).

### 4. Equipment Synonym Table

```lua
FindMyStuff.EquipSynonymTable = {
	[1]= {[1]="hat",[2]=EQUIP_TYPE_HEAD,[3]="light"},
	...
	[22]= {[1]="gauntlet",[2]=EQUIP_TYPE_HAND,[3]="heavy"}
}
```

This table lets players type intuitive armor-piece names (e.g. `helm`, `cuirass`, `pauldrons`, `sabatons`) instead of having to know generic slot names like "head" or "chest", and **also implicitly supplies the armor weight** for that term (a "cuirass" is always heavy, a "jack" is always medium, etc.). Each entry is a 3-tuple: `{synonym, EQUIP_TYPE_* constant or the 16/17 sentinel, default weight}`. Per the comment above the table, each synonym has had its trailing `s` stripped ("one `s` removed from each for optimum matching ease") so the lookup code only has to strip a trailing `s` off the user's input once (see `bridge()`) rather than handle every singular/plural variant.

### 5. The Item-Link Suffix & Default Quality/Level

```lua
FindMyStuff.ItemTable = {built=0,items={},}
FindMyStuff.Default = {quality=364,level=50}
```

`FindMyStuff.ItemTable` is the in-memory name cache: `built` is a flag (`0`/`1`) indicating whether the full ID→name cache has been populated yet this session, and `items` is the sparse array itself (`items[id] = "lowercased item name"` or `""` for invalid IDs).

`FindMyStuff.Default` seeds the quality/level values used to build synthetic item links for lookups before the user has ever run `/fmssetquality`. `364` happens to correspond to a specific style/material quality byte that the author found reliably resolves to a valid, inspectable item link for most items at level 50 — but it can be overridden per-character (see [§14](#14-setquality--calibrating-searches-to-a-real-item)).

### 6. `bridge()` — Preparing a Database Search

```lua
local function bridge(text,min,max,trait,etype,weight)
	if min == nil then gmin = 1 else gmin = min end
	if max == nil then gmax = pmax else gmax = max end
	trait = trait:lower()
	...
```

`bridge()` is the entry point that turns parsed user input (search text, optional min/max ID range, trait, equipment type, armor weight) into a running, chunked search:

1. **Normalizes the ID range** — defaults to the full `1..pmax` range if the user didn't supply explicit bounds, otherwise uses their custom `min`/`max`.
2. **Applies two hard-coded "known dead zone" optimizations** that skip ranges of IDs the author has empirically determined never contain matches for certain filters, to speed up full scans:
   - If searching for equipment type `"poison"`, jump straight to ID `75000` (poisons don't begin until then).
   - If searching for a Summerset-era jewelry trait (`bloodthirsty`, `harmony`, `infused`, `protective`, `swift`, `triune`), jump to ID `139000`.
3. **Resets search counters** (`total_results`, `current_cycle`) and computes `cycle_size = ceil((gmax-gmin)/cycles)` — i.e., splits the ID range into `cycles` (5) roughly-equal chunks so the search can be spread across multiple update ticks instead of blocking the game thread for the entire range at once. `tmin`/`tmax` (both *global*, not `local` — note the lack of a `local` keyword, meaning these are implicit Lua globals shared with the table-builder and dump systems) are set to the first chunk's bounds.
4. **Prints a human-readable "Beginning search for..." summary line** describing exactly what filters are active.
5. **Resolves equipment synonyms**: if an `etype` string was supplied, it strips a trailing `s`, then looks it up in `EquipSynonymTable` for an exact match; if found, replaces `etype` with the table's canonical display string and — if the user didn't separately specify a `weight` — infers the weight from the synonym (e.g. typing `type:cuirass` automatically implies `weight:heavy`).
6. **Lower-cases** the final `etype`/`weight` strings for consistent matching.
7. Finally, registers a repeating update callback: `EVENT_MANAGER:RegisterForUpdate(FindMyStuff.Name, 100, function() FindMyStuff.UseName(text,trait,etype,weight) end)` — this is what actually drives the chunked search, firing roughly every 100ms until it unregisters itself.

### 7. `split_str()` — Tokenizing Search Text

```lua
local function split_str(inputstr, sep)
	if sep == nil then sep = "%s" end
	local t={}
	local i=1
	for str in string.gmatch(inputstr, "([^"..sep.."]+)") do
		t[i] = str
		i = i + 1
	end
	return t
end
```

A generic whitespace (or custom-separator) string splitter built on Lua pattern matching, used both to break a search phrase into individual word-tokens (`UseName`) and to split a colon-delimited item link into its component fields (`GradeItem`, `SetQuality`).

### 8. `FindMyStuff.UseName()` — The Search Loop

This is the core matching function, invoked repeatedly by the `EVENT_MANAGER` callback set up in `bridge()`. Each invocation processes only the current chunk, `tmin` through `tmax`:

```lua
function FindMyStuff.UseName(text,trait,etype,weight)
	local searchstring = text:lower():gsub("-","--")
	...
	local searcharray = split_str(searchstring)
	local terms = #searcharray
	...
	for i = tmin, tmax, 1 do
		...
```

Step by step, for every candidate ID `i` in the current chunk:
1. **Skip cache misses** — if `FindMyStuff.ItemTable.items[i]` is `nil` or `""`, the cached name-build pass already determined this ID doesn't resolve to a real item, so it's skipped instantly without calling any ESO API (this is the whole point of pre-building the cache: subsequent searches are cheap substring lookups instead of expensive API calls).
2. **Name matching** — checks that the first search term appears as a substring of the cached (lowercased) item name; if there are additional terms, each must *also* appear somewhere in the name (an implicit AND across all whitespace-separated words, not a phrase match).
3. **Trait filtering** — if a `trait:` filter was given, it resolves the real trait via `GetItemLinkTraitInfo(...)` (looked up through `FindMyStuff.TraitTable`) and requires it to (fuzzy-)match.
4. **Equipment-type / weapon-type filtering** — if `type:` was given, resolves the item's `EquipType` via `GetItemLinkEquipType`, with special-case logic to disambiguate robes/shirts/plain-chest items by checking whether the cached name contains `"robe"`. If the equip type is itself "weapon", it further resolves the specific `WeaponType` via `GetItemLinkWeaponType` to match against keywords like `sword`/`bow`/`firestaff`.
5. **Armor-weight filtering** — if `weight:` was given, resolves `GetItemLinkArmorType` and requires a match.
6. **Recording a hit** — matched items are appended to `out_ary` as a formatted string like `ID #61225: |H1:item:61225:364:50:...|h|h Nirnhoned`, and `total_results` is incremented. Once `total_results` reaches `max_results` (100), the inner loop breaks immediately (remembering the breaking `item_id` so the next search can resume exactly where it left off).

After the per-chunk loop, three possible outcomes are handled:
- **Still under the cap, more range left** (`total_results < max_results and tmax < gmax`): print what was found this chunk and let the update event fire again for the next chunk.
- **Still under the cap, scan complete** (`tmax >= gmax`): print the final results plus a "Finished search. Total results: N" summary.
- **Hit the 100-result cap**: print what was found, then **build a follow-up command string** (e.g. `/findid night mother trait:divines type:ring 142001`) that would resume the search starting right after the last match, and — if not on console UI — **auto-populate the chat edit box** with that command via `CHAT_SYSTEM.textEntry:Open(...)` so the player can just hit Enter to continue.

Finally, regardless of outcome, the function advances `current_cycle` and recomputes the next chunk's `tmin`/`tmax`, or calls `EVENT_MANAGER:UnregisterForUpdate(FindMyStuff.Name)` to stop once `tmax >= gmax`.

> There's a small debug leftover in this function: `if i > 176000 and (temp_trait == nil or link_suffix == nil) then d(i) end` — this prints the raw ID to chat if an edge case near the top of the ID range produces a `nil` trait/suffix, apparently left in from debugging.

### 9. Dead/Legacy Helpers: `ShowItem`, `BreakItem`

```lua
local function ShowItem(text)        -- resolves an ID (or raw item-link fragment) to a clickable link
local function BreakItem(text)       -- converts an item link back into plain, editable text
```

`ShowItem(text)` takes either a bare numeric item ID (prints `ID #<id>: |H1:item:<id><suffix>|h|h`) or, if given non-numeric text, assumes it's already an item-link fragment and just prepends a chat-link pipe (`|`). `BreakItem(text)` does the reverse: it takes a real item link, neutralizes the opening `|H` escape sequence so it won't render as a clickable link, and pushes `/showitem <neutralized text>` into the chat edit box for the user to copy/edit — intended as a round-trip "inspect and tweak an item link" workflow. **Neither function is currently wired to a slash command** (see [Known Quirks & Observations](#known-quirks--observations)); they appear to be retained from a previous command set described in `IFHelp()`.

### 10. `TableBuilder()` / `FindMyStuff.BuildTable()` — Caching Item Names

```lua
local function TableBuilder(text,mode)
	current_cycle = 1
	bcycles = 3
	cycle_size = math.ceil((gmax-gmin)/bcycles)
	tmin = gmin
	tmax = gmin+cycle_size
	d("Building current item table...")
	EVENT_MANAGER:RegisterForUpdate(FindMyStuff.Name, 100, function() FindMyStuff.BuildTable(text,mode) end)
end
```

Before *any* name-based database search can run, the full `1..pmax` ID range has to be probed once per session to populate `FindMyStuff.ItemTable.items`. `TableBuilder` kicks this off, again split into chunks (3 chunks this time, via the implicit-global `bcycles`) processed by repeated `EVENT_MANAGER` ticks.

```lua
function FindMyStuff.BuildTable(text,mode)
	if text == '' then text = nil end
	if mode == nil then mode = 0 end
	for i = tmin, tmax, 1 do
		local link = "|H1:item:" .. i .. link_suffix
		local snaglink = GetItemLinkName(link)
		if snaglink ~= "h" then
			FindMyStuff.ItemTable['items'][i] = tostring(snaglink):lower()
		else
			FindMyStuff.ItemTable['items'][i] = ""
		end
	end
	if tmax >= gmax then
		FindMyStuff.ItemTable.built = 1
		d("Item table built.")
		EVENT_MANAGER:UnregisterForUpdate(FindMyStuff.Name)
		if text ~= nil and mode == 0 then
			RangeCheck(text)
		elseif mode == 1 then
			DumpBridge(text)
		end
	else
		d("Building...")
		current_cycle = current_cycle+1
		tmin = gmin+(cycle_size*(current_cycle-1))+1
		tmax = gmin+cycle_size*current_cycle
	end
end
```

For every ID in the current chunk it asks ESO for the item's display name via a synthetic link. ESO returns the literal string `"h"` for IDs that don't correspond to any real item (an artifact of how an unresolved link renders), so that's used as the invalid-ID sentinel and normalized to an empty string for cheap `~= ""` checks later. Once the whole range has been walked (`tmax >= gmax`), it marks the cache `built = 1`, unregisters the update, and **chains into whichever caller originally needed the cache** — either resuming a name search (`RangeCheck(text)`, `mode == 0`) or a recipe/furnishing dump (`DumpBridge(text)`, `mode == 1`).

### 11. `RangeCheck()` — Parsing Flags Out of User Input

```lua
local function RangeCheck(text)
	if FindMyStuff.ItemTable.built == 0 then
		TableBuilder(text)
	else
		local trait = ""
		if text:match("%s-trait:%w+") ~= nil then ... end
		local etype = ""
		if text:match("%s-type:%w+") ~= nil then ... end
		local weight = ""
		if text:match("%s-weight:%w+") ~= nil then ... end
		...
```

This is effectively the real command parser behind the database search feature. If the name cache hasn't been built yet this session, it first triggers `TableBuilder` (which will call back into `RangeCheck` once the cache is ready — see [§10](#10-tablebuilder--findmystuffbuildtable--caching-item-names)). Otherwise it:

1. **Strips out `trait:xxx`, `type:xxx`, and `weight:xxx` flags** from anywhere in the input string using Lua patterns, capturing their values and removing them from the remaining free-text search phrase.
2. **Detects a leading numeric range** — after stripping flags, if the remaining text is purely numeric/whitespace, it treats any remaining number(s) as an explicit `min` (and optional `max`) item-ID bound rather than as search terms (this is what lets the auto-continuation command from `UseName()`, e.g. `/findid night mother 142001`, resume a capped search starting at a specific ID).
3. Hands everything off to `bridge(text, min, max, trait, etype, weight)` to actually start (or continue) the chunked search.

### 12. The Dump System (`Dump`, `DumpBridge`, `FindMyStuff.DumpBuilder`)

```lua
local function Dump(text)
	if text == 'furniture' or text == 'provisioning' or text == 'reagent' then
		if FindMyStuff.ItemTable.built == 0 then
			TableBuilder(text,1)
		else
			DumpBridge(text)
		end
	else
		d('Current dump options: "furniture" "provisioning" "reagent"')
	end
end
```

Bound to `/fmsdump`. This feature scans the *entire* item database (again, building the name cache first if needed) and extracts every item ID whose specialized item type matches one of three categories:

- **`furniture`** — any crafting-station blueprint/pattern/diagram/schematic/sketch/formula that produces a **furnishing** (blacksmithing, clothier, enchanting, alchemy, provisioning, woodworking, jewelrycrafting).
- **`provisioning`** — standard food or drink recipes.
- **`reagent`** — alchemy reagents (animal parts, fungus, herbs).

```lua
function FindMyStuff.DumpBuilder(text)
	...
	for i = tmin, tmax, 1 do
		if FindMyStuff.ItemTable['items'][i] ~= "" then
			if i ~= 55462 then   -- Only ignore Roast Pig BOP
				local itemLink = "|H1:item:" .. i .. link_suffix
				local _,sType = GetItemLinkItemType(itemLink)
				if mode == 0 then ... elseif mode == 1 then ... elseif mode == 2 then ... end
				if #dump_output >= 100 then
					FindMyStuff.SavedVariables[text .. dump_position] = implode(",",dump_output)
					dump_position = dump_position+1
					TableReset(dump_output)
				end
			end
		end
	end
	...
```

Like the other scans, this is chunked over several update ticks. Matching IDs accumulate in the `dump_output` array; whenever it reaches 100 entries it's flattened into a comma-separated string (via the custom `implode()` helper — Lua has no built-in `table.concat`-with-this-exact-semantic... actually `table.concat` exists, but the author wrote their own) and saved into `FindMyStuffSV[text .. dump_position]` (e.g. `FindMyStuffSV.furniture1`, `FindMyStuffSV.furniture2`, ...) so that a human can review the exported ID lists directly in the SavedVariables file after a `/reloadui`, since chat output would be too long/volatile to copy reliably. One hard-coded exception skips item ID `55462` ("Roast Pig" bind-on-pickup) which apparently false-matches the provisioning category.

### 13. `GradeItem()` / `AssembleItem()` — Quality Tier Guessing

```lua
local function AssembleItem(item_data,quality)
	if quality ~= nil then item_data[4] = quality end
	local out = ""
	if tonumber(item_data[4]) < 0 then item_data[4] = 0 end
	for i = 1, #item_data, 1 do out = out .. ":" .. item_data[i] end
	return out:sub(2)
end
```
Rebuilds a colon-delimited item-link string from its component fields, optionally overriding the quality field (index 4) — used to probe "what does this same item ID look like at a different quality byte?"

```lua
local function GradeItem(text)
	local item_data = split_str(text,":")
	...
	local quality = tonumber(item_data[4])
	local quality_tier = GetItemLinkQuality(text)
	...
```

Given a full item link, `GradeItem` tries to reverse-engineer the raw "quality" byte values that correspond to each of the 5 named quality tiers (Normal/Fine/Superior/Epic/Legendary) for *that specific item*, since ESO's internal quality byte isn't a simple linear 1–5 scale — it varies by item level bracket (there are documented CP10–100 "batches" with different offsets, handled by two large hard-coded lookup tables of byte-offsets in the `39-120` and `125-174` ranges) and falls back to a **brute-force probing loop** for items outside those known offset patterns: it repeatedly tweaks the quality byte up/down and calls `GetItemLinkQuality()` to empirically discover which raw byte produces each of the five quality tiers, capping attempts at small retry limits (`quality_attempts < 7`) to avoid infinite loops. Results are printed as 5 item links, one per quality tier, letting a player preview/link an item at every quality level even if they've only seen one version of it. **This function is not currently bound to any slash command** (see [§9](#9-deadlegacy-helpers-showitem-breakitem) and [Known Quirks & Observations](#known-quirks--observations)).

### 14. `SetQuality()` — Calibrating Searches to a Real Item

```lua
local function SetQuality(text)
	local item_data = split_str(text,":")
	if tonumber(item_data[4]) ~= nil and tonumber(item_data[5]) ~= nil then
		FindMyStuff.SavedVariables.quality = item_data[4]
		FindMyStuff.SavedVariables.level = item_data[5]
		FindMyStuff.Default = {quality=FindMyStuff.SavedVariables.quality,level=FindMyStuff.SavedVariables.level}
		link_suffix = ":" .. FindMyStuff.Default.quality .. ":" .. FindMyStuff.Default.level .. base_suffix
		d("Item Finder quality updated.")
	else
		d("Item Finder refuses your offering. Try an item with a level.")
	end
end
```

Bound to `/fmssetquality <item_link>`. Splits a pasted item link on `:` and reads out fields 4 and 5 (quality byte and item level) directly from it, then persists them to `FindMyStuffSV` and rebuilds the module-level `link_suffix` used by every synthetic item-link lookup from then on. This matters because some items only resolve a valid name/trait/etc. at specific quality/level combinations — giving the user control over which "template" item link is used for all subsequent database probes.

### 15. `IFHelp()` — In-Game Help Text

A straightforward block of `d(...)` (ESO's debug/chat print function) calls that prints colorized command usage help, including command syntax, examples, and the available `trait:`/`type:`/`weight:` filter keywords (enumerating every weapon type, armor slot, and the weight-specific synonym vocabulary from `EquipSynonymTable`). Bound to `/fmshelp`.

### 16. Inventory Tracking System

```lua
-- Quality color codes indexed by ESO ItemQuality (0=Trash .. 5=Legendary)
local FMS_QUALITY_COLORS = {
	[ITEM_DISPLAY_QUALITY_TRASH] = "|c9D9D9D",
	[ITEM_DISPLAY_QUALITY_NORMAL] = "|cFFFFFF",
	[ITEM_DISPLAY_QUALITY_MAGIC] = "|c2DC50E",
	[ITEM_DISPLAY_QUALITY_ARCANE] = "|c4169E1",
	[ITEM_DISPLAY_QUALITY_ARTIFACT] = "|cA02EF7",
	[ITEM_DISPLAY_QUALITY_LEGENDARY] = "|cffff00",
	[ITEM_DISPLAY_QUALITY_MYTHIC_OVERRIDE] = "|cFFA500",
}
```
A simple map from ESO's built-in item-quality enum to ESO chat color codes, used purely for cosmetic output formatting in `/finditem` results.

```lua
local function FMS_ScanBag(bag, inv)
	local numSlots = GetBagSize(bag)
	for slot = 0, numSlots - 1 do
		local itemLink = GetItemLink(bag, slot)
		if itemLink ~= "" then
			...
			local key = itemName .. (traitName and ("\t" .. traitName) or "")
			local existing = inv[key]
			if existing and type(existing) == "table" then
				existing.n = existing.n + stack
			else
				inv[key] = {n = stack, q = quality, t = traitName, name = itemName, bound = boundStr, known = knownStr}
			end
		end
	end
end
```

For a given bag ID, this walks every occupied slot and records, per distinct item:
- **`n`** — running total stack count (so multiple stacks/slots of the same item+trait combo are merged into one line).
- **`q`** — display quality (for color-coding).
- **`t`** — trait name (resolved via the shared `TraitTable`; normalized to `nil` if "None").
- **`bound`** — whether the item is `"Unbound"`, `"BoE"` (Bind on Equip), or `"Bound"`, via `GetItemLinkBindType`.
- **`known`** — for recipes, crafted-ability scripts, racial motifs, and collectibles, whether that recipe/motif/collectible has already been learned/unlocked account-wide (`IsItemLinkRecipeKnown` / `IsItemLinkBookKnown`), `nil` for everything else.

The composite key `itemName .. "\t" .. traitName` deliberately keeps differently-trait-rolled copies of "the same" item (e.g., two rings with the same name but different traits) as **separate inventory entries** rather than incorrectly merging their stack counts.

```lua
function FindMyStuff.ScanInventory()
	if not FindMyStuff.SavedVariables then return end
	if not FindMyStuff.SavedVariables.charInventory then
		FindMyStuff.SavedVariables.charInventory = {}
	end
	local charName = GetUnitName("player")
	local charInv = {}
	FMS_ScanBag(BAG_BACKPACK, charInv)
	FMS_ScanBag(BAG_WORN, charInv)
	FindMyStuff.SavedVariables.charInventory[charName] = charInv
	local sharedInv = {}
	FMS_ScanBag(BAG_BANK, sharedInv)
	if GetBagSize(BAG_SUBSCRIBER_BANK) > 0 then FMS_ScanBag(BAG_SUBSCRIBER_BANK, sharedInv) end
	if GetBagSize(BAG_CRAFT_BAG) > 0 then FMS_ScanBag(BAG_CRAFT_BAG, sharedInv) end
	if next(sharedInv) ~= nil then
		FindMyStuff.SavedVariables.charInventory["[Bank/Craft Bag]"] = sharedInv
	end
end
```

The public scan entry point (bound to `/fmsscan`, and auto-triggered on login/loot — see [§17](#17-findmystuffinitialize-and-addon-bootstrapping)). It completely **replaces** (rather than merges) the stored inventory snapshot for the current character and for the shared `"[Bank/Craft Bag]"` pseudo-character each time it runs, so the saved data always reflects a fresh, accurate point-in-time snapshot rather than accumulating stale entries for items that have since been moved/sold/consumed. Bank/subscriber-bank/craft-bag contents are account-wide and shared by every character, so they're intentionally stored once under a synthetic key instead of being duplicated per character.

```lua
local function FindInInventory(searchText)
	...
	local search = searchText:lower()
	local results = {}
	for charName, items in pairs(FindMyStuff.SavedVariables.charInventory) do
		for key, entry in pairs(items) do
			if type(entry) == "number" then
				entry = {n = entry, q = 1, t = nil, name = key}
			end
			local displayName = entry.name or key
			if displayName:lower():find(search, 1, true) then
				results[#results + 1] = {char = charName, name = displayName, entry = entry}
			end
		end
	end
	...
	table.sort(results, function(a, b) return a.char < b.char end)
	...
```

Bound to `/finditem <name>`. Performs a plain (non-pattern, `find(..., 1, true)` is a **literal** substring search, so special characters in item names won't break matching) case-insensitive substring search across every character's stored inventory, including a **legacy-format compatibility shim**: if a stored entry is a bare number (an older saved-variable schema that only tracked counts, before trait/quality/bind/known tracking was added), it's upgraded on the fly into the richer table shape so old saved data doesn't break the newer display logic. Results are sorted alphabetically by character name and printed with full color/trait/bind/known annotations.

### 17. `FindMyStuff:Initialize()` and AddOn Bootstrapping

```lua
function FindMyStuff:Initialize()
	FindMyStuff.SavedVariables = ZO_SavedVars:NewAccountWide("FindMyStuffSV", FindMyStuff.SettingsVersion, nil, FindMyStuff.Default)
	EVENT_MANAGER:UnregisterForEvent(FindMyStuff.Name, EVENT_ADD_ON_LOADED)
	FindMyStuff.Default = {quality=FindMyStuff.SavedVariables.quality,level=FindMyStuff.SavedVariables.level}
	link_suffix = ":" .. FindMyStuff.Default.quality .. ":" .. FindMyStuff.Default.level .. base_suffix
	d("FindMyStuff v" .. FindMyStuff.Version .. " loaded. Type /fmshelp to begin searching for items.")
	SLASH_COMMANDS["/finditem"] = FindInInventory
	SLASH_COMMANDS["/fmsscan"] = function() FindMyStuff.ScanInventory() end
	SLASH_COMMANDS["/fmsdump"] = Dump
	SLASH_COMMANDS["/fmssetquality"] = SetQuality
	SLASH_COMMANDS["/fmshelp"] = IFHelp
	EVENT_MANAGER:RegisterForEvent(FindMyStuff.Name .. "_Inv", EVENT_PLAYER_ACTIVATED, function()
		zo_callLater(FindMyStuff.ScanInventory, 3000)
	end)
	EVENT_MANAGER:RegisterForEvent(FindMyStuff.Name, EVENT_LOOT_UPDATED, function(eventCode)
		zo_callLater(function() FindMyStuff.ScanInventory() end, 3000)
	end)
end

function FindMyStuff:OnAddOnLoaded(event, addonName)
	FindMyStuff:Initialize()
end

EVENT_MANAGER:RegisterForEvent(FindMyStuff.Name, EVENT_ADD_ON_LOADED, function(event, addonName)
	if addonName == FindMyStuff.Name then
		FindMyStuff:OnAddOnLoaded(event, addonName)
	end
end)
```

This is the file's entry point:

1. The very last statement in the file registers a one-time listener for `EVENT_ADD_ON_LOADED`, which ESO fires once for *every* AddOn as it loads — the handler filters for the event matching `FindMyStuff`'s own name (`addonName == FindMyStuff.Name`) so it only reacts to its own load event, then calls `FindMyStuff:OnAddOnLoaded`, which simply forwards to `FindMyStuff:Initialize()`.
2. `Initialize()` creates/loads the persistent saved-variables table via `ZO_SavedVars:NewAccountWide(...)` — "AccountWide" (rather than per-character) is what makes the item-database quality/level defaults and `charInventory` table shared and visible across all characters on the account, which is essential for the cross-character search feature to work at all.
3. It immediately unregisters the `EVENT_ADD_ON_LOADED` listener (no longer needed once initialized).
4. It re-derives `FindMyStuff.Default` and `link_suffix` from whatever was actually loaded from disk (falling back to the hard-coded `{quality=364, level=50}` default from [§5](#5-the-item-link-suffix--default-qualitylevel) the very first time the AddOn ever runs for an account).
5. It prints a load confirmation message and registers all five slash commands.
6. It registers two auto-scan triggers: `EVENT_PLAYER_ACTIVATED` (fires once the player finishes zoning in / finishes loading a character) and `EVENT_LOOT_UPDATED` (fires whenever the player's loot/inventory changes) — both delayed 3 seconds via `zo_callLater` to let the game's bag data fully populate/settle before scanning, since scanning too early after a zone-in can read incomplete bag contents.

---

## Saved Variables Structure

Persisted to `FindMyStuffSV` (viewable in `...\live\SavedVariables\FindMyStuff.lua` after a `/reloadui` or logout):

```lua
FindMyStuffSV = {
	["Default"] = {
		["$AccountWide"] = {
			quality = 364,              -- last quality byte set via /fmssetquality
			level = 50,                 -- last item level set via /fmssetquality
			charInventory = {
				["MyMainCharacter"] = {
					["Item Name\tTraitName"] = { n = 3, q = 5, t = "Nirnhoned", name = "Item Name", bound = "Bound", known = nil },
					...
				},
				["[Bank/Craft Bag]"] = { ... },  -- shared account-wide storage
			},
			["furniture1"] = "1234,5678,...",   -- from /fmsdump furniture (chunks of up to 100 IDs)
			["provisioning1"] = "...",
			["reagent1"] = "...",
		},
	},
}
```

## Known Quirks & Observations

- **Help/command mismatch**: `IFHelp()` documents commands (`/findindb`, `/findid`, `/findbyname`, `/showitem`, `/findname`, `/findbyid`, `/breakitem`, `/gradeitem`) that are not registered in `SLASH_COMMANDS` anywhere in the current file. Only `/finditem`, `/fmsscan`, `/fmsdump`, `/fmssetquality`, and `/fmshelp` are live. The full item-database **name search** engine (`bridge`/`UseName`/`RangeCheck`/`TableBuilder`) is fully implemented but currently has **no slash command wired to trigger it** — likely an oversight from a rename/refactor (older command names probably started with `/find...` or `/if...` to match the old `IFHelp` branding) or a work-in-progress removal while the newer inventory-tracking feature was being built out.
- **Implicit globals**: several loop-control variables (`tmin`, `tmax`, `bcycles`) are assigned without `local`, making them true Lua globals shared across the search, table-build, and dump subsystems. This is relied upon intentionally (so a chunked operation can resume state across update ticks called from different functions), but it also means these subsystems cannot run concurrently without interfering with each other.
- **Debug leftovers**: a stray `d(i)` print statement exists in `UseName()` for a specific edge case near item ID 176000+.
- **Hard-coded magic numbers**: `pmax = 184000`, the `75000`/`139000`/`55462` special-cased IDs, and the CP10–100 quality-byte offset tables in `GradeItem` are all empirically derived from observing the live game and will need periodic updates as new game content raises the maximum valid item ID or introduces new quality-byte ranges.
- **Account-wide only**: because `ZO_SavedVars:NewAccountWide` is used (not per-character saved vars), quality/level calibration and the item-database caches/dumps are shared account-wide rather than per-character — intentional, since the whole point of the AddOn is cross-character visibility.

## Glossary of ESO Concepts Used

| Term | Meaning |
|---|---|
| **Item Link** | A specially-formatted clickable chat string like `\|H1:item:<id>:<quality>:<level>:...\|h<name>\|h` that the client can resolve into an item's name/stats/tooltip. |
| **`d(...)`** | ESO's built-in debug-print function; writes a line to the chat window. |
| **`EVENT_MANAGER:RegisterForUpdate`** | Schedules a callback to run repeatedly (here, every 100ms) — used to spread expensive loops across multiple frames instead of freezing the UI thread. |
| **`SLASH_COMMANDS`** | A global table where AddOns register `/command` → handler-function mappings. |
| **`ZO_SavedVars`** | ESO's standard wrapper for reading/writing an AddOn's persisted `SavedVariables` Lua table, with per-character vs. account-wide and settings-version-migration support. |
| **Specialized Item Type** | A finer-grained item classification than the basic item type (e.g., distinguishing a Blacksmithing furnishing diagram from a Clothier furnishing pattern, both of which are otherwise `ITEMTYPE_RECIPE`). |
