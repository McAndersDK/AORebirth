# NPC events per spawn point (NpcEvents.json)

Status: proposal (2026-10-19), for discussion. Nothing implemented.

## The problem

What an NPC does after it spawns - how it moves, what it plays, says or poses - belongs to the **spawn point** in live,
not to the creature. GameData puts it on the template today:

- `NpcTemplates.json` `Patrol` goes to every NPC of the hash (`SpawnService.cs:240`, `NpcBrain.EnablePatrol` keeping
  waypoints within 60 m of home). On Arete Landing (6553) all 24 Cleaning Robot points are hash JUCO; live has two of
  them walking a loop and the rest standing, GameData has all of them near the loop walking it. The Protesters (ROTE,
  5 points) walk two separate loops from two points; the other three stand.
- `NpcTemplates.json` `Events` and `SpawnState` (`NpcEvents.cs`: the Wounded Dockworker, the Omni-AF Private, the
  robots' flames and sparks) are per hash too, so a one-off scene at one spot plays at every spot of the hash.

`Playfields/<pf>/Spawns.json` can't carry it: it is extracted from the RDB and rewritten on every extraction, and its
points have no identity but their place in the list.

## What the RDB tells us

Bit 0x20 of a spawn point's `nativeFlags` marks the points whose creature walks a route. Against every patrol
SessionDump recorded after it stopped counting fight movement (aosharp `add22b0`), counting only the points of the
hashes that spawn that creature: 21 of 21 points on a captured graph walk and 10 of 11 on a captured loop have it,
32 of 236 points off a route (likely walkers whose walk wasn't captured), 10 % of all points. The route itself is not
in the RDB. So the flag is a check on the data, not a source of it.

Nothing in the RDB marks a point that wanders: the Malfunctioning Cleaning Robots' points look like the standing
Cleaning Robots' (no bit 0x20), and the server never wanders a hash point today. A hash point's `Wander`, like its route, comes from what was seen in live, not from the RDB.

## Design

1. **Spawn point ids** from the extractor, stable across extractions.
2. **`NpcEvents.json`**: entries that give NPCs movement, a spawn state and events, with two targets: a template's
   **hash** (every NPC of that template, wherever it spawns; `GameData/NpcEvents.json`) or a **spawn point** (the NPC
   that comes out of that one point; `PlayfieldContent/<pf>/NpcEvents.json`).
3. **Merge**: an NPC gets its hash's entry with its spawn point's merged on top.
4. **Movement in the same file**: `Stationary`, `Patrol` (its own route) or `Wander`, plus a new event trigger
   `Arrived` for what it does at a waypoint.
5. `NpcTemplates.json` loses `Patrol`, `Events` and `SpawnState`; they move into the new file, per hash as `Hash`
   entries. The template keeps what the creature *is* (name, bands, stats, equipment, look, loot).

### 1. Spawn point ids

The extractor (`Tools/RDBDataExtractor`) writes an `id` on every point of `Spawns.json`:
eight characters of Crockford base32 (`0-9`, `A-Z` without `I L O U`), e.g. `PTV7MXAR`, the first 40 bits of the
SHA-256 of

```
<playfield>|<hash>|<x>|<y>|<z>|<n>       e.g.  6553|ROTE|3543.2|7.3|778.7|1   ->  PTV7MXAR
```

with the position rounded to 0.1 m (`%.1f`) and `n` counting points of the same hash at the same place in RDB order
(1 for all but 151 stacked points). The RDB has no id of its own (`version7Extra` is 0 on 97 % of points,
`manifestHash` repeats); playfield, hash and position are what a point is, so the id is the same on every extraction
of the same client data. Over all 630 extracted playfields (33,034 points) the ids are unique, globally (with 40
bits the chance of any collision is about 1 in 2,000); the extractor checks and stops on a collision rather than
writing two points with one id.

The id says nothing by itself; the point's hash and position stay next to it in `Spawns.json`.

`PlayfieldSpawnEntry` gets `Id`; the server keys points by it.

### 2. NpcEvents.json

Two files of the same entry shape, PascalCase like the rest of `PlayfieldContent`:

- `GameData/NpcEvents.json`: `Hash` entries, for every NPC of that template wherever it spawns (what the templates
  carried until now).
- `PlayfieldContent/<pf>/NpcEvents.json`, next to `Npcs.json` and `Props.json`: `SpawnPoints` entries for that
  playfield's points.

The global file:

```json
{
  "SchemaVersion": 1,
  "Entries": [
    {
      "Key": "burning-cleaning-robot-flames",
      "Hash": "GAFR",
      "Events": [
        {
          "Id": "flames",
          "On": "Spawned",
          "StartMinMs": 2200, "StartMaxMs": 36000, "RepeatMinMs": 2200, "RepeatMaxMs": 36000,
          "Steps": [ { "DelayMs": 0, "Gfx": 43121 }, { "DelayMs": 0, "Gfx": 43235 } ]
        }
      ]
    }
  ]
}
```

6553's file:

```json
{
  "SchemaVersion": 1,
  "PlayfieldId": 6553,
  "Entries": [
    {
      "Key": "protester-loop-a",
      "SpawnPoints": ["PTV7MXAR"],
      "Movement": {
        "Mode": "Patrol",
        "Patrol": {
          "Waypoints": [[3543.2, 7.3, 778.7], [3549.9, 6.9, 778.7], [3559.2, 6.9, 779.1], [3566.8, 6.9, 776.8],
                        [3578.7, 6.9, 769.6]],
          "Forks": { "4": [5, 9], "11": [4, 9, 12] },
          "Run": false,
          "MinPauseSeconds": 0.5,
          "MaxPauseSeconds": 2.0
        }
      },
      "Provenance": "capture:Anders 2026-10-18"
    },
    {
      "Key": "protester-loop-b",
      "SpawnPoints": ["5X7YERRS"],
      "Movement": {
        "Mode": "Patrol",
        "Patrol": {
          "Waypoints": [[3509.1, 5.5, 793.8], [3512.4, 5.1, 794.3], [3517.3, 5.1, 797.3], [3526.3, 5.1, 798.4],
                        [3533.2, 5.1, 801.8]],
          "Forks": { "9": [10, 5, 14], "13": [10, 5, 14], "17": [0, 4, 5] },
          "Run": false,
          "MinPauseSeconds": 0.5,
          "MaxPauseSeconds": 2.0
        }
      },
      "Provenance": "capture:Anders 2026-10-18",
      "Events": [
        {
          "Id": "chant",
          "On": "Arrived",
          "Waypoints": [4],
          "HoldMovement": true,
          "Steps": [ { "DelayMs": 0, "Anim": 123 }, { "DelayMs": 2500, "Say": "Omni-Tek out!" } ]
        }
      ]
    }
  ]
}
```

The ids, the first waypoints and the robots' flames are real data; the waypoint lists are cut short (loop A has 13,
loop B 18), and the Protester's chant is made up to show `Arrived`. One spawn point is in one entry, so loop B's
movement and its events are the same entry. The forks are the capture's: loop A mostly circles 4-11 (11 -> 4) but
sometimes cuts across 4 -> 9 or goes on to 12 and back to its point; loop B mostly circles 10-13, sometimes a bigger
circle back to 5, sometimes home over 14-17. The three standing Protesters and the 21 standing Cleaning Robots need no entry: an NPC with no
`Movement` from any entry stands.

#### Entry

| Field | |
| --- | --- |
| `Key` | unique in the file; names the entry in logs and diffs |
| target: | |
| `Hash` | global file only: every NPC of that template (`NpcTemplates.json` key), from any point on any playfield |
| `SpawnPoints` | playfield file only: spawn point ids (`Spawns.json` `id`); the NPCs that come out of these points |
| `Movement` | optional, below |
| `SpawnState` | optional: `NpcSpawnState` as today (`Pose`, `HealthPercent`) |
| `Events` | optional: `NpcEventDefinition` as today, plus `Id`, `Waypoints`, `HoldMovement` (below) |
| `RemoveEvents` | optional, spawn point entries: the hash's event ids that don't play at these points |
| `Provenance` | optional: where the data came from (`capture:<who> <date>`, `hand`) |

#### Movement

| Field | |
| --- | --- |
| `Mode` | `Stationary` (stays on its point, turns to fight only), `Patrol`, `Wander` |
| `Patrol` | `Waypoints` (`[x, y, z]` each), `Forks`, `Run`, `MinPauseSeconds` / `MaxPauseSeconds`. Walked in order, after the last back to waypoint 0. On a `Hash` entry (or one with several spawn points) the route is shared: each NPC starts at the waypoint nearest its spawn point. On an entry with a single spawn point the route is that point's own and **waypoint 0 is the spawn point** (its position is the point's). No 60 m cut |
| `Forks` | optional: `{ "<waypoint>": [<waypoint>, ...] }`. From a waypoint listed here it goes on to one of those, picked evenly at random, instead of the next in order. One waypoint in the list is a fixed jump (a route whose walk doesn't follow the list's order there) |
| `Wander` | `{ "Radius": 10, "MinPauseSeconds": 3, "MaxPauseSeconds": 12 }`: after each pause, a spot of its own choosing within the radius of its spawn point |
| `ReturnTo` | after a fight: `Route` (nearest waypoint of its patrol, default for `Patrol`) or `Home` (its spawn point) |

