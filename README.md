# Albion-Wolf-Database

Single source of truth for Albion Online item data, extracted from the
official `ao-bin-dumps` game files. The Albion Wolf site (`assets/data/`)
is regenerated from this database — never the other way round.

## Contents

| Path | What |
|---|---|
| `items/<Group>/<Sub>/<file>.json` | 5,615 shards, 10,912 item records |
| `manifest.json` | every item: id, names (ar/en), category, slot, tier, shard path |
| `id-index.json` | id → [shard path, index in shard] |
| `category-index.json` | category → count (87 categories) |
| `category-map.json` | group → [{value, count}] (20 groups) |
| `filters.json` | groups / subcategories / categories / slots / tiers / enchantments / shards |
| `repository-manifest.json` | totals + per-category counts |
| `source-files.json` | one entry per shard on disk (5,615) |
| `audit-report.json` | last extraction run statistics |
| `source-dictionaries.json` | source dictionaries metadata |
| `schema.json` | schema version + principle |

## Shard record fields

Each record keeps the **full raw dump** (`raw`) plus structured views:

- `name_en/ar/de/fr/es/ru` + `localized_all_names` (15 locales, incl. `TR-TR`)
- `descriptions` (`description_<lang>`, site langs en/ar/de/fr/es/ru/tr)
- `category / slot / two_handed / tier / enchantment / is_artifact`
- `attributes` (raw `@`-attributes), `item_stats`, `combat_stats`
- `crafting_requirements` + `crafting_full` (silver/time/focus/resources)
- `spells` (`all/actives/passives`, slots, tags). Items whose dump entry
  is `craftingspelllist: {"@reference": "<id>"}` carry the **resolved**
  spell lists plus `spells.resolved_from` with the target id.
- `enchantments` (`1..4`: item_power, durability, crafting focus/time,
  crafting + upgrade resources), `shop`, `integration_ids`, `source`

Game-client audio + FX/VFX data is deliberately **excluded**: no
`socket_audio` / `projectile` fields, no `uicraftsound*` /
`vfxAddonKeyword` / `fxbonename` / `fxboneoffset` attributes, and the
stored `raw` copy drops the `AudioInfo`, `SocketPreset`, `attackvfx`,
`FootStepVfxPreset`, `AssetVfxPreset`, `projectile` blocks, matching
`@`-attributes, and `attackvfx` nodes inside `attackvariations`
(damage factors are kept). `uisprite*` (inventory icon ids) are kept:
icon references, not audio/FX.

## Generation

`extractor.py` (v2 pipeline): `ao-bin-dumps` → shards + indexes + site
`items.json`/`item-details.json` name fills. Safe to re-run: existing
shards are merged, never blindly overwritten; `--stage` validates only.

`tools/build-wolf-db.cjs` (site repo): regenerates `craft-gear.json`,
`craft-utility.json`, `craft-stable.json`, `item-details.json`
(names/descriptions/stats/spells/enchants) and `items.json` names —
all from this database.

`tools/build-item-classes.cjs` (site repo): extends
`item-classes.json` to every item code using the classifier extracted
at runtime from `assets/js/items.js`. Existing entries are never
rewritten (they carry curated fixes, e.g. Farming subs).

Classification follows the Albion Wolf filter tree (`filter.txt`):
Weapons / Chest- / Head- / Foot-Armor / Off Hands / Caps (Cape +
14 factions) / Bags (Bags + Satchels of Insight) / Mount / Consumable
(Food, Potions, Tomes, Repair Powder, Fireworks) / Gathering Equipment
/ Crafting / Artifact / Farming (Farm, Herb Garden, Pasture, Kennel,
Farming Products) / Furniture / Vanity (Mounts, Weapons, armor pieces,
Off Hands, Capes, Kill Emotes) / Other (Guilds, Laborers, Tokens,
Luxury Goods [empty], Map, Hardcore Expeditions, Quest Items).
Vanity items are browsable on the site (no longer hidden). In the DB
`filters.json` the `Satchels_Of_Insight` group is merged into `Bags`;
manifest/shard taxonomy is unchanged (file paths depend on it).

## Notes

- Turkish (`tr`) names come from `localized_all_names['TR-TR']`.
- Per-item crafting/refining fame is **not** in the official dump
  (verified: recipes sum to 0 vs community tables). Crafting fame shown
  on the site comes from the community table (`fame.json`,
  albiononlinegrind.com) with ×2^enchant, ×1.5 premium. The table was
  re-verified cell-by-cell against its source (base ×2^enchant exact);
  third-party calculators may differ (different methodology).
- Gathering fame per resource unit lives in `resource-fame.json`
  (site repo), extracted from dump `@famevalue` (raw + `_LEVEL`
  variants). The refining page shows it as embodied fame per refined
  unit, resolved recursively through lower-tier inputs
  (e.g. T5 bar = 3× T5 ore + embodied T4 bar = 90).
- Dump gaps (source-side, not fixable here): shapeshifter weapon recipes
  (only artefacts carry `craftingrequirements`), 550 unlocalized
  test/internal records, per-item refining fame.
- User-deleted junk (2026-09-26, 260 ids): TRASH T1–T8, *NONTRADABLE* /
  *NON_TRADABLE* / *UNTRADEABLE*, *ADC*, *COPY*, *LOOTCHEST_COMMUNITY*,
  consumable/arena/dungeon leftovers and listed tokens. The patterns live
  in `extractor.py` (`USER_DELETE_*` + `is_excluded`) so future runs never
  resurrect them; the same codes were removed from the site data files.
