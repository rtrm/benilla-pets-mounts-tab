# Benilla Pets and Mounts Tab

A client + server mod for vanilla World of Warcraft 1.12, adding character-bound companion pet and mount collections: a one-time-learn, per-character collection in its own spellbook tab, replacing the old permanent-bag-item-toggle model.

This repo's scope ends here, at the pets/mounts tab. Pet battles (or anything else built on top of the collection) are intentionally out of scope - fork this repo rather than growing it, so this one stays simple and finished.

See [ARCHITECTURE.md](ARCHITECTURE.md) for the full design, clean-room/asset policy, and build order.

## Status

Both milestones are complete and verified live.

Milestone 1 — Companion Pet Collection (character-bound):

- [x] 1. Pilot one companion (Black Tabby Cat) through the new Learn/Summon two-spell pattern on the server (verified live: learn, consume, cast, sound — see `server/sql/migrations/20261004120000_world.sql`)
- [x] 2. Add a synthetic "Pets" tab to the client spellbook UI
- [x] 3. Wire the Pets tab to cast directly instead of pick-up-for-actionbar
- [x] 4. Verify summoned-critter client rendering/follow behavior (confirmed live: summons, follows, dismisses on recast)
- [x] 5. Migrate remaining stock companion pets to the new pattern (all 69 real vanilla companion items — see `server/sql/migrations/20261005230630_world.sql`)

Milestone 2 — Mount Collection (character-bound), the same pattern applied to mounts:

- [x] All 112 real vanilla mount items migrated to Learn/Summon (a full clone of each stock mount spell, not a template — see `server/sql/migrations/20261006003000_world.sql`), a synthetic "Mounts" tab added to the client, click-to-cast wired, and the trainer-style learn visual carried over from milestone 1 — confirmed live (summon, dismount via right-click-the-buff, buff speed tooltip, drag-to-hotbar).

Finishing touches, both confirmed live:

- [x] The item tooltip's "Already known" line follows a Teach spell to the Summon spell it actually grants, instead of a dead `spelltrigger == 6` check that could never fire on real 1.12 data.
- [x] The synthetic Pets/Mounts tabs are pinned after every real skill-line tab, the same way the General tab is pinned first, instead of sorting alphabetically alongside spec tabs.

Pet battles proper (icon, right-click, GUI, abilities, camera) are explicitly out of scope for this repo - see the note above.

## Repo layout

- [`client/`](client) — fork of [benilla](https://github.com/samwhosung/benilla) (Rust/Bevy 1.12.1 client reimplementation), git submodule, branch `main`
- [`server/`](server) — fork of [vMaNGOS](https://github.com/vmangos/core) (vanilla 1.2–1.12 server core), git submodule, branch `development`

Both are real forks under this account, not vendored copies, so upstream history is preserved. See ARCHITECTURE.md's "Upstream divergence" note on what is/isn't intended to go back upstream.

## Getting started

Clone with submodules:

```
git clone --recurse-submodules https://github.com/rtrm/benilla-pets-mounts-tab.git
```

Build instructions for the client and server live in their own READMEs ([client/README.md](client/README.md), [server/README.md](server/README.md)) — this repo only tracks the integration work (patches, SQL, docs) on top of them.
