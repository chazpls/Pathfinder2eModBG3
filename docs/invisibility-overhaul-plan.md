# Invisibility Overhaul

Status: `bugfix/undetected`, rebuilt on upstream's invisibility framework (730d01ec). Not yet tested in-game. The old stash/swap prototype (`DB_PF2_InvisCombatSwap`) is gone, and so is the Sanctuary rework, which lives on another branch.

## Rules it follows
| State | Who can target | Flat check | On screen |
|---|---|---|---|
| Invisible, out of combat | nobody | none | engine invisibility (vanilla stealth past guards) |
| Invisible, in combat | everyone | **DC 11** (Hidden); **DC 5** if the attacker has See the Unseen or Blindsight | drawn, with the invisibility shimmer |
| Invisible + Faerie Fire | everyone | DC 5 (Concealed) | drawn |
| Undetected | only from Sneak (existing stealth system), nothing new here | | |

Basic invisibility ends when the creature casts a harmful spell or makes an attack. Greater invisibility doesn't end that way.

## How it works
- **Markers.** Every invisibility source is a BOOST status in `SG_Invisible` with `Tag(INVISIBLE_APPLY)` and `UnlockInterrupt(HiddenReaction)`, built on `INVISIBILITY_APPLY` / `INVISIBILITY_APPLY_GREATER`. Each marker keeps its own duration and end conditions. Basic ones end on `OnSpellCast` when `(HasSpellFlag(SpellFlags.Harmful) or HasStringInSpellRoll('Attack')) and not HasSpellFlag(SpellFlags.Invisible)`.
  - Vanilla INVISIBLE-type sources are overridden as markers: `INVISIBILITY`, `GREATER_INVISIBILITY`, `POTION_OF_INVISIBILITY`, `SHROUD_SELF`, `CLOAK_OF_SHADOWS(_MONK)`, `UMBRAL_CLOAK`, `INVISIBILITY_PANTHER`, `INVISIBILITY_SHADARKAI_GLOOMWEAVER`, `ONE_WITH_SHADOWS`, `SUPREME_SNEAK`. `MISTY_ESCAPE_APPLY` carries Misty Escape's free Misty Step and its own end conditions.
  - Left as engine INVISIBLE statuses: `INVISIBLE` (story scripts use it by name), `FADE_APPEAR` (technical), `SHADOWBLEND`, `HIDE_IN_PLAIN_SIGHT`, `INVISIBLE_MAGEHAND`, `INVOKE_DUPLICITY_TECHNICAL`, `UNDETECTED_NPC`, `MISTY_ESCAPE_INVISIBLE` (now unused).
- **`PF2e_Invisibility.txt`** picks the visible state. It runs whenever an `SG_Invisible` status is applied or removed, Faerie Fire or `SEE_INVISIBILITY_2E_REVEAL` is applied or removed, or an invisible creature enters or leaves combat. It checks `HasActiveStatusWithGroup(SG_Invisible)`, not the tag, so one marker ending doesn't cancel another. Exactly one branch applies:
  1. no marker → remove `INVISIBLE`, `INVISIBLE_FAERIE_FIRE`, `INVISIBLE_COMBAT`
  2. Faerie Fire → `INVISIBLE_FAERIE_FIRE`
  3. in combat → `INVISIBLE_COMBAT` (drawn stand-in; a See the Unseen reveal doesn't matter here)
  4. out of combat, revealed by See the Unseen → `INVISIBLE_FAERIE_FIRE`
  5. out of combat → `INVISIBLE`
- **Hidden.** `INVISIBLE_COMBAT` applies `HIDE_FREE_INVIS` (players: `HIDDEN_INVISIBLE`, plus `HIDDEN_FROM` on enemies without See the Unseen or Blindsight) or `HIDE_FREE_INVIS_NPC` (`HIDDEN_INVISIBLE_NPC`) when it lands, and again each turn through `InvisHideFree`. Neither status adds a sneaking penalty, and `Shout_Hide` is free while `HIDDEN_INVISIBLE` is up.
- **Flat checks.** `IsHiddenFromAttacker(observer, attacker)` (`PF2_Concealment.khn`) drives `HiddenReaction` (DC 11, scope Far) and excludes `FlatCheckReaction` (DC 5), so only one check fires. `IsConcealedTo` counts engine `INVISIBLE` or any `SG_Invisible` marker.
- `PF2e_Darkness.txt`: the `INVISIBLE` tag rule only re-applies `INVISIBLE` out of combat when no marker is on. All `SEE_INVISIBILITY` checks now read `SEE_INVISIBILITY_2E`.

## Test cases
| # | Setup | Expected |
|---|---|---|
| 1 | Cast Invisibility out of combat | Vanilla invisibility; guards don't notice |
| 2 | Invisible party member, combat starts | Drawn with shimmer; `INVISIBLE_COMBAT` + `HIDDEN_INVISIBLE`; enemies roll DC 11 on single-target actions; Fireball hits normally |
| 3 | Same, enemy has See the Unseen | That enemy rolls DC 5; others still DC 11 |
| 4 | Basic invisibility, then Strike or a harmful spell | Marker ends, stand-in and Hidden go away |
| 5 | Greater invisibility, attack | Stays invisible and Hidden |
| 6 | Combat ends with turns left on the marker | Back to engine invisibility for the remaining duration |
| 7 | Faerie Fire on an invisible creature | `INVISIBLE_FAERIE_FIRE`; DC 5 for everyone |
| 8 | Two markers (e.g. potion + spell), one ends | Still invisible |
| 9 | Enemy Shroud Self / Imp Invisibility in combat | Drawn, Hidden (DC 11), AI still uses it |
| 10 | Misty Escape | Cloaking Mist, free Misty Step unlocked |
| 11 | Monk Cloak of Shadows, step into light | Ends |

## Open items
- Phase 5 of the old plan (per-party-member Undetected for enemies, player Point Out) and the AI's flat-check awareness (`pf2e-ai-overhaul`) are not done.
