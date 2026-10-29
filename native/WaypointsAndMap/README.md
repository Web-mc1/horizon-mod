# Waypoints for Horizon

One Horizon code mod (`.hmod`, WebAssembly) with three parts:

- **Waypoints**: save places, name them, colour them, hide them, track one.
- **Minimap**: the game's own terrain minimap, with your waypoints drawn on it.
- **World map** (`M`): the game's real terrain around you (waypoints named on
  it), or a full-screen chart you can pan (`V` switches).
- **In the 3D view**: every waypoint shows as a marker where it is, with its
  name and distance when you look near it; the tracked one pins to the screen
  edge when it is off-screen.

## Build and try it

You need Rust 1.95+ and the `wasm32-unknown-unknown` target:

```sh
rustup target add wasm32-unknown-unknown
cargo install --git https://github.com/horizon-rust/horizon cargo-horizon-mod   # once

cargo test                      # geometry + storage unit tests (no game)
cargo horizon-mod build
cargo horizon-mod test          # runs tests/commands.toml
cargo horizon-mod run --game /path/to/horizon-client
```

The mod needs the experimental `render`, `input` and `ui` APIs (declared in
`horizon-mod.toml`), so your Horizon build must have them: the official
Minimap, Keystrokes and Zoom mods use the same ones. `Cargo.toml` takes the
`horizon-mod-sdk` crate from a Horizon checkout (`path = ...`) or from Git
(`git = ..., rev = ...`); use the revision your Horizon build comes from.

To play with it, copy `target/horizon-mod/waypoints.hmod` into `horizon-mods/`
beside the game's `options.txt` and switch it on in Options > Mods.

## Keys

All rebindable in Options > Mods > Waypoints.

| Key | Does |
| --- | --- |
| `B` | Add a waypoint where you stand |
| `M` | Open / close the world map |
| `N` | Waypoint manager (list, delete, hide, recolour) |
| `[` `]` | Select previous / next waypoint (tracked on the maps and in the 3D view) |
| `V` | World map: terrain view / chart view |
| Mouse wheel | World map zoom (while the map is open) |
| Numpad `8` `4` `2` `6` | World map pan (switches to the chart, the only view that can move) |
| `Home` / `End` | World map (chart): centre on me / fit everything |

The arrow keys are not used on purpose: the game turns your camera with them
while you play. Rebind the pan keys in Options > Mods > Waypoints if you have
no numpad, or use `/wp pan up|down|left|right [blocks]`.

## Commands

`/wp` (not `/waypoint`: vanilla owns that one)

```
/wp add <name> [x y z]      /wp del <name>          /wp list
/wp go <name>               /wp show|hide <name>    /wp color <name> <color>
/wp rename <old> = <new>    /wp map                 /wp menu
/wp view terrain|chart      /wp pan up|down|left|right [blocks]
/wp minimap on|off          /wp world <name>        /wp trail clear
```

Names match by their start (`/wp del ho` finds `Home`). Colours: red, orange,
yellow, green, aqua, blue, purple, pink, white, gray.

## How the two maps work

**Minimap.** Terrain is drawn by the game (`render.set-minimap`: vanilla map
colours, height shading, the floor underground). The mod draws coloured diamonds
over it with `render.set-hud`, which the game draws after the minimap. To land
them in the right place it uses the game's own layout rules (the minimap is the
first overlay in its corner, 2 px from the edges, 128 image pixels scaled to the
chosen size) and the same north-up / turning-with-you maths as the game's map
renderer. Waypoints beyond the map stick to its rim as smaller markers. The
tracked waypoint's name, distance and compass direction show under (or over)
the map.

**World map, terrain view.** Mods cannot read blocks yet (`world-read` is
still a "planned interface" in `docs/mods.md`), but the game's own map can be
shown big: while the map is open the mod asks for it at 256 px (the biggest the
game allows), north up, 0.5 to 4 blocks a pixel (wheel to zoom: up to 512
blocks across), and draws your waypoints, their names and a scale over it. It
is always centred on you: the game builds it around the player and a mod
cannot move it.

**World map, chart view.** What can pan: a full-screen chart with a grid, your
waypoints with names and distances, the 32-block squares you have been near
(saved per dimension, up to 12,000 each), a scale bar and your arrow. Panning
from the terrain view switches to it. Real terrain that you can pan needs a
game-side change (a map the mod can centre elsewhere, or `world-read`).

**3D view.** There is no world-space drawing for mods yet either, so each
waypoint is projected onto the screen from your eye position, where you look
and your field of view (same maths as the game's camera), and drawn as a HUD
marker. Set "My field of view" in Options > Mods > Waypoints to your FOV
setting (default 70). It is exact in first person with the FOV option you
set; sprinting or speed effects widen the game's FOV a little, so a far
marker can drift a few pixels then. Third person is not accounted for. Only
this dimension's waypoints are marked in the view (the 8:1 ones show on the
maps).

## Worlds and dimensions

- Storage belongs to the mod, not to a world, so waypoints are kept in named
  **sets**: `/wp world <name>` switches to (or creates) one, up to 12. Use one
  per save or server.
- In the Nether, Overworld waypoints show at 1/8 distance, and the reverse, so
  you can plan portals (Settings: *Show Overworld/Nether waypoints in both*).
  Other dimensions only show their own.

## Things to know

- **Use only one minimap mod.** The game shows the first loaded mod's map, and
  the markers assume it is this one's. Turn the official *Minimap* mod off.
- **Corner.** The default is bottom right, where the position is exact. Top
  right shifts down under potion-effect icons and top left hides in F3; a mod
  cannot see either, so markers there can be a few pixels off.
- **Waypoints are local.** They are stored in your game's mod storage, nowhere
  else, and are not shared on servers.
- **Budget.** Horizon stops a mod that overruns its time budget (about 1 ms a
  tick). Overlays are rebuilt at most 20 times a second, from at most 1,024
  shapes; if a very busy world map ever stops the mod, lower `TRAIL_DRAW` or
  `MAX_LABELS` (`src/lib.rs`, `src/draw.rs`).

## Layout

```
horizon-mod.toml   manifest: capabilities game-read, chat, render, input, ui, storage
src/lib.rs         state, input, commands, manager screen, settings page
src/geom.rs        all map maths (unit tested)
src/draw.rs        minimap markers and the world map as render shapes
src/model.rs       waypoints, storage format, name lookup (unit tested)
tests/commands.toml   scenario test for /wp
```
