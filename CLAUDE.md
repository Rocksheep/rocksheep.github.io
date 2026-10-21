# Rocksheep Arcade

A GitHub Pages site (`rocksheep.github.io`). No build step, no package.json, no framework:
every game is **one self-contained HTML file** in the repo root, and `index.html` is the arcade
hub that lists them.

```
index.html              arcade hub: GAMES list, filters, NEW!/NEW UPDATE! badges
<game>.html             one file per game (HTML + CSS + JS inline)
assets/<game>.svg       cover art for the hub tile (320x200 viewBox, pixel-art style)
.claude/launch.json     "arcade" dev server: python3 -m http.server 8765
```

Run it locally with the `arcade` preview config (or `python3 -m http.server 8765`) and open
`http://localhost:8765/`. A static server is needed because of ES module import maps and
the PeerJS script; `file://` mostly works but don't rely on it.

## Adding a game to the hub

Add an object to the `GAMES` array near the top of the `<script>` in `index.html`. The comment
above the array documents every field. Things that are easy to forget:

- `added` **and** `updated`: ISO timestamps with offset (`date -Iseconds`). New games set both
  to the same time.
- When shipping a real update to an existing game (features, not typo fixes), bump its
  `updated`, or visitors won't see NEW UPDATE!. Badges are per visitor (localStorage
  `arcade-played`), so nothing else needs touching.
- `tags`: reuse existing tags where possible (`Multiplayer`, `Strategy`, `Racing`, `Action`,
  `Arcade`, `Simulation`, `3D`…). New tags get a filter chip automatically; give important
  ones a colour in `TAG_COLORS`.
- `image`: an SVG in `assets/`, same 320x200 viewBox and chunky pixel look as the others.
- Friends' games set `by` + `repo` (+ `path`) and only need `added`; their update date comes
  from the GitHub API.

Newest games go at the top of `GAMES`.

## Shared look and feel

All games reuse the hub's palette and fonts so they feel like one arcade:

- Fonts: Google Fonts `Press Start 2P` (headings, buttons, labels) and `VT323` (body text).
- CSS tokens: `--bg #0f0820`, `--panel #21143f`, `--line #4b3590`, `--ink #efe6ff`,
  `--dim #a898cf`, `--yellow #ffd23f`, `--pink #ff4fa3`, `--cyan #3ff0ff`, `--green #7dff6a`,
  `--orange`, `--shadow #05020c`.
- Overlays: `#setup`, `#pause`, `#over` are `.overlay > .panel` blocks toggled with a
  `show(id)` helper. The setup panel has a `◀ ARCADE` back link, an `h1` with a `span`
  subtitle, a `.seg.tabs` row with **ON THIS SCREEN / HOST ONLINE / JOIN ONLINE** (or
  **VS CPU / HOST ONLINE / JOIN ONLINE**), and a `<details>` HOW TO PLAY block.
- Buttons: `.btn` with a `--c` colour; `aria-pressed="true"` marks the chosen option in a
  `.seg` group (built by a `seg(id, options, current, pick, color)` helper).
- Sound: made on the fly with WebAudio (`audio()`, `tone()`, `noise()`, an `SFX` table and
  `sfx(name)`). `audio()` must run from a user gesture (the start/join buttons call it).
- Settings persist in localStorage under a per-game `SAVE_KEY`, always wrapped in try/catch
  (private mode throws). The `host` mode is never restored on reload; `join` is.
- External scripts only from cdnjs / jsdelivr with pinned versions (three.js via an import
  map in the 3D games, PeerJS for online play).

## Multiplayer (online play)

Every multiplayer game uses the same recipe, copied between files rather than shared:
**PeerJS 1.5.4 over WebRTC, with PeerJS's free public broker for matchmaking. There is no
server of our own.**

### Rooms and joining

- `const PEER_LIB = 'https://cdnjs.cloudflare.com/ajax/libs/peerjs/1.5.4/peerjs.min.js'` is
  loaded lazily by `loadPeerLib()` the first time someone hosts or joins.