Where a template's NPCs all move the same way, it goes on the `Hash` entry: the Malfunctioning Cleaning Robots (MLCE)
all wander their yard. Where they walk different routes, each route goes on its spawn point's entry: JUCO's two
walking points each have their own, and its other 21 points stand (no `Movement` from any entry).

What was captured fits these:

- `Patrol`, with forks where there is a choice: the two walking Protesters are two entries, one per spawn point, each with its own loop
  (`PTV7MXAR` loop A, `5X7YERRS` loop B). JUCO's two walking Cleaning Robots are likewise two entries. The ICC
  Peacekeepers and Garbage Fleas too: a route with a fork here and there, not a random walk. They step to a nearby
  waypoint (a typical move 7 m, while their waypoints lie 42 m and 67 m apart on average), and most waypoints have
  one way on (1.3 and 1.8 exits on average, the most used taking 83 % and 67 % of moves).
- `Wander`: the Malfunctioning Cleaning Robots. From each of the 8 places they were seen going to they went on to 4-5
  of the 7 others, and a move is as long as the average distance between two of them (15 m and 16 m): no route, a
  roam around a small yard. Their spawn points lack bit 0x20, like every other point that doesn't walk a route. (The
  capture shows them coming back to the same 8 places; a wander picks new ones in the same area.)

