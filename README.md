# Benilla Pets and Mounts Tab

A client + server mod for vanilla World of Warcraft 1.12, adding character-bound companion pet and mount collections: a one-time-learn, per-character collection in its own spellbook tab, replacing the old permanent-bag-item-toggle model.

## Repo layout

- [`client/`](client) — fork of [benilla](https://github.com/samwhosung/benilla) (Rust/Bevy 1.12.1 client reimplementation), git submodule, branch `main`
- [`server/`](server) — fork of [vMaNGOS](https://github.com/vmangos/core) (vanilla 1.2–1.12 server core), git submodule, branch `development`

## Getting started

Clone with submodules:

```
git clone --recurse-submodules https://github.com/rtrm/benilla-pets-mounts-tab.git
```

Build instructions for the client and server live in their own READMEs ([client/README.md](client/README.md), [server/README.md](server/README.md)) — this repo only tracks the integration work (patches, SQL, docs) on top of them.