- `ROOM_PREFIX = 'rocksheep-<game-name>-'`. The host registers peer id
  `ROOM_PREFIX + code` where `code` is 5 chars from `ABCDEFGHJKLMNPQRSTUVWXYZ23456789`
  (`roomCode()`). If the id is taken (`unavailable-id`) it simply tries a new code. **Each
  game needs its own prefix** so rooms of different games never collide.
- Guests create an anonymous `Peer()` and `peer.connect(ROOM_PREFIX + code, { reliable: true,
  serialization: 'json' })`, then send `{ t: 'hello', name, … }`.
- Invite link: `?room=CODE` on the game URL. On load the game switches to JOIN ONLINE and
  fills in the code (`roomParam`).
- Error handling to copy: `peer-unavailable` → "No room called X"; a 15 s timeout when the
  room exists but the data channel never opens (strict NAT / office Wi-Fi); `network`,
  `server-error`, `socket-error` → "Lost the connection to the matchmaking server";
  `peer.on('disconnected')` → `peer.reconnect()`.
- `net` object holds `{ role: 'offline' | 'host' | 'client', peer, conn, code, players, lobby }`.
  `leaveOnline()` closes everything and resets it. `send(conn, m)` swallows errors on dead
  connections; `broadcast(m)` sends to every guest.
- Names from the network always go through `cleanName()`, and every number through
  `clamp()` / `isFinite` checks: treat guest messages as untrusted.

### Messages

All messages are small JSON objects with a `t` type field. The common ones:

| t          | direction      | meaning |
|------------|----------------|---------|
| `hello`    | guest → host   | name (and car/race/etc.) on connect |
| `full`     | host → guest   | room is full |
| `lobby`    | host → guest   | current setup + player list, re-sent on every change (`broadcastLobby()`) |
| `start`    | host → guest   | everything needed to build the same level, plus which slot is yours |
| `in`/`cmd` | guest → host   | the guest's input (held keys, or RTS commands) |
| `s`/`P`    | host → guest   | world snapshot, ~10–20 times a second |
| `over`/`res` | host → guest | results screen |

### Two sync models in use

1. **Host-authoritative snapshots** (Tabletop Turbo, Caterpillar Chaos, Lord of the Pings):
   the host's browser runs the only real simulation. Guests send input/commands and render
   snapshots (`game.replica = true`, interpolating between the last two). Use this when
   things interact (collisions, combat, shared resources). Tips:
   - Skip a snapshot for a guest whose `conn.dataChannel.bufferedAmount` is large instead of
     queueing lag.
   - Send rarely-changing state as deltas since that guest's last snapshot (`p.sentFrame`),
     with a full copy in `start` for late joiners.
   - Cosmetic events (sounds, explosions) ride along in an `e`/`ev` array on the snapshot.
   - A guest who disconnects mid-game is handed to the CPU.
2. **Relay** (Super Sheep Ball): every browser simulates its own player and sends its state;
   the host relays everyone's state and keeps score. Use this only when players never
   affect each other.

Levels/maps are generated from a **seed** sent in `start`, using a seeded PRNG (`mulberry32`)
and never `Math.random()` during generation, so host and guests build identical worlds
without sending them.

### Testing online play locally

Open two tabs of the dev server: host in one, join with the code in the other. PeerJS's
broker is on the internet, so this needs a network connection even on localhost.

## Lord of the Pings (the RTS) in short

`lord_of_the_pings.html` is the biggest game; its script is split into banner-commented
sections in this order:

- **Constants and game data**: `BASE` stats per kind (worker, melee, ranged, special, hall,
  farm, barracks, lab, tower, mine, mill, flyer, siege, ultimate, wall, workshop, altar, shipyard,
  bridge, transport, warship, battleship; the index in `KINDS` goes over the network, so add new kinds
  at the end and update `isUnitK`/`isBldK`/`isBoatK`) and `RACES` (Humans, Orcs, Goblins, Funguys, Bugs,
  Undead, Mechs, Lemmings) with
  names, jokes, a passive and per-kind overrides (`over`). `DEFS[race][kindIndex]` is the
  merged result. Each race also has a signature rule set by flags on its `RACES` entry
  (`repair`, `rage`, `plunder`, `train`, `blast`, `myco`, `carry`, `raise`, `shield`, `unitCost`, `navyCost`,
  `pop`, `fuse`, `bridge`, ...) that the simulation checks by name. Ship names, descriptions and boat
  overrides per race live in `NAVY`, merged into `RACES` (a transport's `seats` is how much food it carries). Balance changes go in `BASE`/`RACES`, not in the simulation code. `AI_DIFF`
  tunes the CPU levels.
- **Map generation**: `genMap(seed, size, players)`, seeded and deterministic (see above).
  Up to 4 players use the corners; the HUGE map (112 tiles, `HUGE` in `SIZES`) adds a start on
  every side for up to 8 players, plus the Moat (a water ring with four bridges round a rich mine)
  and a road from every expansion to the ring road. Which player gets which spot is shuffled from
  the seed. When touching it, check every start and mine is reachable over many seeds, and run
  8-CPU free-for-alls: the CPU's rally point and target choice decide who gets ganged up on.
  `cfg.mapType === 'islands'` uses `genIslands()` instead: every base on its own island (channels
  at least two squares wide), a rich middle island, islets with mines, cleared harbours for
  shipyards, and fat trees (island wood runs out fast). `labelRegions(w)` numbers every island
  (`w.isle`, all non-water land) and stretch of water (`w.seaId`); it runs again when a bridge
  goes up or comes down. `canReach(w, u, t, d)` uses them so nobody chases a target across water.
- **The world**: `makeWorld(cfg, replica)` → `G`. Entities live in `G.ents` / `G.byId`;
  buildings and mines mark tiles in `G.occ`. `findPath()` is grid A* with a goal callback.
- **Orders**: every player action is a command object run through `applyCmd(world, slot, cmd)`
  (`move`, `attack`, `gather`, `return`, `build`, `train`, `cancel`, `rally`, `stop`, `hold`,
  `boom`, `board`, `unload`). Local players, CPU players and online guests all go through it, so it's the one
  place to validate and the one place to add a new order.
- **Simulation**: `step(w)` at 20 steps a second; units are small state machines
  (`unitTick`: idle, move, attack, gather, return, build, inside). Flyers (`e.fly`) skip
  pathfinding and fly straight; `canHit()` decides who can hit them (`hitsAir` on a def).
  Boats (`e.sea`) path over water only (`sail` instead of `walk`, see `passOf`); melee can't hit
  them. Passengers are `hidden` with `inId` = the transport and die with it. Lemming bridges
  are buildings that turn their water tile into `T_BRIDGE` once built (`bridgeUp`/`bridgeDown`).
  Watching: `cfg.you = -1` makes you a spectator (player 1 set to YOU WATCH, or a guest
  without a seat). `eyeOf(w)` says whose fog is shown (`w.pov`, -1 = everything) and
  `G.speed` speeds up the host's simulation.
  Upgrades are queue items `UP_ATK`/`UP_ARM` (lab) and `UP_WOOD` (mill), levels in `P.up`. Things the screen should
  show (sounds, projectiles, messages) are pushed to `w.ev` as `[name, ...args]` and played
  by `runEvent()`, on the host and on guests alike.
- **CPU players**: `aiThink()` once a second per CPU, issuing commands through `applyCmd`.
  When no enemy shares its island (`naval`), a CPU builds a shipyard, transports and warships,
  and `aiFerryStart`/`aiFerryTick` run a ferry: board by the shore, sail, unload next to the
  target. A second ferry (`wferry`) ships workers to another island once the home mine runs dry.
  `shoreNear()` finds landing spots by walking out from the troops, so they're always reachable.
- **Drawing**: everything is drawn with canvas calls (`drawUnit`, `paintBuilding`,
  `paintRegion`), no image files. Unit and building sprites are cached per race/colour/zoom.
  The grimdark look (`grimOn(w)`: the GRIMDARK land, or LOOK set to GRIMDARK) is purely
  visual and per screen: colour grading, firelight, ash, fog, grain and battle scars on top
  of the normal drawing.
- **Online**: host-authoritative. Guests send `{ t: 'cmd', c: <command> }`; the host sends
  snapshots (`encodeEnts`/`applySnapshot`, compact arrays per entity, felled trees as a
  delta per guest). Guests who join mid-battle spectate; a guest who leaves is replaced by a
  CPU.

To test the simulation without a browser, cut the script from "Constants" up to "Drawing:"
into a Node file, add a stub `endBattle`, and run `makeWorld` + `step` with CPU players.
A useful check after touching movement: flag units that are busy (move, gather, return, build)
but haven't changed state, cargo or position for 8 seconds. Sample every step or two, not
once a second, or a 2-second mining loop looks stuck.
For balance, run a round-robin: every race against every other race, CPU hard vs hard,
several seeds with both start corners, and compare win rates. Aim for every race between
roughly 40% and 60%, on both SMALL (48) and MEDIUM (64) maps, and on both map types (CONTINENT
and ISLANDS: island games are slower, 12 minutes, and cost discounts and transport seats matter much
more there). Also watch the number of games that hit the time limit on islands. CPU games lean on economy and
defence (towers, repair, regen), so also run equal-cost army fights on open ground (spawn the
same gold+wood worth of units per side, attack-move into each other): a race that wins nearly
every even fight will feel overpowered to a human player even at 50% in CPU games.

## Commits

Commit messages describe the game for players (what it is, what you can do), not the code.
New-game commits are titled `Add <Game>, a <one-line pitch>`.

## Polderheim (the city builder) in short

`polderheim.html` is a single-player Caesar III / Pharaoh-style city builder on the Frisian coast
around 690 AD. Its script is split into banner-commented sections:

- **Constants and game data**: `BLD` (every building: size, cost, workers, `des` = [desirability,
  range], `fire`, `makes`/`uses`/`rate` in units a month), `HOUSES` (the six house levels, capacity
  and tax), `GODS` (Wodan, Donar, Frija, Nehalennia, Fosite, each with a wrath and a blessing),
  `GOOD`, `MKT_MAX`. Balance changes go here.
- **Map generation**: `genMap(seed, N)`, seeded: sea on the low-x edge, a river to the sea, `elev`
  0-3 from the distance to the sea (0 = salt marsh), fertility, bogs, trees, the old road in from
  the inland edge (`entry`, where settlers arrive) and a hunebed.
- **The world / roads / walkers / simulation**: no DOM, so it runs in Node. Everything moves on
  foot along roads: settlers (`immigrant`), handcarts (`cart`, `fetch`), the market `buyer`, and
  roaming service walkers (`seller`, `priest`, `watch`, `tax`, `skald`) that serve every building
  within 2 tiles of where they step (`serve`). `houseBest()` decides what level a house can reach,
  `houseWants()` says why in words. `newMonth()` does taxes/wages, eating, fires, gods, storms
  (`floodReach(level)`: water spreads over land lower than the storm, dikes block it) and mood.
  The simulation talks to the screen only through `G.ev` (`['msg', text, kind, tile]`, `['sfx', name]`).
- **Drawing**: all art is canvas code (`ART[type]`, `box`, `gable`, `hip`, `roofPlane`, `wallTex`),
  cached per zoom level in `spr()`. Ground colour is one N×N canvas drawn through the iso transform
  (`drawTint`) so tiles blend. Objects are drawn per diagonal: a building at its tile
  (x + size - 1, y), walkers on the bigger x+y of the two tiles they walk between.

To test without a browser, cut the script from "Constants" up to "Drawing:" into Node, call
`makeWorld({ seed, size: 'medium', diff: 'normal' })`, lay roads/buildings with `build()`, and run
`step()` (20 steps = 1 second, 200 = 1 month). Watch pop, silver, mood, house levels and whether
farms say "Nowhere to take the goods".
