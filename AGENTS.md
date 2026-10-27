# AGENTS.md

Guidance for anyone — human or agent — changing this repo.

## What this repo is

A SkyrimNet **content plugin**: one prompt file plus a manifest. There is no SKSE plugin, no ESP, no
Papyrus, and no save data. If a change would require any of those, it belongs in a different repo.

## Layout

```
manifest.json                                              # hub metadata; required
prompts/submodules/user_final_instructions/0205_...prompt   # the entire product
README.md                                                  # user-facing
DESIGN.md                                                  # why the hooks are what they are
AGENTS.md                                                  # this file
```

The prompt's **path is load-bearing**. See "Layering" below.

## Hard rules

1. **Line 1 of the prompt must stay the `render_mode` guard.**
   `{% if render_mode == "full" or render_mode == "thoughts" or render_mode == "transform" %}`
   Other render modes have no `player` object and the block would throw.

2. **Never place a file at a path that already exists in SkyrimNet's base library** unless you
   intend to *replace* that base prompt for every user. A new path adds a block; the same path
   replaces one. This plugin adds `0205_`, between the base `0200_combat_status` and
   `0500_response_format`, and replaces nothing.

3. **Every decorator must exist in the shipped SkyrimNet build.** A misspelled decorator is a silent
   no-op, or a render error that loses the whole block. Do not guess names. The reliable way to check
   is to grep the DLL:

   ```powershell
   $t = [IO.File]::ReadAllBytes("<path>\SkyrimNet.dll")
   $s = [Text.Encoding]::Latin1.GetString($t)
   $s.Contains("is_knocked_down")
   ```

   A decorator that appears zero times does not exist, whatever the docs say.

4. **Inja must balance.** Count `{%` against `%}` and `{{` against `}}`. Both must match.

5. **UTF-8, no BOM.** A BOM at the head of a `.prompt` breaks the first-line guard.

6. **Guard the whole file on the mod it depends on.** `is_plugin_loaded("Shadow of Skyrim.esp")` is
   the outermost conditional and must stay that way. Without it, everyone who installs this without
   SOS is told they were defeated.

7. **Bump `version` and write a real `changelog` on every publish.** The hub keeps prior changelog
   entries from git history; describe only the current version, do not paste the history in.

## Layering

SkyrimNet assembles `prompts/submodules/user_final_instructions/` in filename order. The number
prefix is the insertion point. Gaps of 10 are intentional, so a new component can be slotted without
renumbering.

The block is speaker-scoped: it renders into the speaking NPC's own prompt. It is standing state, not
an event, so it needs no cooldown and cannot spam.

## Local install

A SkyrimNet *external* content layer lives inside SkyrimNet's content root, in a directory named for
its manifest `id`:

```
<mod folder>/SKSE/Plugins/SkyrimNet/external/kmichailg.shadow-of-skyrim-defeat/
├── manifest.json
└── prompts/submodules/user_final_instructions/0205_player_defeat.prompt
```

The repo root holds the layer's *contents*, not the layer itself. Copy them into a directory named
`kmichailg.shadow-of-skyrim-defeat` under `SKSE/Plugins/SkyrimNet/external/`.

Two rules, both enforced by the engine:

- The directory name must equal the manifest `id`. A mismatch is ignored with
  `directory name is not its manifest id`.
- The content must be under `Data/SKSE/Plugins/SkyrimNet`, because that is the only place discovery
  scans. A `manifest.json` at the mod folder root deploys to `Data/manifest.json` and is never seen.

Enable it after SkyrimNet and boot; look for
`newly registered external: kmichailg.shadow-of-skyrim-defeat` in `SkyrimNet.log`.

## Publishing to the hub

The hub repo is [`MinLL/SkyrimNet-Plugins`](https://github.com/MinLL/SkyrimNet-Plugins). Plugins live
at `plugins/{github-user}/{plugin-slug}/`. For this one that is
`plugins/kmichailg/shadow-of-skyrim-defeat/`.

The hub's own dashboard has a **Publish** page that forks, writes the files, and opens the pull
request for you; that is the path of least resistance. By hand, fork the repo, add the directory
with `manifest.json` and `prompts/`, and open a PR.

Before submitting:

- `manifest.json` `id` must be `{author}.{slug}` and match the directory.
- `mods` must list every load-order mod the content depends on — here, `Shadow of Skyrim.esp`.
- `nsfw` must be accurate. This plugin has no sexual content.
- A submission with no `actions/` is reviewed automatically; a submission with actions needs human
  review. This repo has no actions and should stay that way.

## Do not

- Do not add a trigger, diary entry, or memory generation without a strong reason. Each costs an LLM
  call per fire; the prompt block costs none.
- Do not name the nemesis via `get_quest_alias_actor` unless you have confirmed the alias is never
  empty. A throwing decorator loses the entire block.
- Do not assert a specific relocation location. A cell change is not proof of a teleport.
- Do not hardcode a player name, a save, or a load order.
