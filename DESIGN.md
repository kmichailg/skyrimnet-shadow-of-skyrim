# Design — Shadow of Skyrim: Defeat Awareness

A SkyrimNet prompt plugin that makes the player's defeats legible to NPCs.

This document records the problem, the hooks that were investigated, why most of them were
rejected, and why the two that survived are the right ones. It is written so a future maintainer
can re-derive the design rather than trust it.

---

## 1. The problem

[Shadow of Skyrim](https://www.nexusmods.com/skyrimspecialedition/mods/57008) replaces death with
defeat. The player is not killed. They are struck down, their gear and gold are taken by whoever
beat them, and they wake somewhere else with a nemesis to hunt.

SkyrimNet sees none of it.

The consequence is not cosmetic. SkyrimNet composes an NPC's prompt from what it can observe, and a
party wipe produces almost no observable trace: no death event, no player relocation it attributes
to a defeat, no dialogue. A language model handed that gap does the natural thing — it fills it in.
The usual failure is a follower congratulating the player on a fight they lost, or inventing a clean
escape that never happened.

### Evidence the defeat is invisible

| Check | Result |
|---|---|
| `Kill` on the player in `_ShadowOfSkyrimScript.pex` | 0 (the one `Kill` call targets the nemesis) |
| `Resurrect` anywhere in SOS scripts | 0 |
| `SkyrimNet.log` references to `shadow of skyrim` / `nemesis` / `defeat` / `resurrect` | 0 |
| `death` events in the SkyrimNet DB | 39, of which 0 mention the player |
| Base prompts handling bleedout as a mechanic | 0 (175 term matches were all lore prose) |

SOS is silent by design: it has no SkyrimNet integration, no `SendModEvent`, and no `SkyrimNet`
symbol anywhere in its 53 compiled scripts.

---

## 2. Hooks investigated

### 2.1 `enter_bleedout` — it fires, but it cannot decide anything

`EnterBleedoutEventHandler` is a `BSTEventSink<TESEnterBleedoutEvent>`, an engine-level sink rather
than a Papyrus listener. It registers on every boot (`SkyrimNet.log` L55, L610), and it does **not**
filter out the player (`Skipping player` = 0, `skip the player` = 0, `SkipPlayer` = 0 in the DLL).

So SOS's real player bleedout does reach SkyrimNet. But the payload is the actor's name and nothing
else, and the event fires identically for "the party was wiped and the player was carried off" and
"the player was revived on the spot". It is a trigger, not a verdict.

> A correction worth recording: `enter_bleedout` is declared `persistent: false` in `Events.yaml`,
> so it is never written to the `events` table. The earlier inference "0 database rows therefore it
> never fires" was invalid. The log is the authority here, not the DB.

### 2.2 Routes that were rejected

**`get_script_property` — rejected.** The obvious hook is SOS's own state: `abPlayerDefeated`,
`abPlayerRevived`, `akSavior`, `bRevivedByFollower`. All four appear in `_ShadowOfSkyrimScript.pex`.
All four are function **locals**, not properties, so `get_script_property` can never read them.
This was proven rather than assumed: `_ShadowOfSkyrimScript` is bound to quest `_ShadowOfSkyrimQuest`
(`00080A:Shadow of Skyrim.esp`), and enumerating that quest's declared properties returns about 48
entries — `kLastEnemy`, `kNTQuest`, `sCauseOfDefeat`, `bIsValidNemesis`, `Alias_Nemesis` and so on —
and none of the four appear.

**Quest stages — rejected.** `_ShadowOfSkyrimQuest` has **0 stages**. There is no stage to poll.

**Globals — rejected.** SOS ships **0** `GLOB` records. There is no MCM global to read either.

**`location_change` — rejected as the primary signal.** It has no player filter and is driven by
cell attach/detach, so a relocation does surface. But it also fires for every door, every fast
travel, and every cell boundary. Using it as the defeat signal means correlating it against a
defeat that was never itself recorded. Brittle, and noisy in normal play.

**`get_quest_alias_actor` for the nemesis name — deferred.** `get_quest_alias_actor(questEditorId,
aliasName)` works, and `_1NemesisTrackerQuest` does carry a `Nemesis` alias. But calling it on an
empty alias risks a template error, and in Inja a throwing call loses the whole enclosing block.
Naming the nemesis is not worth the fragility.

---

## 3. The hooks chosen

Three decorators, all built into SkyrimNet, no plugin of our own.

### 3.1 `is_plugin_loaded("Shadow of Skyrim.esp")`

Exactly one string argument, compared case-insensitively. Used as the outermost guard so the file
is completely inert on a load order without SOS. This is the difference between a plugin and a
liability: without the guard, every user who installs it without SOS gets a prompt block claiming
they were defeated.

### 3.2 `is_knocked_down(player.UUID)`

> "Check if an actor is knocked down (bleeding out or essential down) … This includes both mortal
> NPCs in bleedout and essential NPCs who have been downed."

This is the live state, and it is precisely the window in which the user's second outcome is still
possible. While the player is down, the fight is genuinely undecided — a follower can still kill the
attacker, and the player can still be revived where they fell. The block says exactly that, and
nothing more.

### 3.3 `is_quest_active("_<n>NemesisTrackerQuest")`

SOS keeps five nemesis slots as `_1NemesisTrackerQuest` … `_5NemesisTrackerQuest`
(`000800` … `00081B:Shadow of Skyrim.esp`). Each has `Flags = 0` — not start-game enabled — so they
are started by script only when a nemesis is actually assigned. A tracker is shut down at stage 200
once its nemesis is dead.

So *any tracker active* means: a defeat has been recorded, a nemesis exists, and the debt is
unresolved. That is the user's first outcome, and it is a far better signal than a cell change
because it is a semantic state rather than a movement.

The block ORs all five.

### Two signals, because "active" is ambiguous

`is_quest_active` is documented as "active in the player's quest log", and SkyrimNet's quest cache
tracks `InLog` / `HasJournal` per quest — so a quest that is running but never enters the journal may
never qualify as active. A live session logged `Cached 0 active quests`, which would make the
defeat branch dead code if the active flag were the only signal. `get_quest_stage(editorID, false)`
reads the stage regardless of active status, and the stage scale is unambiguous: 0 is never started
and 200 is shut down, so `0 < stage < 200` means the defeat was recorded and is still unresolved. The
two signals are OR'd, so neither hypothesis has to be right — if the journal-scoped reading fails, the
stage reading still carries the block.

---

## 4. Why a prompt block, not a trigger

A trigger on `enter_bleedout` or `location_change` would have needed a usable payload, a cooldown,
and an LLM call — and would still have had to guess the outcome. The prompt block is:

- **free** — no extra LLM call, it is part of the prompt already being assembled;
- **speaker-scoped** — it renders into the speaker's own prompt, so it costs nothing for NPCs who
  are not talking;
- **standing state** — it stays true for exactly as long as the condition does, so it cannot go
  stale the way a one-shot narration does;
- **unspammable** — there is no per-event firing to rate-limit.

---

## 5. Placement and layering

SkyrimNet renders `prompts/submodules/user_final_instructions/` in filename order. The base game
ships `0200_combat_status.prompt` and `0500_response_format.prompt`. This plugin adds `0205_`, so
the defeat block lands immediately after the combat status block — the natural home for a fact about
combat state.

The critical property, per the hub authoring guide: **only a file at the same relative path as a
SkyrimNet base prompt replaces it.** `0205_player_defeat.prompt` is a new path, so it *adds* a block
and overrides nothing. This was verified against the hub's `docs/AUTHORING.md` before writing, and
against the live base library before choosing the number.

Line 1 must remain the `render_mode` guard. Without it the block renders into render modes that have
no `player` object.

---

## 6. What was deliberately left out

**No relocation claim.** The block says "either dragged clear or woke somewhere else" rather than
asserting a teleport. `location_change` can see a cell change, but cannot see *why*, and asserting
"you were carried to X" from a cell change would be wrong every time the player simply walked
through a door afterwards.

**No nemesis name.** See 2.2 — the alias read is fragile.

**No `allowNPCReaction` change.** `enter_bleedout` has `allowNPCReaction: false` in `Events.yaml`.
Flipping it would make NPCs react to the collapse, but it opens the same event-leak class that had
to be closed for `hit`, and would need its own `shortLivedEnabled` decision. Out of scope for v1.

**No trigger, no diary entry, no memory.** All of those cost LLM calls and would duplicate what the
prompt block already states.

---

## 7. Known limitations

- **"Just now" is not observable.** A nemesis tracker that has been active for six hours reads the
  same as one assigned a minute ago. This is accepted: the player genuinely does still carry the
  defeat until the nemesis dies, so the standing statement is not false.
- **A stuck tracker would persist the block.** If SOS ever left a tracker active after its nemesis
  died, the block would keep rendering. It only affects prompt text, and SOS does shut the tracker
  at stage 200.
- **The two outcomes are described, not distinguished.** The block covers both cases in one
  sentence pair because the discriminator is not reliably observable from a prompt. This is a
  deliberate honesty choice, not an oversight.
- **Not verified in game at the time of writing.** See the verification plan below.

---

## 8. Verification plan

1. **Boot.** `SkyrimNet.log` must contain
   `newly registered external: kmichailg.shadow-of-skyrim-defeat`, and the content snapshot path
   count must rise by one.
2. **Guard.** With SOS disabled, the block must not render. This is the highest-value check: it is
   the failure mode that would affect every user who does not own SOS.
3. **Down state.** Take a defeat and confirm the `IS DOWN` block renders while the player is on the
   ground.
4. **Lost state.** After respawn, confirm `WAS DEFEATED` renders, and that a follower acknowledges
   the loss instead of inventing a win.
5. **No render errors.** The log must show no Inja error for `0205_player_defeat.prompt`.

---

## 9. Decorator reference

All four are built into SkyrimNet; none requires an SKSE plugin.

| Decorator | Signature | Behaviour |
|---|---|---|
| `is_plugin_loaded` | `(pluginName: string) -> bool` | Case-insensitive load-order check |
| `is_knocked_down` | `(actorUUID) -> bool` | Bleeding out or essential-down |
| `is_quest_active` | `(questId: number\|string) -> bool` | Quest active in the player's log |
| `get_quest_stage` | `(questId: number\|string, onlyWhileActive: bool) -> int` | Stage number; -1 if not found, or if inactive and onlyWhileActive is true |

Decorator names, signatures and error strings were read out of `SkyrimNet.dll` directly rather than
from documentation, because a misspelled decorator is a silent no-op in a prompt.