A fork taken 10 % or less of the times its waypoint was left is not kept (a resent order, a step aside).

#### Events

Unchanged `NpcEventDefinition` / `NpcEventStep`, with:

- `Id`: names the event, so a spawn point entry can replace (same `Id`) or remove (`RemoveEvents`) one of its hash's.
- `On: "Arrived"` (new trigger): the NPC reached a patrol waypoint; `Waypoints` limits it to those waypoint indices (empty: any).
- `HoldMovement`: the patrol waits until the event's last step has played (a wave, a speech), instead of its pause.

New behaviour stays in steps, so "stop, dance, say something, walk on" is an `Arrived` event with `HoldMovement`.

### 3. Merge

Two levels:

```
Hash (the NPC's template)  <  SpawnPoints (the point it came out of)
```

The template's hash entry applies to every NPC of the template; the spawn point's entry is merged on top for the NPC
of that point. `NpcTemplates.json` itself no longer has `Events`, `SpawnState` or `Patrol`: what it carried is its
hash's entry now.

- `Events`: the hash's and the spawn point's are both played. A spawn point event with the same `Id` as one of the
  hash's replaces it; `RemoveEvents` drops hash events at that point. Events without `Id` always add.
- `Movement` and `SpawnState`: the spawn point's, if it has one, else the hash's, whole (a route is not merged waypoint
  by waypoint). So a spawn point can stand (`Stationary`) or walk a route of its own while the rest of its template
  walks the hash's route.
- No `Movement` from either, including an NPC whose hash and spawn point have no entry at all: `Stationary`. An NPC
  only moves when `NpcEvents.json` says so.

Placed NPCs (`Npcs.json`) come from no spawn point and are not in this; they keep their own fields.

So the Cleaning Robots stand (no entry gives JUCO movement), the two walking points get their `Patrol` from their
`SpawnPoints` entries, and every robot keeps the hash's events from the global file.

### 4. Server

