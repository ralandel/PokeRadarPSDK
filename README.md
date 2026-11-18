# PokeRadar

A Poké Radar for Pokémon SDK, in the style of Diamond / Pearl / Platinum and X / Y. Use it in the grass to make a few patches rustle, walk into one to battle the Pokémon hiding there, and chain battles against the same species to raise your shiny odds.

- **Version:** 1.0.0.0
- **Author:** ralandel
- **Dependencies:** none

---

## Table of contents

1. [What the plugin does](#what-the-plugin-does)
2. [How to use](#how-to-use)
3. [Installation](#installation)
4. [Uninstallation](#uninstallation)
5. [Graphics](#graphics)
6. [Texts](#texts)
7. [Configuration](#configuration)
8. [Monkey-patching](#monkey-patching)
9. [Public API reference](#public-api-reference)
10. [Technical notes and limitations](#technical-notes-and-limitations)
11. [Changelog](#changelog)

---

## What the plugin does

### Using the radar

When the player uses the radar in the grass:

1. **Up to 4 patches of grass rustle** around the player (`PATCH_COUNT`).
   - They are picked between 2 and 6 tiles from the player (`MIN_DISTANCE` / `MAX_DISTANCE`), with at least 2 tiles between two patches (`MIN_SPACING`).
   - Only tiles on screen are used.
   - A tile is eligible only if it is `TGrass` or `TTallGrass`, passable, free of any visible event, and if a wild group of the current zone matches its system tag **and** terrain tag. A patch therefore never hides a Pokémon that can't live there.
2. **The rustle is a short heads-up.** Each patch plays its animation for about 2.5 seconds (`PARTICLE_DURATION`), then stops. The patches are still remembered: walking into one afterwards still starts its battle.
3. **The battery is emptied.** It takes 50 steps to recharge (`STEPS_TO_RECHARGE`).

The radar fails, and shows a message, when:

- the battery isn't charged yet;
- there is no eligible grass on screen. This also empties the battery.

### Kinds of patches

| Kind       | Default weight | Chance to give the chained species | Look |
|------------|----------------|------------------------------------|------|
| Weak       | 75%            | 50%                                | Light rustle (row 1 of the image) |
| Strong     | 25%            | 85%                                | Strong rustle (row 2 of the image) |
| **Shiny**  | rolled         | 100%                               | Rustle with sparkles (`radar_shiny.png`) |

Each patch rolls for shiny on its own (see [Chain and shiny odds](#chain-and-shiny-odds)). **The particles never lie:** a shiny patch always gives a shiny Pokémon, and a regular patch never does (`FORCE_NON_SHINY_ON_NORMAL_PATCH`). An encounter set to *always* or *never* shiny in Pokémon Studio keeps that setting.

### Encounters while the radar is active

- **Walking into a patch** starts a wild battle with a Pokémon from the matching group, exactly like a normal grass encounter (same levels, forms, abilities, Repel…).
- **Random encounters are blocked** while walking inside the area that was on screen when the radar was used (`SAFE_STATES`). When running or cycling, a random Pokémon can still show up, and that breaks the chain.

### The chain

- Defeating or catching the Pokémon of a patch **adds 1 to the chain** if it is the same species and form as the previous one. A different species starts a new chain at 1.
- After a won battle or a capture, **the radar is used again automatically** (`AUTO_RESCAN_AFTER_BATTLE`), so new patches rustle without having to recharge.
- The chain is capped at 999 (`CHAIN_MAX`).

**The chain breaks when:**

- the player flees, loses, or the wild Pokémon flees (this also empties the battery);
- a random encounter happens;
- the player leaves the area that was on screen (`BREAK_CHAIN_ON_LEAVING_ZONE`);
- the player steps out of the grass (`BREAK_CHAIN_ON_LEAVING_GRASS`);
- the player changes map;
- a roaming Pokémon takes the place of the patch encounter;
- the game is reloaded.

The patches are forgotten (but the chain is kept) after 50 steps without a battle (`PATCH_LIFETIME_STEPS`, `BREAK_CHAIN_ON_EXPIRE`).

### Chain and shiny odds

The base rate comes from PSDK (`PFM::Pokemon#shiny_rate`, 1/4096 by default). Each tier reached in `CHAIN_TIERS` halves the denominator:

| Chain | Shiny odds per patch |
|-------|----------------------|
| 0–4   | 1/4096 |
| 5     | 1/2048 |
| 10    | 1/1024 |
| 15    | 1/512  |
| 20    | 1/256  |
| 30    | 1/128  |
| 40+   | 1/64   |

The Shiny Charm and any other source of extra shiny attempts apply as usual. The fishing chain bonus is ignored, since it has nothing to do with the grass.

---

## How to use

1. In Pokémon Studio, create the Poké Radar item and **link it to a common event** (for example common event 100). Make it a key item so it isn't consumed.
2. In that common event, add a **Script** command containing:

   ```ruby
   poke_radar
   ```

That's all. `poke_radar` uses the radar and shows the right message if it fails.

---

## Installation

1. Put `PokeRadar.psdkplug` in the `scripts/` folder of your project.
2. Start the game once (from Pokémon Studio or `Game.exe`). The PSDK Plugin Manager then:
   - checks that your PSDK version provides every method the plugin needs;
   - extracts the scripts to `scripts/00000 Plugins/xxxxx PokeRadar/`;
   - extracts the texts to `Data/Text/Dialogs/400000.csv`.
3. Add the particle images (see [Graphics](#graphics)).
4. Link an item to the radar (see [How to use](#how-to-use)).

> **Do not edit the extracted files.** The Plugin Manager wipes and re-extracts `scripts/00000 Plugins/` every time the plugin list or the PSDK version changes. Put your settings in a project script instead (see [Configuration](#configuration)).

---

## Uninstallation

1. Delete `scripts/PokeRadar.psdkplug`.
2. Start the game once. The Plugin Manager removes the extracted scripts.
3. Delete `Data/Text/Dialogs/400000.csv` (and its `.dat` files), the particle images, and your own settings script if you made one.

The plugin saves nothing in the save file, so existing saves keep working after uninstalling.

---

## Graphics

**The plugin ships no graphics.** Put your own images in `graphics/particles/`. If an image is missing, the radar still works, but the patches don't show anything.

### Regular patches

Name: `<system_tag>_<terrain_tag>_radar.png`

| Tile                        | File name |
|-----------------------------|-----------|
| `TGrass`, terrain tag 0     | `grass_0_radar.png` |
| `TGrass`, terrain tag 3     | `grass_3_radar.png` |
| `TTallGrass`, terrain tag 0 | `tall_grass_0_radar.png` |

If the exact file doesn't exist, the plugin falls back to `<system_tag>_0_radar.png`, then to `grass_0_radar.png`. **`grass_0_radar.png` alone is enough** to cover every kind of grass.

Layout: **2 rows** of **square frames** laid out horizontally.

- Row 1: weak rustle.
- Row 2: strong rustle.

The frame size is `image height / 2`, and the frame count is `image width / frame size`. Example: 4 frames of 32×32 → a 128×64 image.

### Shiny patch

Name: `radar_shiny.png` (one file for every kind of grass).

Layout: **1 row** of square frames. Example: 4 frames of 32×32 → a 128×32 image.

### Animation speed

Each frame lasts `PARTICLE_TICKS` game frames (weak: 8, strong: 4, shiny: 4). The animation loops for `PARTICLE_DURATION` game frames (150 = 2.5 s), and always ends on a complete loop.

---

## Texts

The failure messages are in `Data/Text/Dialogs/400000.csv`, in every classic PSDK language (`en`, `fr`, `it`, `de`, `es`, `ko`, `kana`).

| Line | Setting key   | English text |
|------|---------------|--------------|
| 0    | `not_charged` | The Poké Radar's battery isn't charged yet... |
| 1    | `no_pokemon`  | There are no Pokémon around... |

- Edit the CSV to change the wording or add a language column.
- If your project already uses file `400000`, move the CSV to another ID and set `TEXT_FILE_ID`.
- In `MESSAGES`, each failure maps to a line number, a raw `String`, or `nil` to show nothing. `no_party` (no Pokémon able to battle) shows nothing by default.

---

## Configuration

### All settings

The default values live in `000 Config.rb`:

| Setting                           | Default | Meaning |
|-----------------------------------|---------|---------|
| `GRASS_TAGS`                      | `[TGrass, TTallGrass]` | System tags where patches can appear |
| `PATCH_COUNT`                     | `4`     | Patches per use |
| `MIN_DISTANCE`                    | `2`     | Minimum distance (tiles) between the player and a patch |
| `MAX_DISTANCE`                    | `6`     | Maximum distance (tiles) between the player and a patch |
| `MIN_SPACING`                     | `2`     | Minimum distance (tiles) between two patches |
| `STEPS_TO_RECHARGE`               | `50`    | Steps needed to recharge the battery (`0` = no recharge) |
| `PATCH_LIFETIME_STEPS`            | `50`    | Steps before the patches are forgotten (`0` = never) |
| `BREAK_CHAIN_ON_EXPIRE`           | `false` | Whether the patches expiring also breaks the chain |
| `BREAK_CHAIN_ON_LEAVING_ZONE`     | `true`  | Whether leaving the on-screen area breaks the chain |
| `BREAK_CHAIN_ON_LEAVING_GRASS`    | `true`  | Whether stepping out of the grass breaks the chain |
| `SAFE_STATES`                     | `%i[walking swamp]` | Player states during which random encounters are blocked in the area (`[]` = never blocked) |
| `AUTO_RESCAN_AFTER_BATTLE`        | `true`  | Use the radar again after a won battle or a capture |
| `CHAIN_CONTINUE_CHANCE`           | `{ weak: 50, strong: 85, shiny: 100 }` | % chance for each kind of patch to give the chained species |
| `KIND_WEIGHTS`                    | `{ weak: 75, strong: 25 }` | Draw weights of the regular patches |
| `CHAIN_TIERS`                     | `[5, 10, 15, 20, 30, 40]` | Each tier reached halves the shiny denominator |
| `CHAIN_MAX`                       | `999`   | Maximum chain length |
| `FORCE_NON_SHINY_ON_NORMAL_PATCH` | `true`  | A regular patch never gives a shiny |
| `PARTICLE_FILENAME`               | `'%<tag>s_%<terrain>d_radar'` | File name format of the regular patch images |
| `PARTICLE_FALLBACK`               | `'grass_0_radar'` | Image used as a last resort |
| `SHINY_PARTICLE`                  | `'radar_shiny'` | Image of the shiny patch |
| `PARTICLE_ROWS`                   | `{ weak: 0, strong: 1 }` | Image row used by each kind of regular patch |
| `PARTICLE_ROW_COUNT`              | `2`     | Number of rows in the regular patch images |
| `PARTICLE_TICKS`                  | `{ weak: 8, strong: 4, shiny: 4 }` | Game frames per animation frame |
| `PARTICLE_DURATION`               | `150`   | Game frames the rustle is shown (60 = 1 s) |
| `TEXT_FILE_ID`                    | `400000`| ID of the text file in `Data/Text/Dialogs/` |
| `MESSAGES`                        | `{ no_party: nil, not_charged: 0, no_pokemon: 1 }` | Failure messages (line number, `String` or `nil`) |

### Changing a setting

Project scripts are loaded **after** plugins. Create a project script anywhere under `scripts/`, outside `00000 Plugins`. Its folder and file names must start with digits. For example:

`scripts/01 Plugin Settings/000 PokeRadar Settings.rb`

```ruby
PokeRadar.configure(
  STEPS_TO_RECHARGE: 30,          # Faster recharge
  PATCH_COUNT: 5,
  PARTICLE_DURATION: 120,         # 2 s heads-up
  SAFE_STATES: [],                # Random encounters stay possible (closer to DPPt)
  CHAIN_TIERS: [10, 20, 30, 40]
)
```

`configure` raises an error if a setting name is misspelled, so typos are caught at boot.

---

## Monkey-patching

All the logic lives in the `PokeRadar` module (module methods). The cleanest way to change its behaviour is to `prepend` a module to `PokeRadar.singleton_class` from a project script. Call `super` to keep the original behaviour, and it will survive plugin updates.

### Forbid the radar on some maps

```ruby
module PokeRadarMapBlacklist
  NO_RADAR_MAPS = [12, 45]

  def use(force: false)
    return false if NO_RADAR_MAPS.include?($game_map.map_id)

    return super
  end
end
PokeRadar.singleton_class.prepend(PokeRadarMapBlacklist)
```

### Use DPPt-like shiny odds

`shiny_settings` is private and returns `[rate over 65536, attempts]`:

```ruby
module PokeRadarDPPtOdds
  private

  def shiny_settings
    _, attempts = super
    rate = 65_536 / [8200 - 200 * chain, 200].max
    return rate, attempts
  end
end
PokeRadar.singleton_class.prepend(PokeRadarDPPtOdds)
```

### Other private methods you can override

- `scan_candidates(zone)`: eligible tiles in the on-screen area.
- `eligible_tile?(x, y, groups)`: whether a tile can hold a patch.
- `pick_tiles(candidates)`: choice of the patch tiles.
- `build_patch(x, y, rate, attempts)`: kind and shiny roll of a patch.
- `weighted_kind`: draw of a regular patch kind.
- `apply_chain(patch, creature_to_select, result)`: forcing the chained species.
- `apply_shiny(patch, group, creature_to_select, lead)`: applying the shiny roll.
- `particle_file(system_tag, terrain_tag)`: image used by a patch.

Keep your overrides private as well.

---

## Public API reference

### Event script calls

| Method | Description |
|--------|-------------|
| `poke_radar` | Uses the radar and shows the failure message. Returns `true` / `false` |
| `poke_radar(force: true)` | Same, ignoring the battery |
| `poke_radar_chain` | Current chain length |
| `poke_radar_reset` | Breaks the chain and clears the patches |

### Module methods

| Method | Description |
|--------|-------------|
| `PokeRadar.configure(**settings)` | Override settings from `000 Config.rb` |
| `PokeRadar.use(force: false)` | Uses the radar without showing any message. Returns `true` / `false` |
| `PokeRadar.last_error` | `:no_party`, `:not_charged` or `:no_pokemon` after a failed `use` |
| `PokeRadar.last_error_message` | Text of the last failure, or `nil` |
| `PokeRadar.charged?` | Whether the battery is full |
| `PokeRadar.steps_before_charged` | Steps left before the battery is full |
| `PokeRadar.active?` | Whether some patches are in memory |
| `PokeRadar.patches` | Patches in memory (`PokeRadar::Patch`: `x`, `y`, `kind`, `shiny`) |
| `PokeRadar.chain` | Current chain length |
| `PokeRadar.chain_species` | `[db_symbol, form]` of the chained species, or `nil` |
| `PokeRadar.break_chain(reason)` | Breaks the chain and clears the patches |
| `PokeRadar.reset(reason, recharge: false)` | Full reset, optionally recharging the battery |

### Engine hooks

The plugin hooks into the engine with `prepend` only:

- `PFM::Wild_Battle#group_encounter_detected?`
- `PFM::Wild_Battle#setup`
- `PFM::Wild_Battle#select_creature`
- `Battle::Logic::BattleEndHandler#process`
- `Yuki::Particle_Object`: one new method, `update_poke_radar_flow`
- `Interpreter`: the event script calls above
- Scheduler: `Scheduler::EventTasks` (`begin_step`, `end_step`), `Scene_Map` (`on_update`, `on_warp_start`), `PFM::GameState.on_expand_global_variables`

---

## Technical notes and limitations

- **Nothing is saved.** The chain, the patches and the battery live in memory only. Loading a save starts with a clean, fully charged radar.
- **No graphics, no item.** You provide the particle images and the item (see [Graphics](#graphics) and [How to use](#how-to-use)).
- **Text compilation.** In debug, the CSV is read directly. Make sure `400000.<lang>.dat` files are generated before a release build (PSDK compiles them when it recompiles the texts).
- **Grass only.** Caves, water and other tags are not supported. Tool groups (rods, Rock Smash, Headbutt) are ignored.
- **On-screen area.** The area is the screen at the moment the radar is used. Leaving it forgets the patches.
- **Roaming Pokémon** take priority over a patch encounter, and break the chain when they do.
- **Compatible** with any plugin that also uses `prepend` on the same classes.

---

## Changelog

### 1.0.0.0

- Initial release.
