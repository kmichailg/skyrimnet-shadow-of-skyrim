# Shadow of Skyrim - Defeat Awareness

A [SkyrimNet](https://github.com/MinLL/SkyrimNet-GamePlugin) content plugin that tells NPCs
when the player has been defeated.

## The problem

[Shadow of Skyrim](https://www.nexusmods.com/skyrimspecialedition/mods/57008) replaces death with
defeat. You are not killed: you are struck down, your gear is taken by whoever beat you, and you
wake somewhere else.

SkyrimNet has no bleedout handling of its own, so none of that reaches dialogue. To every NPC the
player simply vanished mid-fight and turned up later with nothing, and a language model handed that
gap will fill it in - usually by congratulating you on a fight you lost, or by inventing a clean
escape that never happened.

## What this plugin does

It adds one block to the speaking NPC's own prompt, covering two states:

**The player is bleeding out right now.** NPCs are told the player is on the ground and cannot
answer, and that the fight is not yet decided - whoever is still standing decides what happens next.

**The party has lost.** NPCs are told the player was struck down, that what followed was not a
victory, that gear or gold was taken, and that they should not pretend otherwise or invent a win.

The block is standing state: it applies for as long as the defeat is unresolved, which is exactly
how long the player really does carry it.

## What it deliberately does not do

- It does not narrate, trigger dialogue, or make extra LLM calls. It is a prompt block, nothing more.
- It does not claim the player was relocated *to a specific place*. It says either they were dragged
  clear or they woke elsewhere, because that distinction is not reliably observable from a prompt.
- It does not name the nemesis. `get_quest_alias_actor` can read the alias, but calling it on an
  empty alias risks a template error, and a prompt that throws loses the whole block.

## Requirements

- **SkyrimNet Beta 25 (0.25.0) or newer**, configured and working.
- **Shadow of Skyrim - Nemesis and Alternative Death System** (`Shadow of Skyrim.esp`).

Nothing else. No SKSE plugin, no ESP of its own, no save data. Removing it leaves no trace.

## Installation

Copy `manifest.json` and `prompts/` into a mod folder, so the result is:

```
<your mod>/
├── manifest.json
└── prompts/submodules/user_final_instructions/0205_player_defeat.prompt
```

That folder is a SkyrimNet *external* content layer. SkyrimNet discovers it on the next boot and
registers it automatically; look for `newly registered external: kmichailg.shadow-of-skyrim-defeat`
in `SkyrimNet.log`.

Enable it after SkyrimNet in your mod manager.

## How it works

SkyrimNet composes an NPC's prompt from numbered components under
`prompts/submodules/user_final_instructions/`, rendered in filename order. The base game ships
`0200_combat_status.prompt` and `0500_response_format.prompt`; this plugin adds `0205_` between
them, so the defeat block lands right after the combat status block.

Only a file at the *same relative path* as a SkyrimNet base prompt replaces it. This one is a new
path, so it adds a block and overrides nothing.

Three decorators, all built into SkyrimNet:

| Decorator | Use |
|---|---|
| `is_plugin_loaded("Shadow of Skyrim.esp")` | Keeps the whole file inert when Shadow of Skyrim is absent |
| `is_knocked_down(player.UUID)` | True while an actor is bleeding out or in an essential-down state |
| `is_quest_active("_<n>NemesisTrackerQuest")` | True while an unresolved nemesis exists - i.e. the party lost |

The five tracker quests are Shadow of Skyrim's nemesis slots. Any one being active means a defeat
is still outstanding.

## Licence

Content is released under the [SkyrimNet Plugin License](https://github.com/MinLL/SkyrimNet-Plugins/blob/main/LICENSE.md).

## Development

- **[DESIGN.md](DESIGN.md)** — the problem, every hook that was investigated, why most were
  rejected, and the evidence behind the three decorators this plugin uses.
- **[AGENTS.md](AGENTS.md)** — hard rules for changing this repo: layering, decorator verification,
  Inja balance, and how to publish to the hub.