- `PlayfieldSpawnsData` / `PlayfieldSpawnEntry`: `Id`. `HashSpawnSystem` keeps the id on `HashSpawnPoint`.
- New loader in `IGameData` for the global `GameData/NpcEvents.json` (by hash) and `PlayfieldContent/<pf>/NpcEvents.json`
  (next to `GetPlayfieldNpcs`; by spawn point id).
- `SpawnService.SpawnMob` gets the spawn point id (it already gets `spawnHash`); after the template is applied it
  resolves the merged entry and attaches it to the `NpcCharacter`: movement to the brain (`EnablePatrol` takes the
  route, starting at waypoint 0 on a single spawn point's route, else at the waypoint nearest its spawn point, no
  radius filter; `TryPickPatrolPoint` walks the waypoints in order, taking a fork where there is one; `EnableWander` as now), spawn state, events to `NpcEventService`, which reads
  them from the NPC instead of `npc.MobTemplate.Events`.
- `NpcEventService`: the `Arrived` trigger from the brain's patrol, `HoldMovement`, `Id` / `RemoveEvents`.
- Validation (`GameDataContractCheck`, and on load as a warning): ids that match no point, hashes with no template, a hash or a
  spawn point in more than one entry, `Hash` in a playfield file or `SpawnPoints` in the global one,
  `Forks` naming a waypoint that isn't there, a single spawn point's `Patrol` whose waypoint 0 is not its point's
  position (0.1 m), and the flag check on what each point ends up with after the merge: a point with bit 0x20 that
  won't patrol, or one without it that will (a `Wander` on a point without it is expected).

### 5. Moving existing data

- Template `Patrol`: for each hash, the migration splits it into its connected parts (separate routes). Patrols
  recorded before aosharp `add22b0` are not used (they contain fight movement).
  - One route, walked by every point of the hash (MLCE): a `Hash` entry with it.
  - Otherwise each route goes to the flagged points on it, a `SpawnPoints` entry per point (JUCO's two, ROTE's two):
    the waypoint the point lies on becomes waypoint 0, set to the point's exact position, the others renumbered from
    it. The hash's other points stand (the default).
  - Points that can't be matched (no waypoint within 5 m) are reported, not guessed.

  A route becomes a `Patrol`: its waypoints ordered so that as many moves as possible go to the next one (on a spawn
  point's route starting at the point), and `Forks` for every waypoint whose kept moves don't: more than one way on,
  or one that isn't the next. Moves taken 10 % or less of the times their waypoint was left are dropped first. A
  walk where most places lead to most others (the Malfunctioning Cleaning Robots) is no route: it becomes `Wander`,
  with a radius that takes in the places it went to.
- Template `Events` / `SpawnState`: moved unchanged to a `Hash` entry in the global `GameData/NpcEvents.json`, then
  one-off scenes narrowed to their playfield or spawn points by hand.
- One change, no period of reading both: the same commit moves the data, removes the three fields from
  `NpcTemplates.json` and from `MobTemplate`, and switches `SpawnService` / `NpcEventService` / `NpcBrain` to the
  merged entry. `GameDataContractCheck` refuses a template that still has one of them.

## Phases

1. Extractor: spawn point ids in `Spawns.json`; server reads them.
2. `NpcEvents.json` loader, merge, movement (`Stationary` / `Patrol` / `Wander`), events and spawn state from the
   file, `Id` / `RemoveEvents`; in the same change all template `Patrol` / `Events` / `SpawnState` moved out and the
   fields removed. Live check on 6553: 21 Cleaning Robots stand, two walk the loop; ROTE's two loops each walked by
   its own Protester, three standing; the robots' flames and sparks, the Wounded Dockworker and the Omni-AF Private
   play as before.
3. `Arrived` and `HoldMovement`.

## Open questions

- Which wins after a fight: back to the nearest waypoint, or to the spawn point first?
- The Garbage Flea route merges ten Fleas from ten spawn points, and the ICC Peacekeepers' has only 53 moves. Some of
  their forks may be where two NPCs' separate routes meet; captures that keep each NPC's walk apart will tell whether
  they are one forked route on the hash or routes of their own per spawn point.
- What the other `nativeFlags` bits mean (0x40 is on three named roamers; the low three vary). Worth the same check
  as 0x20 once more playfields are captured.
- Are current SpawnState actally just an item?
- the MLCE npc dont have the flag set, but they seems to be wander? and they are not a districtspawn ?
