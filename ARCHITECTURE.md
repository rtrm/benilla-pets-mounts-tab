# Benilla Pets and Mounts Tab — Architecture

A client + server mod for vanilla World of Warcraft 1.12, built on forks of two open-source projects:

- **Client:** [`client/`](client) — a fork of [benilla](https://github.com/samwhosung/benilla), a from-scratch Rust + Bevy reimplementation of the 1.12.1 client. Users must supply their own legally-obtained 1.12.1 client data; no Blizzard assets are bundled here.
- **Server:** [`server/`](server) — a fork of [vMaNGOS](https://github.com/vmangos/core) (branch `development`), a vanilla-only (1.2–1.12) server core.

Both are real GitHub forks (not vendored copies), wired in as git submodules, so upstream history and the ability to pull/PR back (where in scope — see "Upstream divergence" below) are preserved.

## Clean-room policy

Turtle WoW's server source was leaked via an unauthorized security breach and is mirrored on GitHub by third parties. This project does not read, clone, reference, or derive any code from that leak. Where Turtle WoW's publicly-documented *player-facing behavior* inspired a feature (the companion-pet collection tab, see below), the implementation here is designed fresh from vanilla's own public client/server mechanics — this is ordinary game-design parity, the same way every private server reimplements Blizzard's own retail features, not a derivative of anyone else's code.

## Asset policy

v1 uses vanilla-only client assets (existing spell icons, existing creature models for companion critters). No MoP-era or other expansion assets are used. Every end user supplies their own 1.12.1 client; nothing proprietary is committed to this repo.

## Upstream divergence

benilla's own `AGENTS.md` scopes it as "a faithful, modern implementation of [1.12.1], not a place to get creative." The companion-pet collection tab (and anything pet-battle related later) is a later-expansion-era feature and intentionally out of scope for upstream benilla. This fork will diverge permanently on these features — they are not intended to be upstreamed. Bug fixes/compatibility work unrelated to new features may still be upstreamed where it makes sense.

## Milestone 1: Companion Pet Collection (character-bound)

**Goal:** rework vanity/companion pets from "permanent bag-item toggle" into a one-time-learn, character-bound collection accessible from a dedicated spellbook tab — the same underlying concept MoP's actual Pet Journal was later built from, scoped per-character (not account-wide) per explicit design choice.

### Current vanilla mechanic (baseline)

A companion item (e.g. "Cat Carrier (Black Tabby)", item 8491) has an on-use spell (e.g. spell 10675) with `spelltrigger_1 = 0` (infinite charges — the item is never consumed). That spell uses `SPELL_EFFECT_SUMMON_CRITTER` (effect 97, `Spell::EffectSummonCritter`), which toggles a non-combat mini-pet (`player->_SetMiniPet`) backed by a `creature_template` entry. Nothing is persisted between summons; the critter is purely re-spawned from the item each time.

### New mechanic

A two-spell pattern, both pure `spell_template` SQL rows — no client DBC patch required as long as existing icon IDs are reused:

1. **Learn spell** — the item's on-use spell changes to use `SPELL_EFFECT_LEARN_SPELL` (`Spell::EffectLearnSpell`), pointing at the Summon spell below. The item is consumed on use (standard "recipe" semantics already used elsewhere in vanilla — no new server mechanic needed).
2. **Summon spell** — a new spell, one per companion, keeping the existing `SPELL_EFFECT_SUMMON_CRITTER` effect pointing at the same `creature_template` entry the item already used. This is what gets permanently learned.

Reserved spell-ID range: companion Summon spells live in a dedicated custom ID block (e.g. `60000–60999`, exact range TBD when we touch `spell_template` for real) that the client-side tab logic (below) recognizes directly — this sidesteps needing a `skill_line_ability`/`SkillLine.dbc` entry for tab placement at all.

Persistence is automatically character-bound: learned spells land in `character_spell` (guid-keyed), and this core has no account-wide data table to accidentally share across characters.

### Client: synthetic "Pets" tab

benilla computes spellbook tab grouping **natively in Rust** (`crates/benilla-app/src/ui_spellbook.rs`), not via Lua/DBC lookup like stock FrameXML — tabs are built from the player's known-spell list cross-referenced against `SkillLine.dbc`. We add a synthetic tab that instead buckets any known spell whose ID falls in the reserved companion range above, bypassing `SkillLine.dbc` for this tab's membership entirely. This keeps the feature fully within the vanilla-only asset policy (no DBC patch).

Clicking an entry in this tab casts/summons it directly, exactly like every other spell in every other tab does — this needed no special-casing at all. Confirmed by reading the extracted real `SpellBookFrame.lua` (`SpellButton_OnClick`) directly: a plain click already calls `CastSpell` for any spell in the stock game; only a drag gesture or a shift-click calls `PickupSpell` to place it on the action bar. An earlier version of this doc assumed clicking needed a custom `PickupSpell` override to cast directly (and an implementation briefly shipped one) — that assumption was wrong, confirmed live: the override did nothing for the click case (already handled by stock `CastSpell` dispatch) while breaking the legitimate drag/shift-click pickup path, which is also why this tab's spells can be dragged to the action bar like any other, same as stock behavior, not a separate feature.

### Open items to verify during implementation

- How a summoned critter actually renders/follows the player client-side today, since no "companion" concept exists anywhere in benilla yet (likely generic NPC-follow rendering already works, since the client has no special-cased concept to be missing — but this needs confirming against real gameplay, not assumed).

## Build order (milestone 1)

1. Pick one existing companion (Black Tabby Cat) as the pilot. Add its Learn + Summon `spell_template` rows and repoint the item's on-use spell. Verify via server console/DB: using the item once teaches the Summon spell and consumes the item.
2. Add the synthetic "Pets" tab to the client spellbook UI. Confirm the learned Summon spell appears there after re-login.
3. Confirm clicking the Pets tab's entry summons/dismisses the critter with no bag item present (needs no implementation of its own — stock `SpellButton_OnClick` already casts on a plain click for any spell).
4. Verify the summoned critter renders and follows the player correctly; fix client-side if special-casing turns out to be needed.
5. Migrate the remaining stock companion pets to the new two-spell pattern (bulk, data-only SQL once the pattern is proven on one pet).

## Milestone 2: Mount Collection (character-bound)

**Goal:** the same treatment as milestone 1, for mounts: a one-time-learn, character-bound collection in its own spellbook tab, instead of a permanent bag-item toggle.

### Current vanilla mechanic (baseline)

A mount item (e.g. "Brown Horse Summoning", item 875) has an on-use spell (e.g. spell 458) applying `SPELL_AURA_MOUNTED` (aura 78, `effect1 = SPELL_EFFECT_APPLY_AURA`) plus a `SPELL_AURA_MOD_INCREASE_MOUNTED_SPEED` aura (32) on `effect2` for the speed bonus — real per-mount variation (speed tier, level requirement, duration, flavor text) lives on this spell, unlike a critter's largely-uniform summon effect. `spelltrigger_1 = 0` (infinite charges).

### New mechanic

The same Learn/Summon two-spell pattern as milestone 1, with one real difference: a mount's Summon spell is a **full clone of the original stock spell** (every column, not a generic template) — speed/level/duration/flavor text all carry real per-mount data a shared template would flatten. Only `entry` and `name` change. The Teach spell is the same generic `SPELL_EFFECT_LEARN_SPELL` wrapper as a companion pet's.

Reserved spell-ID range: `61000–61999`, a separate block from companion pets' `60000–60999` so the client can tell a Pets-tab entry from a Mounts-tab entry with a plain id-range check, rather than one combined range needing a second signal.

### Client: synthetic "Mounts" tab

Exactly `synthetic_spells.rs`/the Pets tab's pattern, mirrored in a sibling module (`synthetic_mounts.rs`) and a second sentinel tab key in `ui_spellbook.rs`'s `build_book`, rather than generalizing the two into one - the data shape differs enough (full-clone vs. template) that forcing one abstraction over both would cost more than the duplication does.

The real buff tooltip's "Increases speed by $s2%." line needs real data too, not just the text: `synthetic_mounts.rs`'s `Mount` carries each mount's own `description`/`aura_description` plus the `effect_base_points`/`effect_base_dice`/`effect_die_sides`/`effect_dice_per_level`/`effect_real_points_per_level`/`effect_apply_aura` values `benilla_formats::substitute` resolves `$s2`-style tokens from - all pulled from the already-cloned server row, not invented. Missed on the first pass (reported after live testing: the tooltip showed the mount's name but no speed line) because the synthetic catalog entry only carried enough fields for casting to work, not for every tooltip token a real stock spell's description might reference.

### Operational gotcha: `.reload spell_template` is not enough for a NEW spell id

`mangosd` precomputes a handful of per-spell classification bits exactly once, at boot (`SpellMgr::AssignInternalSpellFlags()`, called from `World.cpp`'s startup sequence, never from `SpellMgr::LoadSpells()` itself) - including `SPELL_INTERNAL_POSITIVE` and `SPELL_INTERNAL_DISMOUNT`, which `SpellEntry::IsPositiveSpell()` (the zero-argument overload specifically - there's a second, live-computing overload taking `caster`/`victim` that most of the live aura-application path correctly uses instead) reads as a cached bit rather than recomputing. `HandleCancelAuraOpcode` uses the cached, zero-arg overload.

Consequence, found the hard way: after adding new spell ids via a migration and `.reload spell_template`, those *ids'* right-click-to-cancel on their own aura silently fails forever - the live computation (used for slotting the aura as a buff and setting its wire-level cancelable flag) says positive, so the aura shows up fine and the client correctly tries to cancel it, but the opcode handler's cached check says otherwise and refuses, with no error shown anywhere. Pre-existing spell ids (present at the server's last real boot) are unaffected, which is what made this look id-specific rather than general at first. A full `mangosd` restart (not a hot `.reload`) fixes every affected id at once, since it re-runs `AssignInternalSpellFlags()` against whatever is in the DB *now*. Any future spell-table migration that adds new ids needs a restart, not just a reload, before those ids' auras are fully usable - worth remembering before spending another round of live testing chasing what looks like a data bug.

### Client gotcha: a synthetic Teach spell needs its own learned-spell registration

`SpellCatalog::learned_spell` (the hop a trainer window's wire id already resolves through) is built from `Spell.dbc`'s `EffectTriggerSpell` columns at load time, so it only ever knows about real DBC-backed spells. Our synthetic Teach spells (the reserved custom-id ranges, `synthetic_spells.rs`/`synthetic_mounts.rs`) have no DBC row to read that hop from - a Teach spell's catalog entry is visual-only (`TEACH_VISUAL`, no effect data), by design, since it's never learned itself. The item tooltip's "Already known" line (`ItemTemplateView::taught_spell`, resolved in `ui_items/feed.rs` from the item's on-use spell) needs that hop anyway, to follow a Teach item to the Summon spell it actually grants - so each `synthetic_*.rs`'s `install()` separately registers its own `(teach_id, summon_id)` table via `SpellCatalog::insert_learned_spell`, read off the matching SQL migration's `effectTriggerSpell1` column rather than invented. Missed on the first pass: the pre-existing `ITEM_SPELL_KNOWN` check in `render.rs` was gated on `spelltrigger == 6`, a value the real `ItemSpelltriggerType` enum (confirmed both server-side and in the client's own wire-parsing doc comment) does not have in 1.12 at all - 0/1/2 (`ON_USE`/`ON_EQUIP`/`CHANCE_ON_HIT`) are the only values that ever arrive over the wire, so that check could never have fired, for our items or for any real stock recipe item either.

## Scope boundary: pet battles are a fork, not a milestone here

This repo's scope ends at milestones 1-2: the pets and mounts collection tabs, finished and verified live. A pet battle system (a pet-battle icon above wild critters, right-click to enter battle, battle GUI, turn-based combat engine, abilities, camera transition) is a substantially larger effort with its own design space - it belongs in a fork built on top of this repo's foundation, not as a further milestone added in place here. Keeping this repo's scope fixed is deliberate: it stays simple, focused and finished, rather than becoming an ever-growing base for unrelated systems.
