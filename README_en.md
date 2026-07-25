# Pet Battle Simulator - Documentation

This is the complete documentation set for the Pet Battle Simulator, combined into one file:

1. **[Core Mechanics](#part-1-core-mechanics)** - how the battle engine works, system by system, with trivia on non-obvious behavior.
2. **[Pet Ability Documentation](#part-2-pet-ability-documentation)** - every pet's ability broken down to exact game-logic behavior, grouped by element.
3. **[Developer / Function Reference](#part-3-developer--function-reference)** - where everything lives in the code, and how to add a new pet or ability.

Start with Part 1 if you want to understand how the game plays. Start with Part 2 if you're
looking up one specific pet. Start with Part 3 if you're changing the code.

---

# Part 1: Core Mechanics
---

### 1. Teams, Decks, and HP

- Each side brings **2 decks** of **3 pets each** - 6 pets total per side. You build both decks
  in the team builder before the "Start Battle" button appears (it only shows once both sides
  have all 6 slots filled).
- Only one deck is **active** at a time; the other sits in reserve until you (or the AI) swap.
- Each deck has its own HP pool, tracked completely separately from its partner deck.
- **Base max HP is 150** per deck. If a deck contains a pet with the `passive_hp_boost` effect
  (Mini Mammoth Leash), that deck's max HP becomes **195** (+30%) - only the deck actually
  holding that pet gets the boost, not its partner.
- A deck is "dead" once its HP hits 0. A side only loses when **both** of its decks are dead
  at the same time.
- It's technically possible to "cheat death": if a deck receives healing on the exact same
  tick that a source of damage would have killed it, the heal can land in time to keep it
  alive with the healed amount of HP.
- Pets occasionally get automatically healed to full between matches without needing bandages
  (a known quirk, not an intentional mechanic).

---

### 2. The Tick Clock

- The whole battle runs on a **1-second tick** (`gameTick`, called via `setTimeout(gameTick, 1000)`),
  incrementing a global `battle.tick` counter.
- Every tick, in order: weather ticks, swap-cooldowns and cooldowns count down, DoTs/HoTs tick,
  Doom timers are checked, buffs/debuffs count down and expire, delayed-damage and heal-back
  queues advance, swap-trap timers advance, passives (like heal-when-benched) are re-checked,
  the **AI acts every other tick** (`battle.tick % 2 === 0`), and finally win conditions are
  checked.
- This means the enemy AI effectively moves at half the tick rate of raw cooldown ticking -
  cooldowns and DoTs update every tick, but the AI only gets a chance to act on even ticks.

---

### 3. Cooldowns & Using Abilities

**Starting cooldown.** At battle start, every non-passive pet's cooldown is set to
`floor(base_cooldown / 2)`, capped at a maximum of 12s, and floored at a minimum of 1s. If the
deck contains a Windspeed-type pet (`passive_cd_reduce`), that starting value is further
reduced by 2s (still floored at 1s minimum). So a 20s-cooldown pet starts at 10s (or 8s with
Windspeed), while a 3s-cooldown pet starts at 1s regardless.

**Using an ability** requires the pet to:
- be the currently *selected* active pet (not just present in the active deck),
- have cooldown ≤ 0,
- not be a passive pet (passives are never manually cast),
- not be disabled by Stun / Freeze / Sleep / Dance, and
- the deck must not be flagged `canAct: false` (also driven by Dance).

**Mess Up roll.** If the deck is carrying a `mess_up`-family debuff, the cast rolls against
that chance *before anything else happens*. On a failed roll: the ability still goes on
cooldown as normal, an "Ability failed!" message is logged, and none of the intended
effect goes out - **except** for two carve-outs: any weather-type move (Firestorm, Toxic
Cloud, Purple Haze, Count Sheep, Soothing Mist) still triggers even on a messed-up cast, and
Swoop still grants its self-dodge even on a messed-up cast.

**On a successful cast:**
1. The pet's cooldown resets to `max(1, base_cooldown - windspeed_reduction)`.
2. If an `extend_cd` debuff is armed on the deck, it adds its bonus seconds onto the
   freshly-reset cooldown and then consumes itself.
3. Every *other* pet in the same deck that was already off-cooldown (and isn't passive) is
   bumped to a 2-second cooldown - you can't freely chain-cast every ready pet in the same
   deck back-to-back; using one locks the rest for a couple seconds. **This lock always
   applies after a cast, regardless of Windspeed** - Windspeed does not prevent it here. (Its
   special no-lock behavior only applies on deck-*swap*, not on ability use.)
4. The move's damage/heal/effect resolves.
- Windspeed's own text says "minimum 1s," but that floor is bypassed entirely in one specific
  case: when a Windspeed deck is swapped *in*, none of its ready pets get the usual +2s lock
  at all - it's not that the lock is reduced to 1s, it's skipped completely.
- A pet's own listed cooldown (e.g. "10s") is really only relevant after its *first* cast -
  the opening cooldown of a battle is always the halved (and capped) value above.

---

### 4. Swapping Decks

- Each side has a Swap button/action that flips `activeDeck` between deck 1 and deck 2 (and
  resets `activePet` back to index 0 on the deck being swapped in).
- Swapping is blocked if: the current deck's swap cooldown is still counting down, the current
  deck is Frozen, or the deck is flagged `canSwap: false` (from an `anti_swap`/Banish-type
  debuff).
- On a successful swap: the deck's own **Doom** curse (if any) is immediately cancelled - this
  is the official way to shake off Doom. The newly active deck gets a **3-second swap
  cooldown** before you can swap again.
- Just like using an ability locks the deck's other ready pets to a 2s cooldown, swapping
  *into* a deck also bumps every one of its already-ready, non-passive pets to a 2s cooldown -
  **unless** that incoming deck contains a Windspeed pet, in which case this lock is skipped
  entirely and its ready pets stay ready.
- If any **swap traps** are currently armed against you, swapping away springs them
  immediately for their bonus damage before the swap resolves.
- Forced swaps (Trample, Banish, Force Partner Forward, Spirit Swap) use the same
  "clears Doom on the deck being left" rule as a voluntary swap - Doom only survives if the
  cursed deck is *prevented* from swapping (e.g. it's Frozen or locked) until the timer expires.
- Freeze is the only common crowd-control effect that also locks out swapping outright; Stun,
  Sleep, and Dance disable *actions* but don't stop you retreating to your backup deck.

---

### 5. Elements & Damage Type Advantage

- Four elements: **Fire → Earth → Air → Water → Fire** (each beats the next in the cycle).
- A hit that lands from the advantaged element deals **+25%** damage; a hit from the
  disadvantaged side deals **-25%** damage. Neutral (non-cyclical) match-ups get no modifier.
- The element used for this check isn't just "the active pet's own element" - it's the
  **deck's dominant element**: the code counts how many pets of each element are in the whole
  deck (all 3 slots, not just the active one) and uses whichever element has the most pets. If
  there's a tie, the tiebreaker falls back to the currently active pet's own element.
- This elemental multiplier is applied automatically to (almost) every hit inside `dealDamage`
  - separate from any pet-specific "elemental_bonus" ability that stacks an *additional*
  bonus on top of it for certain Dino-line moves when their own element beats the target's.
- Some effects are explicitly exempt from ever being touched by the elemental modifier at all
  - most notably Toasties (`summon`), which always hits for its flat listed damage regardless
  of either side's element, and any damage applied directly to HP outside of `dealDamage`
  (delayed spore bursts, weather ticks, recoil damage, etc.).

---

### 6. Damage Resolution Order (`dealDamage`)

When a hit is resolved, the simulator applies modifiers in this exact order:

1. **Purple Haze weather check** - if active, Fire attackers get +25% (or whatever the
   weather's stored modifier is), everyone else gets -25%, applied before anything else.
2. **Elemental advantage/disadvantage** (±25%) based on both decks' dominant elements - skipped
   for a small set of special-cased abilities (like Toasties) that ignore it entirely.
3. **Dodge check** - unless the attack ignores dodge (`anti_dodge`, `anti_dodge_block`,
   `mess_up_undodgeable`, or a Precision/Critical hit against a non-Soaring dodge), a temporary
   Dodge status or a passive dodge-chance roll can reduce the hit to 0 outright.
4. **Block check** - same idea for Block, unless the attack ignores block.
5. **Ethereal** - a flat percentage reduction to whatever damage remains, if the defender has
   an active Ethereal status.
6. **Double Damage chance** and **Next-Hit bonus** - rolled/applied for the *attacker's* buffs
   (skipped for certain "ignore my own buffs" abilities like Radiation Beam).
7. **Attacker's ongoing damage-dealt modifier** (sum of all their buffs/debuffs to damage
   dealt, e.g. Howl, Bad Luck, Face Slap).
8. **Defender's damage-taken modifier** (e.g. Bad Luck, a `dmg_taken` debuff).
9. **Flat 25% reduction** if the defender's active pet has the `passive_dmg_reduce` effect
   (only while that specific pet is selected).
10. **Hit sound plays.**
11. **Damage floor** - after every multiplier above, damage is floored to a **minimum of 1**
    (a hit can never be reduced to 0 by stacking damage-reducing modifiers alone - only a
    dodge/block/absorb can bring it to literal 0).
12. **Absorb check** - if the defender has an Absorb shield up, the entire hit (whatever it
    ended up being) is nullified and instead heals the defender for that same amount (blocked
    by Trauma if present), and the function returns 0 without ever subtracting HP.
13. **HP is subtracted.**
14. **Static Charge stacking** - any pet in the defending deck with `stack_burst` gains 1
    charge purely because the deck took damage, regardless of which pet was actually hit.
15. **Counter/Thorns reflection** - if the defender has an active counter buff, the attacker's
    deck takes the stored flat reflect damage right after the main hit lands.
- Because the damage floor (step 11) happens *before* the Absorb check (step 12), Absorb
  nullifies "at least 1 damage" even against an attack that would have been reduced to
  effectively nothing by other modifiers - it always heals for whatever the post-floor number
  came out to.
- A few special "ignores everything" attacks (Toasties, i.e. `summon`) don't go through
  `dealDamage` at all - they subtract their flat damage straight off the target's HP. That
  means every check in the pipeline above (elemental, dodge, block, Ethereal, buffs/debuffs,
  and **Purple Haze**) is skipped, not just some of them - despite the wiki's claim that
  Purple Haze is the one exception.

---

### 7. Status Effects (time-limited, tracked per-deck)

| Status | Disables acting? | Blocks swap? | What it actually does |
|---|---|---|---|
| **Stun** | Yes | No | Can't use abilities; can still swap away. |
| **Freeze** | Yes | **Yes** | Can't act *and* can't swap - the only CC that locks out retreating. |
| **Sleep** (weather) | Yes | No | Same as Stun, but caused by the Count Sheep weather; duration resets to "remaining weather time + 1s" each weather tick unless already asleep. |
| **Dance** | Yes (`canAct: false`) | No | Forced dancing; some dance-inflicting moves also grant the victim a Dodge status for the same window. |
| **Dodge** | - | - | Temporary guaranteed dodge chance layered on top of any passive dodge. |
| **Block** | - | - | Temporary guaranteed block of non-piercing hits. |
| **Trauma** | - | - | Doesn't disable actions at all - it's a pure **anti-heal flag**. Every heal source in the game (base heals, Hot ticks, weather healing, Life Drain, Absorb's payout, Revive) checks for Trauma on the target and skips the heal if present, while everything else about the hit (damage dealt, absorption, revive itself) still happens normally. |
| **DoT** | - | - | Flat damage per second for its duration, subtracted directly from HP (not run through `dealDamage`). |
| **Hot** | - | - | Same DoT clock, but heals instead of damaging (blocked by Trauma). |
| **Ethereal** | - | - | Flat % reduction to all incoming damage while active (not a dodge chance). |
| **Absorb** | - | - | Nullifies incoming damage and converts it into a heal for the defender (unless Trauma is present, in which case only the nullification happens). |
| **Doom** | - | - | A timer on the *target's* deck; if it ever reaches 0 while still present, that deck's HP is instantly zeroed. Cleared the moment that deck is voluntarily (or forcibly) swapped away. |
- **Wiki-vs-code discrepancy:** the wiki states that weather counts as a "negative effect" for
  the purposes of triggering Duck's Back (`passive_resist_negative`) or being purged by Toss
  Cookies (`cleanse`). The code doesn't implement this - weather lives entirely in a separate
  global `battle.weather` object, not in any deck's `statusEffects` or `modifiers.debuffs`
  arrays, and neither `tryResist` nor the cleanse handler ever looks at `battle.weather`. In
  this build, weather can be neither resisted nor purged.
- The "doesn't tick on its very last second" behavior is **specific to weather**, not a
  general rule for timed effects. Regular DoT ticks (and Hot heals) actually still apply on
  the tick where their `remaining` counts down to exactly 0, before being removed - so a DoT
  dealing X/s for Y seconds lands its full `X × Y` total. Only the weather system has the
  extra bookkeeping (`tickCounter` vs. `remaining`) that skips a payout on its final tick.
- Recharge-speed modifiers (from slow/fast-recharge debuffs/buffs) also stretch or shrink how
  long Stun/Freeze/Sleep durations last on the affected deck, not just how fast cooldowns fill.

---

### 8. Buffs & Debuffs (deck-wide modifiers, distinct from status effects)

These live in a deck's `modifiers.buffs` / `modifiers.debuffs` arrays and get recomputed into
flat totals every tick (`recomputeModifiers`):

**Buffs**
- `buff_damage` - flat % more damage dealt.
- `dmg_speed` - % more damage dealt AND doubles recharge speed simultaneously.
- `double_dmg` - % chance for any given hit to be doubled.
- `next_dmg` - % bonus that applies to exactly one future hit, then deletes itself.
- `fast_cd` - doubles recharge speed (cooldowns tick down twice as fast).
- `counter_reflect` - reflects a flat amount of damage back at whoever lands the next hit(s).

**Debuffs**
- `dmg_taken` - flat % more damage taken.
- `debuff_damage` - flat % less damage dealt.
- `bad_luck` - both of the above at once, driven by a single value.
- `slow_cd` - halves recharge speed.
- `mess_up` - % chance for the deck's own next cast to simply fail.
- `no_swap` - swapping is completely locked out.
- `extend_cd` - a one-shot debuff that adds bonus seconds to the *next* ability this deck
  finishes casting, then disappears.

**Stacking rule.** For most sources, re-applying the same buff/debuff type from the same pet
just **refreshes its duration** to the new value rather than adding another copy. A small,
specifically-flagged set of pets instead **stack**: reapplying from the *same* named source
adds to the existing value (capped, typically at 400% for damage modifiers or 200% for
damage-taken), and extends the duration to whichever is longer rather than resetting it.
Different pets/sources always coexist as independent entries regardless of stacking rules.
- Because `buff_damage`/`dmg_speed`/`debuff_damage`/`bad_luck`/`dmg_taken` all feed into the
  same running totals, a deck can simultaneously be buffed and debuffed on damage dealt from
  different sources - they add algebraically rather than one overriding the other.

---

### 9. Weather (global, affects both sides equally)

Unlike status effects (per-deck) and buffs (per-deck), **weather is a single global slot** -
only one weather effect can be active on the whole battlefield at a time, and casting a new
one overwrites whatever was there before. Every weather effect ticks on its own internal rate
independent of the main per-second clock:

| Weather | Trigger source | Tick rate | Effect |
|---|---|---|---|
| Firestorm | `isDot` + 8 dmg | every 2s | Both sides' **active** decks take flat unmitigated damage. |
| Toxic Cloud | `isDot` + 4 dmg | every 5s | Same as Firestorm, smaller tick. |
| Purple Haze | flat `effectValue`, no DoT | every 1s | Modifies elemental damage globally: Fire attackers +25%, everyone else -25%, for the duration - this is checked first in `dealDamage`, ahead of the normal element-cycle bonus. |
| Soothing Mist | `weather_hot` | every 5s | Both sides' active decks are healed (blocked by Trauma per side). |
| Count Sheep | `sleep` | every 1s | Both sides' active decks are put to Sleep, refreshed each tick to "remaining weather + 1s" unless already asleep. |
- Weather always triggers even if the caster's own move was messed up by a `mess_up`-type
  debuff - it's one of the two hard-coded exceptions to mess-up (the other being Swoop's
  guaranteed self-dodge).
- Damage/heal weather effects skip their payout on the very last tick before the weather
  expires, so the realized total is always a bit less than "tick amount × ticks you'd naively
  expect."
- **Wiki-vs-code discrepancy:** the wiki claims Purple Haze is the one exception to "Toasties
  ignores every damage modifier" (i.e. that a Toasties hit should still be adjusted by Purple
  Haze's Fire/non-Fire split). The current code doesn't implement that - Toasties' `summon`
  handler bypasses `dealDamage` entirely, and Purple Haze's modifier only ever runs
  inside `dealDamage`, so in this build Purple Haze has **no effect** on Toasties damage
  despite what the wiki says.

---

### 10. Special One-Off Mechanics

- **Delayed damage** - some moves don't hit immediately; instead they queue a future burst
  that resolves itself after N ticks pass, dealt with no attacking-pet context (so it isn't
  affected by that pet's own piercing traits, only whatever state the target is in when it
  actually lands).
- **Heal-back** - the opposite of delayed damage: a hit's damage is scheduled to return to the
  target as healing, spread roughly evenly across a follow-up window (blocked by Trauma like
  any heal).
- **Swap traps** - an armed, invisible timer on the enemy; if they voluntarily swap away while
  it's active, it springs for bonus damage and then consumes itself. The trap's own *setup*
  animation is visible even though its presence isn't flagged anywhere in the UI once armed.
- **Chain-on-kill** - if a hit finishes off the target's active deck, a second identical hit
  immediately follows into their backup deck (if it's alive); no chain happens if the first
  hit doesn't finish the active deck.
- **Force-swap family** (Trample / Banish / Force Partner Forward / Spirit Swap) - all push
  the enemy onto their backup deck if it's alive, swappable, and not Frozen; they differ only
  in extras layered on top (Banish additionally locks the deck they land on from swapping back
  out for a duration; Spirit Swap additionally drags every current status/buff/debuff off the
  active deck and onto the deck being swapped into).
- **Absorb-class stacking** - a couple of specific shield-type moves are treated as the "same
  move class": recasting one before the other has expired *extends* the remaining duration
  instead of resetting it.

---

### 11. Deck Death & Revival

- **Living Dead Remote** (`passive_revive`) only does anything while it is sitting in the
  **benched** (inactive) deck - it has no effect while its own deck is the active one. The
  instant the OTHER deck (its own deck's active partner) hits 0 HP, and provided the Remote's
  own benched deck is still alive, it immediately revives that just-died deck in place to a
  percentage of its max HP - no swap happens, the revived deck simply keeps going as the
  active deck. This is a **once-per-battle** effect per side: after it fires, the Remote is
  spent for the rest of the match (shown as a "USED" state on its passive icon), regardless of
  how many copies of it the player fields.
- If a deck dies and there is **no** live, benched Remote available to save it (no copy present,
  it was already used this battle, or the Remote's own deck happens to be dead too), the game
  falls back to the normal behavior: the active deck automatically switches over to the
  surviving partner deck at the next tick, if that partner deck is still alive.
- Separately, an active-use **Revive** ability can bring back a dead *backup* deck (not the
  currently active one) to a percentage of its max HP any time it's used and the backup deck
  happens to be at 0 - casting it while the backup deck is still alive just does nothing. This
  is independent of Living Dead Remote and isn't limited by the same once-per-battle flag.
- The battle ends the instant one side has both decks dead at the same time (checked right
  after the death-handling above has had its chance to intervene) - displayed as a win/loss
  banner, and the tick loop stops running.
- Because Living Dead Remote triggers off the specific moment its partner deck's HP hits 0,
  it will not save a side where *both* decks are brought to 0 on the very same tick (e.g. by
  an effect that hits both decks at once) - the Remote's own deck must still be standing at
  the moment of the check, so a simultaneous double-KO goes through as a normal loss.
- Trauma does **not** block a Revive/Reanimate-type effect from working - Trauma only blocks
  the passive *heal* portion of things, not a full revive.

---

### 12. Enemy AI Behavior

Every other tick, the AI:
1. If its current active pet is off-cooldown, usable, and the deck isn't disabled - casts it.
2. Otherwise, if any *other* pet in the same active deck is off-cooldown and usable, it
   switches its selection to that pet instead (still spending that tick just to switch).
3. Otherwise, if its swap is off cooldown, its backup deck is alive, swapping isn't locked out,
   and the backup deck actually has *some* usable pet ready to go - it swaps decks.
4. If none of the above apply, it does nothing that tick and waits for a cooldown to tick down.

The AI never manually queues a "wait" - every action it takes is basically "use the best thing
that's currently available," re-evaluated fresh every other tick.

---

### 13. Win Condition

A side loses the instant *both* of its decks are simultaneously at 0 HP, checked at the very
end of every tick - after Living Dead has had its one chance to intervene. If both sides
somehow reach 0 on the exact same tick, the check order still resolves "your side" first, so a
true simultaneous double-KO is decided in the *enemy's* favor by evaluation order (your death
is checked, and if true, would already end the battle as an enemy win before your own
Living-Dead-adjusted status could flip it back, unless your revive triggers on the same pass).


---

# Part 2: Pet Ability Documentation


### Table of Contents
- [Air](#air) (49 pets)
- [Earth](#earth) (48 pets)
- [Fire](#fire) (47 pets)
- [Water](#water) (49 pets)

---

### Air

1. Eye Of Growganoth - Air | Gift of Growganoth | Passive
`effect: passive_heal_inactive, effectValue: 7, effectDuration: 5`
**Heal:** 7
> *Wiki ability text - Gift of Growganoth:* Passive - Heal 7 health every 5s when not active.

Always active. Every 5 game-ticks (checked as `battle.tick % 5 === 0`), if this pet's deck is currently benched (not the active pet on the active deck) and the deck is alive and not carrying Trauma, it heals 7 HP. Silent, automatic - no cast, no cooldown, no log entry beyond the heal.

2. Ladybug Leaf - Air | Exoskeleton | Passive
`effect: passive_shorten_debuff, effectValue: 2`
> *Wiki ability text - Exoskeleton:* Negative effects on you have 2s shorter duration (minimum 1s).

Always active (`exoReduction`). Any negative duration-based effect this deck would receive from an opponent - DoTs, Stun/Freeze/Trauma, and generic debuffs applied via `applyModifiers`/`applyStatusEffect` - has its duration cut by 2s (floored at 0) at the moment it's applied. Self-inflicted effects (e.g. a pet debuffing itself) are not shortened, since the reduction only applies on the `!isSelf` path.

3. Summer Kite - Air | Windspeed | Passive
`effect: passive_cd_reduce, effectValue: 2`
> *Wiki ability text - Windspeed:* Passive - all cooldowns are 2s shorter (minimum 1s).

Always active. Three separate effects from one passive: (1) every pet's starting cooldown in the same deck is reduced by 2s at battle start, (2) whenever a pet in this deck finishes casting, its new cooldown is also -2s, and (3) if you swap into a deck containing this pet, the usual "+2s lock" penalty on ready pets is skipped entirely. Benefits the whole deck, not just itself.

4. Yeonnalligi - Air | Soaring | Passive
`effect: passive_dodge, effectValue: 25`
> *Wiki ability text - Soaring:* Passive - 25% chance to dodge attacks.

Always active, no cooldown. While this pet is the one currently selected (not just in the deck - the active slot specifically), every incoming hit has a 25% chance to whiff entirely (0 damage). There's a specific carve-out in `dealDamage` for it: normally "Precision"-line dino attacks (`anti_dodge_block_conditional`) pierce dodge, but if the dodge roll came from this pet's passive specifically, Precision does not pierce it - the passive wins and the attack is dodged like normal. Matches the wiki trivia about Soaring vs. Precision/Critical.

5. Betty Bluetooth Doll - Air | Static Charge | Active (3s CD)
`effect: stack_burst, effectValue: 10`
**Damage:** 5 (air) | **Cooldown:** 3s
> *Wiki ability text - Static Charge:* Each time you take damage, gain 1 static charge (up to 10). This skill launches them all at once, for 5 damage each.

Passive-and-active hybrid: passively, every time this pet's deck takes damage - from ANY source, whether or not this pet is the one active - it gains 1 charge, capped at 10. Its own active ability, 3s cooldown, then launches all banked charges at once, dealing `5 × charges` total damage and resetting the charge count to 0. Casting with 0 charges banked deals no damage at all (the ability still goes on cooldown, but nothing is launched).

6. Playful Wind Sprite - Air | Playful Wind | Active (4s CD)
`effect: buff_chance_double_damage, effectValue: 15, effectDuration: 8`
**Damage:** 6 (air) | **Cooldown:** 4s
> *Wiki ability text - Playful Wind:* Grants you a 15% chance to do double damage for 8 seconds. Inflicts 6 Air damage.

Active ability, 4s cooldown. Deals 6 damage, then grants the caster's own deck a `double_dmg` buff (`15%` chance) lasting 8s. In `dealDamage`, every subsequent attack from this deck while the buff is up rolls against `doubleDmgChance`; on a hit, that single attack's damage is doubled. Reapplying before it expires simply refreshes the timer to 8s (buffs from different sources stack independently rather than overwriting each other).

7. Chick Leash - Air | Chirp | Active (5s CD)
`effect: anti_block`
**Damage:** 9 (air) | **Cooldown:** 5s
> *Wiki ability text - Chirp:* Cheep adorably. Inflicts 9 Air damage. Can't be blocked.

Active ability, 5s cooldown. Deals 9 damage. Sets `ignoresBlock` in `dealDamage`, so a Block status on the target is skipped entirely for this hit - Dodge is unaffected and can still proc normally against it.

8. Flying Bell - Air | Sound Blast! | Active (5s CD)
`effect: anti_dodge`
**Damage:** 9 (air) | **Cooldown:** 5s
> *Wiki ability text - Sound Blast!:* Blast sound waves to deal 9 Air damage! Inflicts 9 Air damage. Can't be dodged.

Active ability, 5s cooldown. Deals 9 damage. Sets `ignoresDodge` in `dealDamage`, so neither a temporary Dodge status nor a passive dodge chance (e.g. Soaring) can save the target from this hit - Block is unaffected and can still stop it.

9. Grey Pet Pteranodon - Air | Precision Strike A | Active (5s CD)
`effect: anti_dodge_block_conditional, effectValue: 25`
**Damage:** 7 (air) | **Cooldown:** 5s
> *Wiki ability text - Precision Strike A:* If the opponent attempts to dodge or block, hits for 25% more damage. Inflicts 7 Air damage. Can't be dodged or blocked.

Active ability, 5s cooldown, the "Precision/Critical" dino line. Deals 7 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 25% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

10. Leashed Silkworm - Purple - Air | Face Slap | Active (5s CD)
`effect: debuff_damage_dealt, effectValue: 25, effectDuration: 10`
**Damage:** 12 (air) | **Cooldown:** 5s
> *Wiki ability text - Face Slap:* Smack the target in the face, enraging them to inflict 25% more damage for 10s. The rage stacks up to 20 times. Inflicts 12 Air damage.

Active ability, 5s cooldown ("Face Slap" line). Deals 12 damage, then applies a `debuff_damage` debuff to the TARGET (not the caster) for 10s: -25% to whatever damage the target deals while it's up. Per wiki trivia this is phrased as "enraging them" - it's a debuff placed on the enemy, not a self-buff on the caster. If this pet is one of the stacking sources (Leashed Silkworm - Purple, Pet Present Goblin), repeated casts stack the reduction (capped at 400%) instead of resetting it.

11. Leashed Silkworm - White - Air | Psyblade | Active (5s CD)
`effect: self_damage, effectValue: 4`
**Damage:** 12 (air) | **Cooldown:** 5s
> *Wiki ability text - Psyblade:* Shoot a mind blade, causing psychic backlash to you for 4 damage. Inflicts 12 Air damage.

Active ability, 5s cooldown ("Psyblade"). Deals 12 damage to the enemy through the normal generic damage path, and separately, this pet's own deck takes 4 flat unmitigated recoil damage (subtracted directly from HP, bypassing `dealDamage` entirely - so it ignores the caster's own dodge/block/Ethereal/etc.).

12. Sonar Bracelet - Air | Sonic Burst | Active (5s CD)
`effect: anti_dodge`
**Damage:** 9 (air) | **Cooldown:** 5s
> *Wiki ability text - Sonic Burst:* Blast sonic waves. Inflicts 9 Air damage. Can't be dodged.

Active ability, 5s cooldown. Deals 9 damage. Sets `ignoresDodge` in `dealDamage`, so neither a temporary Dodge status nor a passive dodge chance (e.g. Soaring) can save the target from this hit - Block is unaffected and can still stop it.

13. Grey Pet Apatodon - Air | Dino Dive A | Active (6s CD)
`effect: elemental_bonus, effectValue: 50`
**Damage:** 10 (air) | **Cooldown:** 6s
> *Wiki ability text - Dino Dive A:* Against a Water opponent, inflicts +50% damage (on top of normal bonus.) Inflicts 10 Air damage.

Active ability, 6s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 10 base Air damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 50/100` (i.e. +50%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 10 damage with no bonus.

14. Grey Pet Apatoceratops - Air | Dino Charge A | Active (8s CD)
`effect: elemental_bonus, effectValue: 60`
**Damage:** 14 (air) | **Cooldown:** 8s
> *Wiki ability text - Dino Charge A:* Against a Water opponent, inflicts +60% damage (on top of normal bonus). Inflicts 14 Air damage.

Active ability, 8s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 14 base Air damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 60/100` (i.e. +60%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 14 damage with no bonus.

15. Rainbow Kite - Air | Taste The Rainbow | Active (8s CD)
`effect: random_element`
**Damage:** 16 (air) | **Cooldown:** 8s
> *Wiki ability text - Taste The Rainbow:* Blast a rainbow beam that inflicts damage of a random element. Inflicts 16 Air damage.

Active ability, 8s cooldown ("Taste the Rainbow"). For this cast only, the pet's element is swapped to a randomly rolled one of Fire/Earth/Air/Water before damage is resolved (affecting both the elemental-advantage multiplier and the Purple Haze fire-check), then restored back to its real element immediately afterward. Deals 16 base damage, modified by whichever element was rolled that cast.

16. Grey Pet Pteratops - Air | Precision Attack A | Active (9s CD)
`effect: anti_dodge_block_conditional, effectValue: 17`
**Damage:** 11 (air) | **Cooldown:** 9s
> *Wiki ability text - Precision Attack A:* If the opponent attempts to dodge or block, hits for 17% more damage. Inflicts 11 Air damage. Can't be dodged or blocked.

Active ability, 9s cooldown, the "Precision/Critical" dino line. Deals 11 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 17% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

17. Pineapple Kite - Air | Pineapple Bomb | Active (9s CD)
`effect: none`
**Damage:** 18 (air) | **Cooldown:** 9s
> *Wiki ability text - Pineapple Bomb:* Lob a heavy pineapple. Inflicts 18 Air damage.

Active ability, 9s cooldown. Deals 18 flat damage with no secondary effect attached - a plain, unconditional hit.

18. Red Floaty Balloon - Air | Floaty Bomb | Active (9s CD)
`effect: none`
**Damage:** 18 (air) | **Cooldown:** 9s
> *Wiki ability text - Floaty Bomb:* Lob a heavy balloon at your foes. Inflicts 18 Air damage.

Active ability, 9s cooldown. Deals 18 flat damage with no secondary effect attached - a plain, unconditional hit.

19. Butterfly Leash - Air | Acrobatics | Active (10s CD)
`effect: dodge, effectDuration: 5`
**Cooldown:** 10s
> *Wiki ability text - Acrobatics:* Dodge all attacks for 5s.

Active ability, 10s cooldown. Grants the caster's own deck a Dodge status for 5s. While Dodge is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore dodge (not anti_dodge/anti_dodge_block/mess_up_undodgeable, and not a piercing Precision hit) and reduces it to 0 before the damage floor. Per wiki trivia, at least one pet on this effect (Swoop) is guaranteed to grant its dodge window even if the cast itself was messed up by a mess_up-type debuff.

20. Electro Magnifying Glass - Air | Shock Blast | Active (10s CD)
`effect: stun, effectDuration: 1`
**Damage:** 18 (air) | **Cooldown:** 10s
> *Wiki ability text - Shock Blast:* Electrocute for a 1s stun. Inflicts 18 Air damage.

Active ability, 10s cooldown. Deals 18 damage, then stuns the target for 1s - disabling its ability to act, but (unlike Freeze) NOT blocking its Swap button, so a stunned deck can still retreat to its backup at any time.

21. Grey Pet Apatos Rex - Air | Dino Chomp A | Active (10s CD)
`effect: elemental_bonus, effectValue: 25`
**Damage:** 18 (air) | **Cooldown:** 10s
> *Wiki ability text - Dino Chomp A:* Against a Water opponent, inflicts +25% damage (on top of normal bonus). Inflicts 18 Air damage.

Active ability, 10s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 18 base Air damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 25/100` (i.e. +25%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 18 damage with no bonus.

22. Grey Pet Apatosaurus - Air | Dino Slam A | Active (10s CD)
`effect: elemental_bonus, effectValue: 100`
**Damage:** 15 (air) | **Cooldown:** 10s
> *Wiki ability text - Dino Slam A:* Against a Water opponent, inflicts +100% damage (on top of normal bonus). Inflicts 15 Air damage.

Active ability, 10s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 15 base Air damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 100/100` (i.e. +100%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 15 damage with no bonus.

23. Grey Pet Triceradon - Air | Defensive Flurry A | Active (10s CD)
`effect: block, effectDuration: 2`
**Damage:** 12 (air) | **Cooldown:** 10s
> *Wiki ability text - Defensive Flurry A:* Strike and then block attacks for 2s. Inflicts 12 Air damage.

Active ability, 10s cooldown. Deals 12 damage to the enemy, then grants the caster's own deck a self-Block status for 2s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

24. Grey Pet Tyranodon - Air | Piercing Jaws A | Active (10s CD)
`effect: trauma, effectDuration: 4`
**Damage:** 15 (air) | **Cooldown:** 10s
> *Wiki ability text - Piercing Jaws A:* Causes trauma, preventing the target from healing for 4s. Inflicts 15 Air damage.

Active ability, 10s cooldown. Deals 15 damage, then inflicts Trauma on the target for 4s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

25. Haunted Pants - Air | Phantom Pain | Active (10s CD)
`effect: heal_back, effectDuration: 10`
**Damage:** 30 (air) | **Cooldown:** 10s
> *Wiki ability text - Phantom Pain:* Inflicts phantasmal damage, which heals back over 10s. Inflict 30 Air damage.

Active ability, 10s cooldown ("Phantom Pain"). Deals 30 damage immediately, then schedules that same 30 amount to be healed back onto the TARGET gradually over the following 10s (roughly evenly split per tick via `battle.healBacks`, blocked by Trauma just like any other heal). Net effect over the full window is a wash in raw HP unless the target dies or is Traumatized before the heal-back finishes.

26. Haunted Synthoid - Air | Draining Beam | Active (10s CD)
`effect: extend_cooldown, effectValue: 3`
**Damage:** 5 (air) | **Cooldown:** 10s
> *Wiki ability text - Draining Beam:* Drain the enemy. Any skills they have on cooldown will take 3s longer to recharge. Inflicts 5 Air damage.

Active ability, 10s cooldown. Two behaviors depending on the target's state: if their active pet is already mid-recharge, this directly adds 3s onto that pet's remaining cooldown right now. If their active pet is instead already off cooldown (ready), this arms an `extend_cd` debuff lasting 0s that adds 3s of extra cooldown the very next time that deck's active pet finishes casting anything, then consumes itself.

27. Lucky Pendant - Air | Lucky Strike | Active (10s CD)
`effect: random_damage, effectValue: 40`
**Damage:** 1 (air) | **Cooldown:** 10s
> *Wiki ability text - Lucky Strike:* Attack enemy with a chance to inflict 1 - 40 Air damage.

Active ability, 10s cooldown ("Lucky Strike"). Rolls a random integer between `min(damage, effectValue)` and `max(damage, effectValue)` inclusive - i.e. between 1 and 40 - and deals that amount as the hit's damage. The pet's listed `damage` field (1) is effectively just the low end of the roll, not a guaranteed minimum hit on its own.

28. Mid-Pacific Owl - Air | Swoop | Active (10s CD)
`effect: dodge, effectDuration: 2`
**Damage:** 15 (air) | **Cooldown:** 10s
> *Wiki ability text - Swoop:* Attack, and dodge all attacks for 2s. Inflicts 15 Air Damage.

Active ability, 10s cooldown. Deals 15 damage, then grants the caster's own deck a Dodge status for 2s. While Dodge is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore dodge (not anti_dodge/anti_dodge_block/mess_up_undodgeable, and not a piercing Precision hit) and reduces it to 0 before the damage floor. Per wiki trivia, at least one pet on this effect (Swoop) is guaranteed to grant its dodge window even if the cast itself was messed up by a mess_up-type debuff.

29. Pet Hatchley - Air | Musically Stunned! | Active (10s CD)
`effect: stun, effectDuration: 8`
**Damage:** 10 (air) | **Cooldown:** 10s
> *Wiki ability text - Musically Stunned!:* Sings to deal 10 Air Damage and stuns the target for 8s, making them unable to act.

Active ability, 10s cooldown. Deals 10 damage, then stuns the target for 8s - disabling its ability to act, but (unlike Freeze) NOT blocking its Swap button, so a stunned deck can still retreat to its backup at any time.

30. Zorbnik Leash - Air | Stun Ray | Active (10s CD)
`effect: stun, effectDuration: 3`
**Damage:** 10 (air) | **Cooldown:** 10s
> *Wiki ability text - Stun Ray:* Stuns the target for 3s, making them unable to act. Inflicts 10 Air damage.

Active ability, 10s cooldown. Deals 10 damage, then stuns the target for 3s - disabling its ability to act, but (unlike Freeze) NOT blocking its Swap button, so a stunned deck can still retreat to its backup at any time.

31. Blue Floaty Balloon - Air | Bouncy Barrier | Active (11s CD)
`effect: counter, effectDuration: 4`
**Damage:** 25 (air) | **Cooldown:** 11s
> *Wiki ability text - Bouncy Barrier:* Bounce attacks back for 25 damage if hit within 4s.

Active ability, 11s cooldown, no direct damage on cast. Arms a `counter_reflect` buff on the caster's deck worth 25 (using the pet's `damage` field as the reflect amount) for 4s. The next time - and every time - this deck actually takes a landed hit while the buff is active, `dealDamage` reflects that flat 25 amount straight back at the attacker's deck, on top of the damage the caster itself took. Recasting replaces any existing counter buff rather than stacking.

32. Grey Pet Triceratops - Air | Defensive Gore A | Active (13s CD)
`effect: block, effectDuration: 4`
**Damage:** 16 (air) | **Cooldown:** 13s
> *Wiki ability text - Defensive Gore A:* Strike and then block attacks for 4s. Inflicts 16 Air damage.

Active ability, 13s cooldown. Deals 16 damage to the enemy, then grants the caster's own deck a self-Block status for 4s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

33. Grey Pet Tyranotops - Air | Crushing Beak A | Active (13s CD)
`effect: trauma, effectDuration: 7`
**Damage:** 19 (air) | **Cooldown:** 13s
> *Wiki ability text - Crushing Beak A:* Causes Trauma, preventing the target from healing for 7 seconds. Inflicts 19 Air damage.

Active ability, 13s cooldown. Deals 19 damage, then inflicts Trauma on the target for 7s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

34. Grey Pet Pterosaurus - Air | Critical Strike A | Active (15s CD)
`effect: anti_dodge_block_conditional, effectValue: 50`
**Damage:** 17 (air) | **Cooldown:** 15s
> *Wiki ability text - Critical Strike A:* If the opponent attempts to dodge or block, hits for 50% more damage. Inflicts 17 Air damage. Can't be dodged or blocked.

Active ability, 15s cooldown, the "Precision/Critical" dino line. Deals 17 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 50% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

35. Pineapple Blump Pet - Air | Spikey Shield | Active (15s CD)
`effect: counter, effectDuration: 8`
**Damage:** 20 (air) | **Cooldown:** 15s
> *Wiki ability text - Spikey Shield:* Strike back for 20 Air Damage if hit within 8s.

Active ability, 15s cooldown, no direct damage on cast. Arms a `counter_reflect` buff on the caster's deck worth 20 (using the pet's `damage` field as the reflect amount) for 8s. The next time - and every time - this deck actually takes a landed hit while the buff is active, `dealDamage` reflects that flat 20 amount straight back at the attacker's deck, on top of the damage the caster itself took. Recasting replaces any existing counter buff rather than stacking.

36. Grey Pet Tyrannosaurus - Air | Crushing Jaws A | Active (16s CD)
`effect: trauma, effectDuration: 3`
**Damage:** 29 (air) | **Cooldown:** 16s
> *Wiki ability text - Crushing Jaws A:* Causes trauma, preventing the target for 3s. Inflicts 29 Air damage.

Active ability, 16s cooldown. Deals 29 damage, then inflicts Trauma on the target for 3s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

37. Grey Pet Tricerus Rex - Air | Defensive Bite A | Active (18s CD)
`effect: block, effectDuration: 2`
**Damage:** 22 (air) | **Cooldown:** 18s
> *Wiki ability text - Defensive Bite A:* Strike and then block attacks for 2s. Inflicts 22 Air damage.

Active ability, 18s cooldown. Deals 22 damage to the enemy, then grants the caster's own deck a self-Block status for 2s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

38. Grey Pet Pteranus Rex - Air | Precision Crush A | Active (19s CD)
`effect: anti_dodge_block_conditional, effectValue: 10`
**Damage:** 25 (air) | **Cooldown:** 19s
> *Wiki ability text - Precision Crush A:* If the opponent attempts to dodge or block, hits for 10% more damage. Inflicts 25 Air damage. Can't be dodged or blocked.

Active ability, 19s cooldown, the "Precision/Critical" dino line. Deals 25 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 10% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

39. Cloud Rabbit - Air | Self Sacrifice | Active (20s CD)
`effect: dodge_swap, effectDuration: 4`
**Cooldown:** 20s
> *Wiki ability text - Self Sacrifice:* If struck in the next 4 seconds, avoid it completely and swap out instead (if able to swap). The smoke left behind helps your teammate dodge all attacks for 4 seconds. Isn't visible to your enemy.

Active ability, 20s cooldown ("Self Sacrifice"/"Trick Escape"). If the caster has a living backup deck and is currently allowed to swap, this instantly swaps them onto it (clearing Doom on the deck being left) and leaves a Dodge status worth 4s on the deck being SWAPPED INTO - the incoming teammate gets the dodge window, not the deck retreating (since the retreating deck is now inactive and can't be hit anyway). Per wiki trivia this move can in principle also be reactively triggered by any other self-targeted move the pet uses (Stoneform, Acrobatics, Hop, etc.), but the simulator only implements it for this pet's own direct cast of the move.

40. Cosmic Unicorn Bracelet - Air | Cosmic Barrage | Active (20s CD)
`effect: anti_dodge`
**Damage:** 36 (air) | **Cooldown:** 20s
> *Wiki ability text - Cosmic Barrage:* Crush the target with a comet from the heavens. Inflicts 36 Air damage. Can't be dodged.

Active ability, 20s cooldown. Deals 36 damage. Sets `ignoresDodge` in `dealDamage`, so neither a temporary Dodge status nor a passive dodge chance (e.g. Soaring) can save the target from this hit - Block is unaffected and can still stop it.

41. Discoball Companion - Air | Just Dance! | Active (20s CD)
`effect: force_dance, effectDuration: 4, dotDamage: 5, dotDuration: 4`
**Damage:** 5 (air) | **Cooldown:** 20s | **DoT:** 5/s for 4s
> *Wiki ability text - Just Dance!:* Infect the target to dance for 4s, unable to do anything other than swap out, and suffering for 5 Air Damage per second.

Active ability, 20s cooldown ("Enthrall"/"Disco Fever"/"Just Dance"). Deals 5 damage (if any) and locks the target into a Dance status for 4s, during which `canAct` is set false - the target can't use any ability at all, though it can still be swapped out normally (Dance isn't in the swap-blocking status list). If this specific move is "Disco Fever", it additionally grants the target a matching Dodge status for the same duration - per wiki trivia, that's the source of the "dance" flavor of the debuff forcing a dodge window on the victim, not the caster.

42. Green Floaty Balloon - Air | Pumped Up | Active (20s CD)
`effect: none`
**Heal:** 30 | **Cooldown:** 20s
> *Wiki ability text - Pumped Up:* Pump yourself up to regenerate health. Heals 30 life.

Active ability, 20s cooldown. No damage, no special effect - restores 30 flat HP to the caster's own deck and nothing else.

43. Grey Pet Tripatosaurus - Air | Defensive Bash A | Active (20s CD)
`effect: block, effectDuration: 7`
**Damage:** 10 (air) | **Cooldown:** 20s
> *Wiki ability text - Defensive Bash A:* Strike and then block attacks for 7s. Inflicts 10 Air damage

Active ability, 20s cooldown. Deals 10 damage to the enemy, then grants the caster's own deck a self-Block status for 7s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

44. Grey Pet Tyranopatos - Air | Grinding Jaws A | Active (20s CD)
`effect: trauma, effectDuration: 30`
**Damage:** 3 (air) | **Cooldown:** 20s
> *Wiki ability text - Grinding Jaws A:* Causes Trauma, preventing the target from healing for 30s. Inflicts 3 Air damage.

Active ability, 20s cooldown. Deals 3 damage, then inflicts Trauma on the target for 30s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

45. Lovebird Pendant - Air | Enthrall | Active (20s CD)
`effect: force_dance, effectDuration: 6`
**Damage:** 10 (air) | **Cooldown:** 20s
> *Wiki ability text - Enthrall:* Mesmerize your enemy, so it can do nothing but swap out for 6s. Inflicts 10 Air Damage.

Active ability, 20s cooldown ("Enthrall"/"Disco Fever"/"Just Dance"). Deals 10 damage (if any) and locks the target into a Dance status for 6s, during which `canAct` is set false - the target can't use any ability at all, though it can still be swapped out normally (Dance isn't in the swap-blocking status list). If this specific move is "Disco Fever", it additionally grants the target a matching Dodge status for the same duration - per wiki trivia, that's the source of the "dance" flavor of the debuff forcing a dodge window on the victim, not the caster.

46. Spiritual Resonator - Air | Possession | Active (20s CD)
`effect: random_skill_wrong_target`
**Cooldown:** 20s
> *Wiki ability text - Possession:* Possesses the target into using a random skill on the wrong target.

Active ability, 20s cooldown ("Possession"). No direct damage from this pet itself. Picks a random pet on the enemy's ACTIVE deck that has any damage attack, and turns that attack against the enemy's own deck for its normal listed damage. If the enemy's active pet lineup has no attacking pet to possess, the move simply fizzles with no effect. (Flagged in the source as a best-guess implementation - the wiki doesn't spell out the exact selection/targeting rule.)

47. Death's Scarf - Air | Doom | Active (30s CD)
`effect: doom, effectDuration: 20`
**Cooldown:** 30s
> *Wiki ability text - Doom:* Mark the target for death in 20s. The effect can be erased by simply swapping pets.

Active ability, 30s cooldown. Deals 0 damage, then curses the target deck with Doom for 20s. If the timer runs out while Doom is still on the deck, `gameTick` instantly zeroes its HP. Crucially, Doom is cleared the instant the cursed deck voluntarily swaps away (`swapDeck` strips any `doom` status effect on swap) - matching the wiki's "erased by simply swapping" note - but it survives being forced to stay in place by anything that locks swapping.

48. Thinking Cap - Air | Mind Swap | Active (30s CD)
`effect: mind_swap`
**Cooldown:** 30s
> *Wiki ability text - Mind Swap:* Take all the positive effects on your opponent, and give all your negative effects to them.

Active ability, 30s cooldown ("Thinking Cap"). No direct damage. Per the wiki: "take all the positive effects on your opponent, and give all your negative effects to them." Mechanically, every buff currently on the enemy's deck is moved onto the caster's deck (the enemy's buff list is wiped), and every debuff currently on the caster's deck is moved onto the enemy's deck (the caster's debuff list is wiped) - a simultaneous full swap of buffs one direction and debuffs the other.

49. Onisim's Genie - Air | Wish | Active (999s CD)
`effect: revive, effectValue: 30`
**Heal:** 30 | **Cooldown:** 999s
> *Wiki ability text - Wish:* Bring your partner back to life with 30% health!

Active ability, 999s cooldown, up to 0s cast window implied by its long cooldown. If the caster's own backup deck is currently at 0 HP, this revives it to 30% of its max HP. If the backup deck is still alive, casting this ability does nothing (no healing, no effect) - it only works as a genuine revive, not a heal.

---

### Earth

50. Mini Mammoth Leash - Earth | Mammoth Heart | Passive
`effect: passive_hp_boost, effectValue: 30`
> *Wiki ability text - Mammoth Heart:* Passive - Maximum life is 30% higher.

Always active. `getMaxHp` checks whether the active deck contains a pet with this effect; if so, max HP for that deck is 195 instead of the normal 150 (a flat +30%, matching the wiki). Only the deck holding this pet gets the boost - the partner deck stays at 150 unless it has its own copy.

51. Pet Burrito - Earth | Stubborn | Passive
`effect: passive_dmg_reduce, effectValue: 25`
> *Wiki ability text - Stubborn:* Passive - Suffers 25% less damage from all attacks.

Always active, but only while this pet is the currently selected active pet (checked in `dealDamage` against `activePet`). Every hit it would take is multiplied by 0.75 (a flat 25% reduction) - this is applied after elemental bonuses, dodge/block checks, and Ethereal, but before the 1-HP damage floor. Per wiki trivia, this reduction switches off the instant the pet is swapped out.

52. Baby Bunny - Earth | Hop | Active (3s CD)
`effect: none`
**Cooldown:** 3s
> *Wiki ability text - Hop:* Hop. (This ability does not do anything beside making the current pet jump)

Active ability, 3s cooldown. Deals 0 flat damage with no secondary effect attached - a plain, unconditional hit.

53. Puppy Leash - Earth | Pounce | Active (4s CD)
`effect: none`
**Damage:** 9 (earth) | **Cooldown:** 4s
> *Wiki ability text - Pounce:* Leap to strike! Inflicts 9 Earth damage.

Active ability, 4s cooldown. Deals 9 flat damage with no secondary effect attached - a plain, unconditional hit.

54. Brown Pet Pteranodon - Earth | Precision Strike E | Active (5s CD)
`effect: anti_dodge_block_conditional, effectValue: 25`
**Damage:** 7 (earth) | **Cooldown:** 5s
> *Wiki ability text - Precision Strike E:* If the opponent attempts to dodge or block, hits for 25% more damage. Inflicts 7 Earth damage. Can't be dodged or blocked.

Active ability, 5s cooldown, the "Precision/Critical" dino line. Deals 7 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 25% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

55. Cinder Sprites - Earth | Multiply! | Active (5s CD)
`effect: revive, effectValue: 5, effectDuration: 5`
**Heal:** 5 | **Cooldown:** 5s
> *Wiki ability text - Multiply!:* Summon a friend for 5s. If beaten, it takes the hit instead, reviving you with 5% life.

Active ability, 5s cooldown, up to 5s cast window implied by its long cooldown. If the caster's own backup deck is currently at 0 HP, this revives it to 5% of its max HP. If the backup deck is still alive, casting this ability does nothing (no healing, no effect) - it only works as a genuine revive, not a heal.

56. Puddy Leash - Earth | Hairball | Active (5s CD)
`effect: none`
**Damage:** 10 (earth) | **Cooldown:** 5s
> *Wiki ability text - Hairball:* Launch a hairball. Gross. Inflitcts 10 Earth damage.

Active ability, 5s cooldown. Deals 10 flat damage with no secondary effect attached - a plain, unconditional hit.

57. Brown Pet Apatodon - Earth | Dino Dive E | Active (6s CD)
`effect: elemental_bonus, effectValue: 50`
**Damage:** 10 (earth) | **Cooldown:** 6s
> *Wiki ability text - Dino Dive E:* Against an Air opponent, inflicts +50% damage (on top of normal bonus.) Inflicts 10 Earth damage.

Active ability, 6s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 10 base Earth damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 50/100` (i.e. +50%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 10 damage with no bonus.

58. Mini Growtopian - Earth | Punch, Build, Grow | Active (6s CD)
`effect: stacking_build, effectValue: 4`
**Damage:** 8 (earth) | **Cooldown:** 6s
> *Wiki ability text - Punch, Build, Grow:* Punch hits for 8 Earth damage. Build builds a wall that blocks attacks for 1s. Grow adds 4 damage to Punch and 1s to Build, stacking up to 5 times.

Active ability, 6s cooldown ("Mini Growtopian"). Cycles through three sub-moves in a fixed rotation on successive casts - Punch, then Build, then Grow, then back to Punch: Punch deals `8 + growStacks × 4` damage; Build grants self-Block for `1 + growStacks` seconds; Grow adds a permanent (per-battle, capped at 5) stack that boosts both of the other two moves. The rotation position and stack count persist on the pet itself for the whole battle.

59. Rhino Horn - Earth | Charge | Active (6s CD)
`effect: self_stun, effectDuration: 3`
**Damage:** 16 (earth) | **Cooldown:** 6s
> *Wiki ability text - Charge:* Ram the enemy, stunning yourself for 3s in the process. Inflicts 16 Earth damage.

Active ability, 6s cooldown ("Rhino Horn"). Deals 16 damage to the enemy via the generic path, then stuns the CASTER'S OWN deck for 3s as recoil - it can't act, but per the general Stun rule it can still be swapped out since Stun isn't in the swap-blocking status list (only Freeze blocks swapping).

60. Unicorn Garland - Earth | Rainbow Beam | Active (6s CD)
`effect: anti_block`
**Damage:** 11 (earth) | **Cooldown:** 6s
> *Wiki ability text - Rainbow Beam:* Fire a blast of mythical colors. Inflicts 11 Earth Damage. Can't be blocked.

Active ability, 6s cooldown. Deals 11 damage. Sets `ignoresBlock` in `dealDamage`, so a Block status on the target is skipped entirely for this hit - Dodge is unaffected and can still proc normally against it.

61. Calf Leash - Earth | Trample | Active (7s CD)
`effect: force_swap`
**Damage:** 10 (earth) | **Cooldown:** 7s
> *Wiki ability text - Trample:* Force opponent to swap pets. Inflicts 10 Earth Damage.

Active ability, 7s cooldown ("Trample"). Deals 10 damage, then - if the enemy has a living, swappable, non-frozen backup deck - forces them onto it, clearing any Doom on the deck they leave. Unlike Banish, this does not lock the deck they land on against swapping back afterward.

62. Brown Pet Apatoceratops - Earth | Dino Charge E | Active (8s CD)
`effect: elemental_bonus, effectValue: 60`
**Damage:** 14 (earth) | **Cooldown:** 8s
> *Wiki ability text - Dino Charge E:* Against an Air opponent, inflicts +60% damage (on top of normal bonus). Inflicts 14 Earth damage.

Active ability, 8s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 14 base Earth damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 60/100` (i.e. +60%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 14 damage with no bonus.

63. Brown Pet Pteratops - Earth | Precision Attack E | Active (9s CD)
`effect: anti_dodge_block_conditional, effectValue: 17`
**Damage:** 11 (earth) | **Cooldown:** 9s
> *Wiki ability text - Precision Attack E:* If the opponent attempts to dodge or block, hits for 17% more damage. Inflicts 11 Earth damage. Can't be dodged or blocked.

Active ability, 9s cooldown, the "Precision/Critical" dino line. Deals 11 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 17% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

64. Brown Pet Apatos Rex - Earth | Dino Chomp E | Active (10s CD)
`effect: elemental_bonus, effectValue: 25`
**Damage:** 18 (earth) | **Cooldown:** 10s
> *Wiki ability text - Dino Chomp E:* Against an Air opponent, inflicts +25% damage (on top of normal bonus). Inflicts 18 Earth damage.

Active ability, 10s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 18 base Earth damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 25/100` (i.e. +25%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 18 damage with no bonus.

65. Brown Pet Apatosaurus - Earth | Dino Slam E | Active (10s CD)
`effect: elemental_bonus, effectValue: 100`
**Damage:** 15 (earth) | **Cooldown:** 10s
> *Wiki ability text - Dino Slam E:* Against an Air opponent, inflicts +100% damage (on top of normal bonus). Inflicts 15 Earth damage.

Active ability, 10s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 15 base Earth damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 100/100` (i.e. +100%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 15 damage with no bonus.

66. Brown Pet Triceradon - Earth | Defensive Flurry E | Active (10s CD)
`effect: block, effectDuration: 2`
**Damage:** 12 (earth) | **Cooldown:** 10s
> *Wiki ability text - Defensive Flurry E:* Strike and then block attacks for 2s. Inflicts 12 Earth damage.

Active ability, 10s cooldown. Deals 12 damage to the enemy, then grants the caster's own deck a self-Block status for 2s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

67. Brown Pet Tyranodon - Earth | Piercing Jaws E | Active (10s CD)
`effect: trauma, effectDuration: 4`
**Damage:** 15 (earth) | **Cooldown:** 10s
> *Wiki ability text - Piercing Jaws E:* Causes trauma, preventing the target from healing for 4s. Inflicts 15 Earth damage.

Active ability, 10s cooldown. Deals 15 damage, then inflicts Trauma on the target for 4s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

68. Leashed Silkworm - Grey - Earth | Stoneform | Active (10s CD)
`effect: block, effectDuration: 5`
**Cooldown:** 10s
> *Wiki ability text - Stoneform:* Turn to stone, blocking attacks for 5s.

Active ability, 10s cooldown. Deals 0 damage to the enemy, then grants the caster's own deck a self-Block status for 5s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

69. Piglet Leash - Earth | Wallow | Active (10s CD)
`effect: none`
**Heal:** 20 | **Cooldown:** 10s
> *Wiki ability text - Wallow:* Wallow in the mud, healing yourself. Heals 20 life.

Active ability, 10s cooldown. No damage, no special effect - restores 20 flat HP to the caster's own deck and nothing else.

70. Tumbleweed Attractor - Earth | Needles | Active (10s CD)
`effect: thorns, effectDuration: 15`
**Damage:** 5 (earth) | **Cooldown:** 10s
> *Wiki ability text - Needles:* For 15s, inflict 5 Earth damage when you are hit.

Active ability, 10s cooldown, no direct damage on cast. Arms a `counter_reflect` buff on the caster's deck worth 5 (using the pet's `damage` field as the reflect amount) for 15s. The next time - and every time - this deck actually takes a landed hit while the buff is active, `dealDamage` reflects that flat 5 amount straight back at the attacker's deck, on top of the damage the caster itself took. Recasting replaces any existing counter buff rather than stacking.

71. Unearthly Synthoid - Earth | Unearthly Beam | Active (10s CD)
`effect: bonus_on_debuff, effectValue: 15`
**Damage:** 10 (earth) | **Cooldown:** 10s
> *Wiki ability text - Unearthly Beam:* Shoot a beam. If the enemy has any negative effects, one is vaporized for an extra 15 damage. Inflicts 10 Earth Damage.

Active ability, 10s cooldown. The base hit lands generically first; then the code checks whether the target is carrying any negative effect - a modifier debuff first (`targetDeck.modifiers.debuffs`), or failing that, a status effect from `['stun','freeze','trauma','dot','dance','sleep']`. If one is found, it's stripped off the target and this pet deals an extra 15 bonus damage. If the target has nothing negative on it, no bonus damage is dealt and nothing is removed.

72. Zombie Hound - Earth | Bursting Spores | Active (10s CD)
`effect: delayed_damage, effectValue: 15, effectDuration: 6`
**Damage:** 10 (earth) | **Cooldown:** 10s
> *Wiki ability text - Bursting Spores:* Bite, infecting with spores that burst after 6s for 15 damage. Inflicts 10 Earth damage.

Active ability, 10s cooldown ("Zombie Hound"). The base hit lands generically, then this schedules a separate delayed burst: after 6s pass (ticked down once per game tick in `gameTick`), the target takes an additional 15 flat damage via `dealDamage` with no attacking pet reference - so it isn't affected by that pet's own dodge/block-piercing traits, only the target's current state at the moment the spores burst.

73. 14DEViL's Friendly Doggy Leash - Earth | Lockjaw! | Active (12s CD)
`effect: trap_swap, effectValue: 40`
**Damage:** 5 (earth) | **Cooldown:** 12s
> *Wiki ability text - Lockjaw!:* Bite its opponent for 5 Earth Damage holding on. If your opponent retreats, it lashes out for 40 Earth Damage. Inflicts 5 Earth Damage. Isn't visible to your enemy.

Active ability, 12s cooldown. Sets an invisible trap on the enemy - per wiki trivia, the trap itself isn't shown to the enemy (though the cast animation is, so they can see it coming), lasting a 4s window. If the enemy voluntarily swaps decks while the trap is armed, it springs for 40 bonus damage; for the specific pet whose ability is named "Surprise!", the spring hits BOTH of the enemy's decks instead of just the one swapping in.

74. Flowersaurus Rex - Earth | Pollen Barrage | Active (12s CD)
`effect: consume_buff, effectValue: 15`
**Damage:** 15 (earth) | **Cooldown:** 12s
> *Wiki ability text - Pollen Barrage:* Spew pollen. Consumes your positive effects to add 15 damage per effect. Inflicts 15 Earth damage.

Active ability, 12s cooldown ("Radiation Beam"/"Pollen Barrage"). Before dealing damage, `dealDamage` is told to ignore the caster's own buffs for this specific hit (`isConsumeBuff`), and afterward the oldest buff on the caster's own deck is deleted outright. Net effect: this attack never benefits from the caster's own damage buffs (Howl/Get Lucky/etc. are skipped for this hit), and one of those buffs is destroyed as the cost of casting - matching the wiki trivia that damage buffs are ignored specifically because they get consumed by the move.

75. Moose Cap - Earth | Antler Bash | Active (12s CD)
`effect: anti_block`
**Damage:** 22 (earth) | **Cooldown:** 12s
> *Wiki ability text - Antler Bash:* Smash them with your mighty antlers! Inflicts 22 Earth damage. Can't be blocked.

Active ability, 12s cooldown. Deals 22 damage. Sets `ignoresBlock` in `dealDamage`, so a Block status on the target is skipped entirely for this hit - Dodge is unaffected and can still proc normally against it.

76. Brown Pet Triceratops - Earth | Defensive Gore E | Active (13s CD)
`effect: block, effectDuration: 4`
**Damage:** 16 (earth) | **Cooldown:** 13s
> *Wiki ability text - Defensive Gore E:* Strike and then block attacks for 4s. Inflicts 16 Earth damage.

Active ability, 13s cooldown. Deals 16 damage to the enemy, then grants the caster's own deck a self-Block status for 4s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

77. Brown Pet Tyranotops - Earth | Crushing Beak E | Active (13s CD)
`effect: trauma, effectDuration: 7`
**Damage:** 19 (earth) | **Cooldown:** 13s
> *Wiki ability text - Crushing Beak E:* Causes Trauma, preventing the target from healing for 7 seconds. Inflicts 19 Earth damage.

Active ability, 13s cooldown. Deals 19 damage, then inflicts Trauma on the target for 7s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

78. Brown Pet Pterosaurus - Earth | Critical Strike E | Active (15s CD)
`effect: anti_dodge_block_conditional, effectValue: 50`
**Damage:** 17 (earth) | **Cooldown:** 15s
> *Wiki ability text - Critical Strike E:* If the opponent attempts to dodge or block, hits for 50% more damage. Inflicts 17 Earth damage. Can't be dodged or blocked.

Active ability, 15s cooldown, the "Precision/Critical" dino line. Deals 17 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 50% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

79. Cool Capybara - Earth | Fortunate Fame! | Active (15s CD)
`effect: dodge, effectDuration: 10`
**Cooldown:** 15s
> *Wiki ability text - Fortunate Fame!:* Dodge all attacks for 10s.

Active ability, 15s cooldown. Grants the caster's own deck a Dodge status for 10s. While Dodge is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore dodge (not anti_dodge/anti_dodge_block/mess_up_undodgeable, and not a piercing Precision hit) and reduces it to 0 before the damage floor. Per wiki trivia, at least one pet on this effect (Swoop) is guaranteed to grant its dodge window even if the cast itself was messed up by a mess_up-type debuff.

80. MickeyMay Leash - Earth | Cower | Active (15s CD)
`effect: block, effectDuration: 7`
**Cooldown:** 15s
> *Wiki ability text - Cower:* Blocks all attacks for 7s.

Active ability, 15s cooldown. Deals 0 damage to the enemy, then grants the caster's own deck a self-Block status for 7s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

81. Pet Bunny - Earth | Big Sharp Pointy Teeth | Active (15s CD)
`effect: none`
**Damage:** 30 (earth) | **Cooldown:** 15s
> *Wiki ability text - Big Sharp Pointy Teeth:* Murder violently. Inflicts 30 Earth damage.

Active ability, 15s cooldown. Deals 30 flat damage with no secondary effect attached - a plain, unconditional hit.

82. Playful Wood Sprite - Earth | Playtime | Active (15s CD)
`effect: buff_chance_double_damage, effectValue: 30, effectDuration: 4`
**Damage:** 5 (earth) | **Cooldown:** 15s
> *Wiki ability text - Playtime:* Grants you a 30% chance to do double damage for 4 seconds. Inflicts 5 Earth damage.

Active ability, 15s cooldown. Deals 5 damage, then grants the caster's own deck a `double_dmg` buff (`30%` chance) lasting 4s. In `dealDamage`, every subsequent attack from this deck while the buff is up rolls against `doubleDmgChance`; on a hit, that single attack's damage is doubled. Reapplying before it expires simply refreshes the timer to 4s (buffs from different sources stack independently rather than overwriting each other).

83. Prickly Pet - Earth | Spikey Shield | Active (15s CD)
`effect: counter, effectDuration: 8`
**Damage:** 20 (earth) | **Cooldown:** 15s
> *Wiki ability text - Spikey Shield:* Strike back for 20 Earth Damage if hit within 8s.

Active ability, 15s cooldown, no direct damage on cast. Arms a `counter_reflect` buff on the caster's deck worth 20 (using the pet's `damage` field as the reflect amount) for 8s. The next time - and every time - this deck actually takes a landed hit while the buff is active, `dealDamage` reflects that flat 20 amount straight back at the attacker's deck, on top of the damage the caster itself took. Recasting replaces any existing counter buff rather than stacking.

84. Brown Pet Tyrannosaurus - Earth | Crushing Jaws E | Active (16s CD)
`effect: trauma, effectDuration: 3`
**Damage:** 29 (earth) | **Cooldown:** 16s
> *Wiki ability text - Crushing Jaws E:* Causes trauma, preventing the target for 3s. Inflicts 29 Earth damage.

Active ability, 16s cooldown. Deals 29 damage, then inflicts Trauma on the target for 3s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

85. Brown Pet Tricerus Rex - Earth | Defensive Bite E | Active (18s CD)
`effect: block, effectDuration: 2`
**Damage:** 22 (earth) | **Cooldown:** 18s
> *Wiki ability text - Defensive Bite E:* Strike and then block attacks for 2s. Inflicts 22 Earth damage.

Active ability, 18s cooldown. Deals 22 damage to the enemy, then grants the caster's own deck a self-Block status for 2s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

86. Brown Pet Pteranus Rex - Earth | Precision Crush E | Active (19s CD)
`effect: anti_dodge_block_conditional, effectValue: 10`
**Damage:** 25 (earth) | **Cooldown:** 19s
> *Wiki ability text - Precision Crush E:* If the opponent attempts to dodge or block, hits for 10% more damage. Inflicts 25 Earth damage. Can't be dodged or blocked.

Active ability, 19s cooldown, the "Precision/Critical" dino line. Deals 25 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 10% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

87. Brown Pet Tripatosaurus - Earth | Defensive Bash E | Active (20s CD)
`effect: block, effectDuration: 7`
**Damage:** 10 (earth) | **Cooldown:** 20s
> *Wiki ability text - Defensive Bash E:* Strike and then block attacks for 7s. Inflicts 10 Earth damage

Active ability, 20s cooldown. Deals 10 damage to the enemy, then grants the caster's own deck a self-Block status for 7s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

88. Brown Pet Tyranopatos - Earth | Grinding Jaws E | Active (20s CD)
`effect: trauma, effectDuration: 30`
**Damage:** 3 (earth) | **Cooldown:** 20s
> *Wiki ability text - Grinding Jaws E:* Causes Trauma, preventing the target from healing for 30s. Inflicts 3 Earth damage.

Active ability, 20s cooldown. Deals 3 damage, then inflicts Trauma on the target for 30s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

89. Pet Leprechaun - Earth | Get Lucky | Active (20s CD)
`effect: buff_chance_double_damage, effectValue: 25, effectDuration: 10`
**Cooldown:** 20s
> *Wiki ability text - Get Lucky:* Grants 25% chance of hitting for double damage for 10s.

Active ability, 20s cooldown. Deals 0 damage, then grants the caster's own deck a `double_dmg` buff (`25%` chance) lasting 10s. In `dealDamage`, every subsequent attack from this deck while the buff is up rolls against `doubleDmgChance`; on a hit, that single attack's damage is doubled. Reapplying before it expires simply refreshes the timer to 10s (buffs from different sources stack independently rather than overwriting each other).

90. Pineapple Finger Ring - Earth | Sticky Pineapple Web | Active (20s CD)
`effect: anti_swap, effectDuration: 5, dotDamage: 4, dotDuration: 5`
**Damage:** 4 (earth) | **Cooldown:** 20s | **DoT:** 4/s for 5s
> *Wiki ability text - Sticky Pineapple Web:* Prevent opponent from swapping out for 5s, dealing 4 Earth damage every second for 5s.

Active ability, 20s cooldown, no direct damage. Applies a `no_swap` debuff to the target deck for 5s (`applyModifiers`). While active, `recomputeModifiers` sets `canSwap = false` on that deck, so its Swap button is locked (`🔒 LOCKED` in the UI) and `swapDeck()` refuses to run for the duration - the target is stuck on its current deck even if it wants to retreat.

91. Pet Clover - Earth | Get Lucky! | Active (25s CD)
`effect: buff_chance_double_damage, effectValue: 25, effectDuration: 10`
**Cooldown:** 25s
> *Wiki ability text - Get Lucky!:* Grants 25% chance of hitting double damage for 10s.

Active ability, 25s cooldown. Deals 0 damage, then grants the caster's own deck a `double_dmg` buff (`25%` chance) lasting 10s. In `dealDamage`, every subsequent attack from this deck while the buff is up rolls against `doubleDmgChance`; on a hit, that single attack's damage is doubled. Reapplying before it expires simply refreshes the timer to 10s (buffs from different sources stack independently rather than overwriting each other).

92. Dryad - Earth | Draining Roots | Active (30s CD)
`effect: life_drain`
**Damage:** 30 (earth) | **Heal:** 30 | **Cooldown:** 30s
> *Wiki ability text - Draining Roots:* Drain life from the target, healing you for the same amount. Inflicts 30 Earth damage.

Active ability, 30s cooldown. Deals 30 damage to the enemy, then - provided the hit actually connected (not dodged/blocked/absorbed) and the caster isn't Traumatized - heals the caster for exactly the amount of damage that was actually dealt (post-mitigation), not the pet's flat listed 30.

93. Wolf Tamer's Glove - Earth | Howl | Active (30s CD)
`effect: buff_damage_dealt, effectValue: 25, effectDuration: 30`
**Cooldown:** 30s
> *Wiki ability text - Howl:* Boost your damage by 25% for 30s.

Active ability, 30s cooldown. Grants the caster's own deck a `buff_damage` self-buff: +25% damage dealt on all its future attacks, lasting 30s. If this pet is one of the special stacking sources (Strawberry Slime, Leashed Silkworm - Purple, Pet Present Goblin), reapplying it before it expires adds another stack (capped at 400%) and refreshes the timer to the longer of the two durations, instead of just resetting the value.

94. Growmoji's Little Partner - Earth | Adorable Eyes... and Claw | Active (40s CD)
`effect: force_partner_forward`
**Damage:** 35 (earth) | **Cooldown:** 40s
> *Wiki ability text - Adorable Eyes... and Claw:* Fool your foe with your adorableness scratching them harshly and bringing their partner forward. Inflicts 35 Earth Damage.

Active ability, 40s cooldown. Deals 35 damage, then - per wiki trivia clarifying that "their partner" means the FOE's partner - forces the enemy to swap onto their own backup deck (provided it's alive, swappable, and not frozen), pulling that backup deck to the front. This is a forced swap on the opponent, not a self-swap by the caster.

95. Lil' Sheepers - Earth | Count Sheep | Active (40s CD)
`effect: sleep, effectDuration: 8`
**Cooldown:** 40s
> *Wiki ability text - Count Sheep:* Weather Effect. Put everybody to sleep for 8s. Replaces any other active weather effect.

Active ability, 40s cooldown ("Count Sheep"), functions as a weather move. Sets the global battlefield weather to a Sleep effect lasting 8s; every tick, BOTH sides' active decks are put to Sleep (which disables acting, same list as Stun/Freeze/Dance) for the remaining weather duration +1s, as long as they aren't already asleep. Per wiki trivia, this sleep timer is itself affected by recharge-speed modifiers on the affected deck: a slow-recharge debuff (Spit Mud/Mud Glob) extends how long that specific deck stays asleep past when the weather ends, while a fast-recharge buff (Ginger Blast/Sugar Rush) shortens it, letting that deck wake up and act again before the weather is over.

96. Muddy Pants - Earth | Mud Glob | Active (40s CD)
`effect: slow_recharge, effectValue: 50, effectDuration: 10`
**Damage:** 20 (earth) | **Cooldown:** 40s
> *Wiki ability text - Mud Glob:* For 10s, the target's abilities recharge half as fast. Inflicts 20 Earth damage.

Active ability, 40s cooldown. Deals 20 damage, then applies a `slow_cd` debuff to BOTH the target's active deck AND its benched partner deck for 10s, halving recharge speed (`speedMult *= 0.5`) for pet cooldowns and for ticking down that deck's Stun/Freeze/Sleep durations - per wiki trivia this partner-deck reach isn't stated outright but is implemented, and the CC-duration slowdown applies as well as ability recharge.

97. Leashed Silkworm - Green - Earth | Toxic Cloud (Weather Effect) | Active (60s CD)
`effect: weather, effectValue: 4, effectDuration: 60, dotDamage: 4, dotDuration: 60`
**Cooldown:** 60s | **DoT:** 4/s for 60s
> *Wiki ability text - Toxic Cloud (Weather Effect):* Spread poison over the whole battle, hurting everyone for 4 Earth damage every 5s, lasting 60s. Replaces any other active Weather Effect.

Active ability, 60s cooldown ("Toxic Cloud"). Sets the global battlefield weather for 60s: every 5 ticks, BOTH sides' current active decks take 4 flat, unmitigated damage. Same "no damage on the final tick" quirk as Firestorm applies here too, per wiki trivia.

---

### Fire

98. Hotpants - Fire | Pepper | Active (3s CD)
`effect: none`
**Damage:** 6 (fire) | **Cooldown:** 3s
> *Wiki ability text - Pepper:* Pepper your foes with this rapid-fire attack. Inflicts 6 Fire damage.

Active ability, 3s cooldown. Deals 6 flat damage with no secondary effect attached - a plain, unconditional hit.

99. Party Fowl - Fire | Party Foul | Active (5s CD)
`effect: auto_attack_inactive`
**Damage:** 5 (fire) | **Cooldown:** 5s
> *Wiki ability text - Party Foul:* Attack when inactive! 5 Fire damage attack is used automatically when inactive.

Passive-like effect (`is_passive` is false in the database, so it still has its own clickable ability too), but on top of that: every time this pet's deck is NOT the currently selected active pet - whether because a teammate in the same deck is active, or the whole deck is benched - it automatically pecks the enemy for 5 flat damage every 5s (`battle.tick % cooldown === 0`), completely independent of its own ability cooldown.

100. Red Pet Pteranodon - Fire | Dino Dive F | Active (5s CD)
`effect: anti_dodge_block_conditional, effectValue: 25`
**Damage:** 7 (fire) | **Cooldown:** 5s
> *Wiki ability text - Dino Dive F:* If the opponent attempts to dodge or block, hits for 25% more damage. Inflicts 7 Fire damage. Can't be dodged or blocked.

Active ability, 5s cooldown, the "Precision/Critical" dino line. Deals 7 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 25% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

101. Dragon Hand - Fire | Fire Breath | Active (6s CD)
`effect: dot, effectValue: 3, effectDuration: 4, dotDamage: 3, dotDuration: 4`
**Damage:** 3 (fire) | **Cooldown:** 6s | **DoT:** 3/s for 4s
> *Wiki ability text - Fire Breath:* Ignite the target for 3 Fire damage plus 3 more per second for 4 seconds. Inflicts 3 Fire damage.

Active ability, 6s cooldown. Applies a damage-over-time status to the target dealing 3 damage per tick for 4s (subject to the passive_resist_negative resist check and the passive_shorten_debuff duration reduction, same as any other negative effect). Per wiki trivia, DoTs (like all timed effects) don't tick on their very last second of duration, so the realized total damage over the full window ends up one tick short of the naive `3 × 4` calculation.

102. Phlogiston - Fire | Toasties | Active (6s CD)
`effect: summon, dotDamage: 2, dotDuration: 3`
**Damage:** 2 (fire) | **Cooldown:** 6s | **DoT:** 2/s for 3s
> *Wiki ability text - Toasties:* Summons a toasty friend who inflicts 2 Fire damage every 3s.

Active ability, 6s cooldown ("Toasties"). Deals 2 damage that completely bypasses `dealDamage` - it's subtracted from the target's HP directly, so it ignores elemental advantage, Purple Haze, dodge/block, Ethereal, and every damage buff/debuff in the game (matching wiki trivia that Toasties "ignore any damage modifying effects"). If the pet also carries a DoT component (`dotDuration > 0`), it additionally applies a burn ticking 2 per second for 3s, which IS a normal DoT and does interact with modifiers normally on its own tick.

103. Red Pet Apatodon - Fire | Precision Strike F | Active (6s CD)
`effect: elemental_bonus, effectValue: 50`
**Damage:** 10 (fire) | **Cooldown:** 6s
> *Wiki ability text - Precision Strike F:* Against an Earth opponent, inflicts +50% damage (on top of normal bonus.) Inflicts 10 Fire damage.

Active ability, 6s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 10 base Fire damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 50/100` (i.e. +50%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 10 damage with no bonus.

104. Lantern's ChronoReaper - Fire | To the Shadow Realm! | Active (7s CD)
`effect: banish, effectDuration: 10`
**Damage:** 10 (fire) | **Cooldown:** 7s
> *Wiki ability text - To the Shadow Realm!:* Open a portal dealing 10 Fire Damage to the enemy banishing them for 10s!

Active ability, 7s cooldown. Deals 10 damage, then - if the enemy has a living backup deck - forcibly swaps them onto it (clearing any Doom on the deck they leave) and slaps a `no_swap` debuff on the deck they land on for 10s, locking `canSwap = false` so they can't swap back for the duration. If the enemy has no living backup deck to be pushed onto, the ability fizzles with no forced swap (but the direct damage still lands).

105. Pet Fox - Fire | Desperate Bite | Active (7s CD)
`effect: desperate`
**Damage:** 10 (fire) | **Cooldown:** 7s
> *Wiki ability text - Desperate Bite:* Inflicts 1% more damage for every 1% of life you are missing. Deals 10 Fire damage.

Active ability, 7s cooldown ("Desperate Bite" - Pet Fox). Damage scales with how much HP the caster's OWN deck is missing: the code computes `missingFrac = 1 - currentHP/maxHP` and deals `10 × (1 + missingFrac)`, so at full HP it just deals its flat 10, ramping up to roughly double at 0 HP. The wiki doesn't give exact scaling numbers for this one (the database's `effectValue` is 0), so this curve is the simulator's own reasonable default rather than a confirmed wiki figure.

106. Playful Fire Sprite - Fire | Playful Fire | Active (8s CD)
`effect: buff_chance_double_damage, effectValue: 10, effectDuration: 5`
**Damage:** 15 (fire) | **Cooldown:** 8s
> *Wiki ability text - Playful Fire:* Grants you a 10% chance to do double damage for 5 seconds. Inflicts 15 Fire damage.

Active ability, 8s cooldown. Deals 15 damage, then grants the caster's own deck a `double_dmg` buff (`10%` chance) lasting 5s. In `dealDamage`, every subsequent attack from this deck while the buff is up rolls against `doubleDmgChance`; on a hit, that single attack's damage is doubled. Reapplying before it expires simply refreshes the timer to 5s (buffs from different sources stack independently rather than overwriting each other).

107. Red Pet Apatoceratops - Fire | Dino Charge F | Active (8s CD)
`effect: elemental_bonus, effectValue: 60`
**Damage:** 14 (fire) | **Cooldown:** 8s
> *Wiki ability text - Dino Charge F:* Against an Earth opponent, inflicts +60% damage (on top of normal bonus). Inflicts 14 Fire damage.

Active ability, 8s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 14 base Fire damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 60/100` (i.e. +60%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 14 damage with no bonus.

108. Red Pet Pteratops - Fire | Precision Attack F | Active (9s CD)
`effect: anti_dodge_block_conditional, effectValue: 17`
**Damage:** 11 (fire) | **Cooldown:** 9s
> *Wiki ability text - Precision Attack F:* If the opponent attempts to dodge or block, hits for 17% more damage. Inflicts 11 Fire damage. Can't be dodged or blocked.

Active ability, 9s cooldown, the "Precision/Critical" dino line. Deals 11 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 17% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

109. Fiesta Dragon - Fire | Party Breath | Active (10s CD)
`effect: hit_both`
**Damage:** 10 (fire) | **Cooldown:** 10s
> *Wiki ability text - Party Breath:* Attack, hitting both enemy pets at once. Inflicts 10 Fire damage to each one.

Active ability, 10s cooldown. The base 10 damage lands on the enemy's active deck generically; if their backup deck is also still alive, this pet then deals the same 10 damage a second time directly to that backup deck as well, hitting both of the enemy's decks in one cast.

110. Red Pet Apatos Rex - Fire | Dino Chomp F | Active (10s CD)
`effect: elemental_bonus, effectValue: 25`
**Damage:** 18 (fire) | **Cooldown:** 10s
> *Wiki ability text - Dino Chomp F:* Against an Earth opponent, inflicts +25% damage (on top of normal bonus). Inflicts 18 Fire damage.

Active ability, 10s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 18 base Fire damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 25/100` (i.e. +25%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 18 damage with no bonus.

111. Red Pet Apatosaurus - Fire | Dino Slam F | Active (10s CD)
`effect: elemental_bonus, effectValue: 100`
**Damage:** 15 (fire) | **Cooldown:** 10s
> *Wiki ability text - Dino Slam F:* Against an Earth opponent, inflicts +100% damage (on top of normal bonus). Inflicts 15 Fire damage.

Active ability, 10s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 15 base Fire damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 100/100` (i.e. +100%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 15 damage with no bonus.

112. Red Pet Triceradon - Fire | Defensive Flurry F | Active (10s CD)
`effect: block, effectDuration: 2`
**Damage:** 12 (fire) | **Cooldown:** 10s
> *Wiki ability text - Defensive Flurry F:* Strike and then block attacks for 2s. Inflicts 12 Fire damage.

Active ability, 10s cooldown. Deals 12 damage to the enemy, then grants the caster's own deck a self-Block status for 2s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

113. Red Pet Tyranodon - Fire | Piercing Jaws F | Active (10s CD)
`effect: trauma, effectDuration: 4`
**Damage:** 15 (fire) | **Cooldown:** 10s
> *Wiki ability text - Piercing Jaws F:* Causes trauma, preventing the target from healing for 4s. Inflicts 15 Fire damage.

Active ability, 10s cooldown. Deals 15 damage, then inflicts Trauma on the target for 4s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

114. Skeletal Dragon Claw - Fire | Deathfire | Active (10s CD)
`effect: life_drain`
**Damage:** 10 (fire) | **Heal:** 10 | **Cooldown:** 10s
> *Wiki ability text - Deathfire:* Drain life from the target, healing you for the same amount. Deals 10 Fire damage.

Active ability, 10s cooldown. Deals 10 damage to the enemy, then - provided the hit actually connected (not dodged/blocked/absorbed) and the caster isn't Traumatized - heals the caster for exactly the amount of damage that was actually dealt (post-mitigation), not the pet's flat listed 10.

115. Toy Lock-Bot - Fire | Lockdown | Active (10s CD)
`effect: anti_swap, effectDuration: 4`
**Damage:** 17 (fire) | **Cooldown:** 10s
> *Wiki ability text - Lockdown:* Smack the target with a lock, keeping them from swapping out for 4s. Deals 17 Fire damage.

Active ability, 10s cooldown, no direct damage. Applies a `no_swap` debuff to the target deck for 4s (`applyModifiers`). While active, `recomputeModifiers` sets `canSwap = false` on that deck, so its Swap button is locked (`🔒 LOCKED` in the UI) and `swapDeck()` refuses to run for the duration - the target is stuck on its current deck even if it wants to retreat.

116. Magic Reindeer Bell - Fire | Glowing Nose | Active (12s CD)
`effect: dot, effectValue: 2, effectDuration: 10, dotDamage: 2, dotDuration: 10`
**Damage:** 10 (fire) | **Cooldown:** 12s | **DoT:** 2/s for 10s
> *Wiki ability text - Glowing Nose:* Ignite them with rays from your nose, doing 2 Fire damage per second for 10s. inflicts 10 Fire damage.

Active ability, 12s cooldown. Applies a damage-over-time status to the target dealing 2 damage per tick for 10s (subject to the passive_resist_negative resist check and the passive_shorten_debuff duration reduction, same as any other negative effect). Per wiki trivia, DoTs (like all timed effects) don't tick on their very last second of duration, so the realized total damage over the full window ends up one tick short of the naive `2 × 10` calculation.

117. Leashed Silkworm - Red - Fire | Death Ray | Active (13s CD)
`effect: chain_on_kill`
**Damage:** 24 (fire) | **Cooldown:** 13s
> *Wiki ability text - Death Ray:* If this beat destroys its target, it also hits the target's partner. Inflicts 24 Fire damage.

Active ability, 13s cooldown ("Death Ray"). The base 24 damage hits the enemy's active deck generically first; if that hit brought the target deck's HP to 0 or below, this pet immediately fires the same 24 damage into the enemy's OTHER (backup) deck too, provided that deck is still alive. If the first hit didn't finish the active deck off, there's no chain - it's strictly a finishing-blow bonus, not a guaranteed second hit.

118. Mechasaurus Rex - Fire | Ratatata! | Active (13s CD)
`effect: none`
**Damage:** 30 (fire) | **Cooldown:** 13s
> *Wiki ability text - Ratatata!:* Fires several shots dealing 30 Fire Damage! Inflicts 30 Fire damage.

Active ability, 13s cooldown. Deals 30 flat damage with no secondary effect attached - a plain, unconditional hit.

119. Radioactive Synthoid - Fire | Radiation Beam | Active (13s CD)
`effect: consume_buff`
**Damage:** 30 (fire) | **Cooldown:** 13s
> *Wiki ability text - Radiation Beam:* Fizzles unless you have a positive effect on you. Consumes the effect to fire a beam. Inflicts 30 Fire damage.

Active ability, 13s cooldown ("Radiation Beam"/"Pollen Barrage"). Before dealing damage, `dealDamage` is told to ignore the caster's own buffs for this specific hit (`isConsumeBuff`), and afterward the oldest buff on the caster's own deck is deleted outright. Net effect: this attack never benefits from the caster's own damage buffs (Howl/Get Lucky/etc. are skipped for this hit), and one of those buffs is destroyed as the cost of casting - matching the wiki trivia that damage buffs are ignored specifically because they get consumed by the move.

120. Red Pet Triceratops - Fire | Defensive Gore F | Active (13s CD)
`effect: block, effectDuration: 4`
**Damage:** 16 (fire) | **Cooldown:** 13s
> *Wiki ability text - Defensive Gore F:* Strike and then block attacks for 4s. Inflicts 16 Fire damage.

Active ability, 13s cooldown. Deals 16 damage to the enemy, then grants the caster's own deck a self-Block status for 4s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

121. Red Pet Tyranotops - Fire | Crushing Beak F | Active (13s CD)
`effect: trauma, effectDuration: 7`
**Damage:** 19 (fire) | **Cooldown:** 13s
> *Wiki ability text - Crushing Beak F:* Causes Trauma, preventing the target from healing for 7 seconds. Inflicts 19 Fire damage.

Active ability, 13s cooldown. Deals 19 damage, then inflicts Trauma on the target for 7s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

122. Demon Control Cap - Fire | Flaming Tongue | Active (15s CD)
`effect: self_dot, effectValue: 5, effectDuration: 7, dotDamage: 5, dotDuration: 7`
**Heal:** 40 | **Cooldown:** 15s | **DoT:** 5/s for 7s
> *Wiki ability text - Flaming Tongue:* Lick your wounds, but then you burn 5 Fire damage per second for 7s. Heal 40 life.

Active ability, 15s cooldown ("Flaming Tongue"). Despite the database flagging this move `isDot: true` (so it ticks on the normal DoT clock), the burn is applied to the CASTER, not the enemy: 5 damage per second for 7s on the caster's own deck. This is checked before the generic DoT branch specifically so it doesn't get misapplied as damage to the target instead.

123. Leashed Silkworm - Black - Fire | Absorb | Active (15s CD)
`effect: absorb, effectDuration: 2`
**Cooldown:** 15s
> *Wiki ability text - Absorb:* Absorb incoming energy for 2s - any attacks that would hurt you will heal you instead.

Active ability, 15s cooldown. No direct damage. Raises an Absorb shield on the caster's own deck for 2s. While up, the very next hit(s) this deck would take are fully nullified (0 damage taken) in `dealDamage`, and - unless the deck is also carrying Trauma - the deck heals for exactly the amount of damage that would have been dealt. If Trauma is present, the damage is still nullified but the healing is blocked. Per wiki trivia, this move shares a "move class" with the other Absorb-type pet: recasting one before the other expires EXTENDS the remaining duration rather than resetting it, and one of the two variants reuses the same 2-second cast animation even though its actual shield window is longer, so there's no visual cue for the remaining time beyond the animation.

124. Lion Taming Whip - Fire | Claws Out | Active (15s CD)
`effect: counter, effectDuration: 4`
**Damage:** 35 (fire) | **Cooldown:** 15s
> *Wiki ability text - Claws Out:* Strike back for 35 Fire damage if hit within 4s.

Active ability, 15s cooldown, no direct damage on cast. Arms a `counter_reflect` buff on the caster's deck worth 35 (using the pet's `damage` field as the reflect amount) for 4s. The next time - and every time - this deck actually takes a landed hit while the buff is active, `dealDamage` reflects that flat 35 amount straight back at the attacker's deck, on top of the damage the caster itself took. Recasting replaces any existing counter buff rather than stacking.

125. Red Pet Pterosaurus - Fire | Critical Strike F | Active (15s CD)
`effect: anti_dodge_block_conditional, effectValue: 50`
**Damage:** 17 (fire) | **Cooldown:** 15s
> *Wiki ability text - Critical Strike F:* If the opponent attempts to dodge or block, hits for 50% more damage. Inflicts 17 Fire damage. Can't be dodged or blocked.

Active ability, 15s cooldown, the "Precision/Critical" dino line. Deals 17 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 50% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

126. Red Pet Tyrannosaurus - Fire | Crushing Jaws F | Active (16s CD)
`effect: trauma, effectDuration: 3`
**Damage:** 29 (fire) | **Cooldown:** 16s
> *Wiki ability text - Crushing Jaws F:* Causes trauma, preventing the target for 3s. Inflicts 29 Fire damage.

Active ability, 16s cooldown. Deals 29 damage, then inflicts Trauma on the target for 3s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

127. Red Pet Tricerus Rex - Fire | Defensive Bite F | Active (18s CD)
`effect: block, effectDuration: 2`
**Damage:** 22 (fire) | **Cooldown:** 18s
> *Wiki ability text - Defensive Bite F:* Strike and then block attacks for 2s. Inflicts 22 Fire damage.

Active ability, 18s cooldown. Deals 22 damage to the enemy, then grants the caster's own deck a self-Block status for 2s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

128. Red Pet Pteranus Rex - Fire | Precision Crush F | Active (19s CD)
`effect: anti_dodge_block_conditional, effectValue: 10`
**Damage:** 25 (fire) | **Cooldown:** 19s
> *Wiki ability text - Precision Crush F:* If the opponent attempts to dodge or block, hits for 10% more damage. Inflicts 25 Fire damage. Can't be dodged or blocked.

Active ability, 19s cooldown, the "Precision/Critical" dino line. Deals 25 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 10% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

129. Battle Mutant Fish - Fire | Acid Spit | Active (20s CD)
`effect: dot, effectValue: 10, effectDuration: 5, dotDamage: 10, dotDuration: 5`
**Cooldown:** 20s | **DoT:** 10/s for 5s
> *Wiki ability text - Acid Spit:* Spray acid that burns the target for 10 Fire damage per second lasting for 5 seconds.

Active ability, 20s cooldown. Applies a damage-over-time status to the target dealing 10 damage per tick for 5s (subject to the passive_resist_negative resist check and the passive_shorten_debuff duration reduction, same as any other negative effect). Per wiki trivia, DoTs (like all timed effects) don't tick on their very last second of duration, so the realized total damage over the full window ends up one tick short of the naive `10 × 5` calculation.

130. Leashed Silkworm - Yellow - Fire | Liquify | Active (20s CD)
`effect: debuff_damage_taken, effectValue: 25, effectDuration: 5`
**Damage:** 15 (fire) | **Cooldown:** 20s
> *Wiki ability text - Liquify:* Melt down the enemy so they take 25% more damage for 5s. Inflicts 15 Fire Damage.

Active ability, 20s cooldown. Deals 15 damage, then applies a `dmg_taken` debuff to the target for 5s: +25% to all damage that deck takes while it's up. Stacks per-source rather than as a flat overwrite if reapplied by one of the designated stacking pets, capped at +200%.

131. Nightmare Magnifying Glass - Fire | Nightmares | Active (20s CD)
`effect: mess_up, effectValue: 50, effectDuration: 6`
**Damage:** 10 (fire) | **Cooldown:** 20s
> *Wiki ability text - Nightmares:* Terrify the target into messing up their skills 50% of the time for 6s. Inflicts 10 Fire damage.

Active ability, 20s cooldown. Deals 10 damage, then applies a `mess_up` debuff to the target for 6s worth 50% chance. While active, every time the debuffed deck tries to use an ability, there's a roll against that chance for the cast to simply fail outright (`useAbility` rolls it first, before anything else happens) - the pet still goes on cooldown as if it had cast, but no damage/heal/effect from the intended move goes out. Weather-type moves and Swoop's dodge grant are both explicitly carved out to still trigger even on a messed-up cast, per wiki trivia.

132. Red Pet Tripatosaurus - Fire | Defensive Bash F | Active (20s CD)
`effect: block, effectDuration: 7`
**Damage:** 10 (fire) | **Cooldown:** 20s
> *Wiki ability text - Defensive Bash F:* Strike and then block attacks for 7s. Inflicts 10 Fire damage

Active ability, 20s cooldown. Deals 10 damage to the enemy, then grants the caster's own deck a self-Block status for 7s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

133. Red Pet Tyranopatos - Fire | Grinding Jaws F | Active (20s CD)
`effect: trauma, effectDuration: 30`
**Damage:** 3 (fire) | **Cooldown:** 20s
> *Wiki ability text - Grinding Jaws F:* Causes Trauma, preventing the target from healing for 30s. Inflicts 3 Fire damage.

Active ability, 20s cooldown. Deals 3 damage, then inflicts Trauma on the target for 30s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

134. Retro Magnifying Glass - Fire | Disco Fever | Active (20s CD)
`effect: force_dance, effectDuration: 4, dotDamage: 5, dotDuration: 4`
**Damage:** 5 (fire) | **Cooldown:** 20s | **DoT:** 5/s for 4s
> *Wiki ability text - Disco Fever:* Infect the target to dance for 4s, unable to do anything other than swap out, and suffering 5 Fire damage per second.

Active ability, 20s cooldown ("Enthrall"/"Disco Fever"/"Just Dance"). Deals 5 damage (if any) and locks the target into a Dance status for 4s, during which `canAct` is set false - the target can't use any ability at all, though it can still be swapped out normally (Dance isn't in the swap-blocking status list). If this specific move is "Disco Fever", it additionally grants the target a matching Dodge status for the same duration - per wiki trivia, that's the source of the "dance" flavor of the debuff forcing a dodge window on the victim, not the caster.

135. Royal Captain Bark - Fire | Supercharge | Active (20s CD)
`effect: buff_next_damage, effectValue: 80`
**Cooldown:** 20s
> *Wiki ability text - Supercharge:* Charge up your next skill with +80% Damage.

Active ability, 20s cooldown, no direct damage. Grants a `next_dmg` buff worth +80% that never expires from a timer (internally given a 99s duration) - it's consumed the moment the caster's deck lands its next hit: `dealDamage` applies the bonus, then immediately strips the buff back off, so it only ever affects exactly one attack no matter how long you wait to use it.

136. Spiritfire Mask - Fire | Spirit Swap | Active (20s CD)
`effect: force_swap_transfer`
**Damage:** 10 (fire) | **Cooldown:** 20s
> *Wiki ability text - Spirit Swap:* Force opponent to swap and transfer all status effects to the new enemy. Inflicts 10 Fire damage.

Active ability, 20s cooldown ("Spirit Swap"). Deals 10 damage, then - if the enemy can be forced to swap - every status effect and every buff/debuff currently sitting on their active deck is transferred wholesale onto the deck they're being pushed into (the active deck's slate is wiped clean), and only then is the forced swap executed. The enemy effectively drags all their baggage onto their backup deck instead of leaving it behind.

137. Pinata Pal - Fire | Surprise! | Active (22s CD)
`effect: trap_swap, effectDuration: 4`
**Damage:** 25 (fire) | **Cooldown:** 22s
> *Wiki ability text - Surprise!:* Prepare a surprise: If the opponent swaps out within 4s, hit both enemies for 25 Fire damage. Isn't visible to your enemy.

Active ability, 22s cooldown. Sets an invisible trap on the enemy - per wiki trivia, the trap itself isn't shown to the enemy (though the cast animation is, so they can see it coming), lasting a 4s window. If the enemy voluntarily swaps decks while the trap is armed, it springs for 25 bonus damage; for the specific pet whose ability is named "Surprise!", the spring hits BOTH of the enemy's decks instead of just the one swapping in.

138. Captain Bark - Fire | Supercharge | Active (30s CD)
`effect: buff_next_damage, effectValue: 50`
**Cooldown:** 30s
> *Wiki ability text - Supercharge:* Charge up your next skill with +50% Damage.

Active ability, 30s cooldown, no direct damage. Grants a `next_dmg` buff worth +50% that never expires from a timer (internally given a 99s duration) - it's consumed the moment the caster's deck lands its next hit: `dealDamage` applies the bonus, then immediately strips the buff back off, so it only ever affects exactly one attack no matter how long you wait to use it.

139. Gingerbread Man - Fire | Ginger Blast | Active (30s CD)
`effect: buff_damage_and_speed, effectValue: 25, effectDuration: 5`
**Cooldown:** 30s
> *Wiki ability text - Ginger Blast:* Deals 25% more damage. Cooldowns go twice as fast.

Active ability, 30s cooldown, no direct damage ("Ginger Blast"/"Sugar Rush" line). Grants the caster's own deck AND its benched partner deck a `dmg_speed` buff for 5s: +25% damage dealt, plus a recharge-speed multiplier that's doubled (`speedMult *= 2`) for every pet's cooldown countdown and for the remaining duration of Stun/Freeze/Sleep on that deck - per wiki trivia, this buff affects the partner deck too even though it isn't stated outright.

140. Maidmare - Fire | Slime Kiss | Active (30s CD)
`effect: mess_up, effectValue: 100, effectDuration: 10`
**Cooldown:** 30s
> *Wiki ability text - Slime Kiss:* Terrify their target into messing up their skills 100% of the time for 10s.

Active ability, 30s cooldown. Deals 0 damage, then applies a `mess_up` debuff to the target for 10s worth 100% chance. While active, every time the debuffed deck tries to use an ability, there's a roll against that chance for the cast to simply fail outright (`useAbility` rolls it first, before anything else happens) - the pet still goes on cooldown as if it had cast, but no damage/heal/effect from the intended move goes out. Weather-type moves and Swoop's dodge grant are both explicitly carved out to still trigger even on a messed-up cast, per wiki trivia.

141. Violet Protodrake Leash - Fire | Purple Haze | Active (30s CD)
`effect: weather, effectValue: 25, effectDuration: 10`
**Cooldown:** 30s
> *Wiki ability text - Purple Haze:* Weather Effect. Cover the world in burning haze for 10s, increasing all Fire damage by 25%, and reducing all other damage by 25%. Replaces any other active Weather Effect.

Active ability, 30s cooldown ("Purple Haze"). Sets the global battlefield weather for 10s with no direct damage. While active, every hit resolved in `dealDamage` is modified by the attacker's element: Fire-element attackers deal +25% damage, and every other element deals -25% damage - applied before dodge/block/Ethereal checks and stacking with normal elemental advantage. Per wiki trivia, Purple Haze is the one weather effect explicitly exempt from the "Toasties ignore all damage modifiers" rule - Toasties damage is still adjusted by Purple Haze.

142. Dragon of Legend - Fire | BURNINATE! | Active (40s CD)
`effect: anti_dodge_block`
**Damage:** 40 (fire) | **Cooldown:** 40s
> *Wiki ability text - BURNINATE!:* Vaporize the target with a massive stream of flame. Inflicts 40 Fire damage. Can't be dodged or blocked.

Active ability, 40s cooldown. Deals 40 damage. Sets both `ignoresDodge` and `ignoresBlock` in `dealDamage` directly (not via the conditional-Precision path), so this hit pierces both a temporary Dodge and a temporary Block status unconditionally. Unlike the Precision/Critical dino line, this ignores Soaring too - there's no carve-out for passive_dodge in this branch.

143. Marshmallow Basket - Fire | Sugar Rush | Active (60s CD)
`effect: fast_recharge, effectValue: 100, effectDuration: 10`
**Cooldown:** 60s
> *Wiki ability text - Sugar Rush:* For 10s, your abilities recharge twice as fast for both your pets.

Active ability, 60s cooldown ("Sugar Rush"). Grants the caster's own deck AND its benched partner deck a `fast_cd` buff for 10s that doubles recharge speed (`speedMult *= 2`) for pet cooldowns and for ticking down Stun/Freeze/Sleep durations on the affected deck - per wiki trivia this benefit extends to the partner deck even though the ability text doesn't say so explicitly.

144. Phoenix Pacifier - Fire | Firestorm | Active (60s CD)
`effect: weather, effectValue: 8, effectDuration: 10, dotDamage: 8, dotDuration: 10`
**Cooldown:** 60s | **DoT:** 8/s for 10s
> *Wiki ability text - Firestorm:* Weather Effect. Ignite the planet! Everybody burns for 8 Fire damage every 2s, lasting 10s. Replaces any other active Weather Effect.

Active ability, 60s cooldown ("Firestorm"). Sets the global battlefield weather for 10s: every 2 ticks, BOTH sides' current active decks take 8 flat, unmitigated damage (applied directly to HP, bypassing `dealDamage`). Per wiki trivia, weather doesn't deal damage on its very last tick before expiring, so a nominal `8 × 10` calculation actually lands a bit less in practice (8 damage short over a 10s Firestorm, per the wiki's own worked example).

---

### Water

145. Mini Mallard - Water | Duck's Back | Passive
`effect: passive_resist_negative, effectDuration: 30`
> *Wiki ability text - Passive - Automatically resist a negative effect completely. This effect recharges every 30s.
|:* -'''

Always active with an internal 30s cooldown (`duckBackCd`, defaults to 30 if unset). The next negative effect this deck would receive - any Stun/Freeze/Trauma via `applyStatusEffect`, any debuff via `applyModifiers`, or a DoT - is entirely blocked (`tryResist`) and the passive goes on cooldown. Because weather effects count as negative effects per wiki trivia, they can also trigger this resist (and by symmetry, they can be purged by a cleanse-type move).

146. Living Dead Remote - Water | Dawn of the Dead | Passive
`effect: passive_revive, effectValue: 30`
**Heal:** 30
> *Wiki ability text - Dawn of the Dead:* Passive - Revive dead partner with 30% health on death.

A once-per-battle failsafe that only works from the bench. `applyPassives`/`gameTick` never check this pet while it's the active one - it has to be sitting in the OTHER (inactive) deck. The instant that deck's active partner hits 0 HP (checked via `tryLivingDeadRevive` at the moment of death, not at a later "both dead" check), and provided the Remote's own benched deck is still alive, it revives the just-died deck in place to 30% of its max HP - no deck swap happens, the revived deck simply stays active. Once triggered, `livingDeadUsed` locks it out for the rest of the battle regardless of how many copies of this pet are fielded, and its passive icon shows a "USED" state.

147. Pet Present Goblin - Water | Face Slap | Active (4s CD)
`effect: debuff_damage_dealt, effectValue: 20, effectDuration: 15`
**Damage:** 10 (water) | **Cooldown:** 4s
> *Wiki ability text - Face Slap:* Smack the target in the face with a tennis ball, enraging them to inflict 20% more Damage for 15s. The rage stacks up to 20 times. Inflicts 10 Water damage.

Active ability, 4s cooldown ("Face Slap" line). Deals 10 damage, then applies a `debuff_damage` debuff to the TARGET (not the caster) for 15s: -20% to whatever damage the target deals while it's up. Per wiki trivia this is phrased as "enraging them" - it's a debuff placed on the enemy, not a self-buff on the caster. If this pet is one of the stacking sources (Leashed Silkworm - Purple, Pet Present Goblin), repeated casts stack the reduction (capped at 400%) instead of resetting it.

148. Strawberry Slime - Water | Corrosion | Active (4s CD)
`effect: buff_damage_dealt, effectValue: 10, effectDuration: 10`
**Damage:** 6 (water) | **Cooldown:** 4s
> *Wiki ability text - Corrosion:* Spray acid that increases damage given by 10% for 10s. Stacks up to 20 times. Inflicts 6 Water damage.

Active ability, 4s cooldown. Grants the caster's own deck a `buff_damage` self-buff: +10% damage dealt on all its future attacks, lasting 10s. If this pet is one of the special stacking sources (Strawberry Slime, Leashed Silkworm - Purple, Pet Present Goblin), reapplying it before it expires adds another stack (capped at 400%) and refreshes the timer to the longer of the two durations, instead of just resetting the value.

149. Green Pet Pteranodon - Water | Precision Strike W | Active (5s CD)
`effect: anti_dodge_block_conditional, effectValue: 25`
**Damage:** 7 (water) | **Cooldown:** 5s
> *Wiki ability text - Precision Strike W:* If the opponent attempts to dodge or block, hits for 25% more damage. Inflicts 7 Water damage. Can't be dodged or blocked.

Active ability, 5s cooldown, the "Precision/Critical" dino line. Deals 7 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 25% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

150. Penguin Leash - Water | Frosty Slide | Active (5s CD)
`effect: extend_cooldown, effectValue: 2, effectDuration: 5`
**Damage:** 9 (water) | **Cooldown:** 5s
> *Wiki ability text - Frosty Slide:* Chill the target, making its abilities take 2s longer to recharge if used in the next 5s. Inflicts 9 Water damage.

Active ability, 5s cooldown. Two behaviors depending on the target's state: if their active pet is already mid-recharge, this directly adds 2s onto that pet's remaining cooldown right now. If their active pet is instead already off cooldown (ready), this arms an `extend_cd` debuff lasting 5s that adds 2s of extra cooldown the very next time that deck's active pet finishes casting anything, then consumes itself.

151. Green Pet Apatodon - Water | Dino Dive W | Active (6s CD)
`effect: elemental_bonus, effectValue: 50`
**Damage:** 10 (water) | **Cooldown:** 6s
> *Wiki ability text - Dino Dive W:* Against a Fire opponent, inflicts +50% damage (on top of normal bonus.) Inflicts 10 Water damage.

Active ability, 6s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 10 base Water damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 50/100` (i.e. +50%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 10 damage with no bonus.

152. Playful Water Sprite - Water | Playful Water | Active (6s CD)
`effect: buff_chance_double_damage, effectValue: 20, effectDuration: 4`
**Damage:** 10 (water) | **Cooldown:** 6s
> *Wiki ability text - Playful Water:* Grants you a 20% chance to do double damage for 4 seconds. Inflicts 10 Water damage.

Active ability, 6s cooldown. Deals 10 damage, then grants the caster's own deck a `double_dmg` buff (`20%` chance) lasting 4s. In `dealDamage`, every subsequent attack from this deck while the buff is up rolls against `doubleDmgChance`; on a hit, that single attack's damage is doubled. Reapplying before it expires simply refreshes the timer to 4s (buffs from different sources stack independently rather than overwriting each other).

153. Polar Bear Leash - Water | Maul | Active (7s CD)
`effect: none`
**Damage:** 14 (water) | **Cooldown:** 7s
> *Wiki ability text - Maul:* Smash and slash! Inflicts 14 Water damage.

Active ability, 7s cooldown. Deals 14 flat damage with no secondary effect attached - a plain, unconditional hit.

154. Black Crystal Dragon - Water | Crystal Spikes | Active (8s CD)
`effect: anti_block`
**Damage:** 15 (water) | **Cooldown:** 8s
> *Wiki ability text - Crystal Spikes:* A blast of razor-sharp crystals. Inflicts 15 Water damage. Can't be blocked.

Active ability, 8s cooldown. Deals 15 damage. Sets `ignoresBlock` in `dealDamage`, so a Block status on the target is skipped entirely for this hit - Dodge is unaffected and can still proc normally against it.

155. Green Pet Apatoceratops - Water | Dino Charge W | Active (8s CD)
`effect: elemental_bonus, effectValue: 60`
**Damage:** 14 (water) | **Cooldown:** 8s
> *Wiki ability text - Dino Charge W:* Against a Fire opponent, inflicts +60% damage (on top of normal bonus). Inflicts 14 Water damage.

Active ability, 8s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 14 base Water damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 60/100` (i.e. +60%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 14 damage with no bonus.

156. Green Pet Pteratops - Water | Precision Attack W | Active (9s CD)
`effect: anti_dodge_block_conditional, effectValue: 17`
**Damage:** 11 (water) | **Cooldown:** 9s
> *Wiki ability text - Precision Attack W:* If the opponent attempts to dodge or block, hits for 17% more damage. Inflicts 11 Water damage. Can't be dodged or blocked.

Active ability, 9s cooldown, the "Precision/Critical" dino line. Deals 11 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 17% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

157. Battle Gar - Water | Snappy Jaws | Active (10s CD)
`effect: trauma, effectDuration: 10`
**Damage:** 15 (water) | **Cooldown:** 10s
> *Wiki ability text - Snappy Jaws:* Causes trauma and prevents the opponent from healing for 10s. Inflicts 15 Water damage.

Active ability, 10s cooldown. Deals 15 damage, then inflicts Trauma on the target for 10s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

158. Familiar Leash - Water | Black Magic | Active (10s CD)
`effect: mess_up_undodgeable, effectValue: 10, effectDuration: 20`
**Damage:** 10 (water) | **Cooldown:** 10s
> *Wiki ability text - Black Magic:* Makes your target's abilities fail 10% of the time for the next 20s. Inflicts 10 Water Damage that can't be dodged.

Active ability, 10s cooldown. Deals 10 damage that can't be dodged (this effect is on the `ignoresDodge` list in `dealDamage`, though Block still works against it), then applies the same kind of `mess_up` debuff as above - 10% ability-fail chance for 20s - the only difference from the plain mess_up pets is that this one's own damage can't be sidestepped by Dodge.

159. Green Pet Apatos Rex - Water | Dino Chomp W | Active (10s CD)
`effect: elemental_bonus, effectValue: 25`
**Damage:** 18 (water) | **Cooldown:** 10s
> *Wiki ability text - Dino Chomp W:* Against a Fire opponent, inflicts +25% damage (on top of normal bonus). Inflicts 18 Water damage.

Active ability, 10s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 18 base Water damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 25/100` (i.e. +25%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 18 damage with no bonus.

160. Green Pet Apatosaurus - Water | Dino Slam W | Active (10s CD)
`effect: elemental_bonus, effectValue: 100`
**Damage:** 15 (water) | **Cooldown:** 10s
> *Wiki ability text - Dino Slam W:* Against a Fire opponent, inflicts +100% damage (on top of normal bonus). Inflicts 15 Water damage.

Active ability, 10s cooldown, the "Dino Slam/Charge/Dive/Chomp" line. Deals 15 base Water damage. Before the hit is resolved, the code checks whether this pet's element beats the defending deck's dominant element (via `ELEMENT_BEATS`); if so, damage is multiplied by `1 + 100/100` (i.e. +100%) before being passed into `dealDamage` - which then still applies the normal +25%/-25% elemental modifier on top of that, so favorable match-ups stack both bonuses. If the type matchup isn't favorable, the pet just deals its flat 15 damage with no bonus.

161. Green Pet Triceradon - Water | Defensive Flurry W | Active (10s CD)
`effect: block, effectDuration: 2`
**Damage:** 12 (water) | **Cooldown:** 10s
> *Wiki ability text - Defensive Flurry W:* Strike and then block attacks for 2s. Inflicts 12 Water damage.

Active ability, 10s cooldown. Deals 12 damage to the enemy, then grants the caster's own deck a self-Block status for 2s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

162. Green Pet Tyranodon - Water | Piercing Jaws W | Active (10s CD)
`effect: trauma, effectDuration: 4`
**Damage:** 15 (water) | **Cooldown:** 10s
> *Wiki ability text - Piercing Jaws W:* Causes trauma, preventing the target from healing for 4s. Inflicts 15 Water damage.

Active ability, 10s cooldown. Deals 15 damage, then inflicts Trauma on the target for 4s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

163. Ice Dragon Hand - Water | Frost Breath | Active (10s CD)
`effect: freeze, effectDuration: 2`
**Damage:** 10 (water) | **Cooldown:** 10s
> *Wiki ability text - Frost Breath:* Freeze the target for 2s. Frozen targets can't even swap out. Inflicts 10 Water damage.

Active ability, 10s cooldown. Deals 10 damage, then freezes the target for 2s. Freeze disables using abilities (it's in the stun/freeze/sleep/dance disable list) exactly like Stun, but additionally blocks that deck's Swap button outright - `swapDeck` explicitly refuses to run while a `freeze` status with remaining time is present, so a frozen deck can't retreat to its backup either.

164. Reanimator Remote - Water | Supercharge | Active (10s CD)
`effect: buff_next_damage, effectValue: 50`
**Cooldown:** 10s
> *Wiki ability text - Supercharge:* Charge up your next skill with +50% damage.

Active ability, 10s cooldown, no direct damage. Grants a `next_dmg` buff worth +50% that never expires from a timer (internally given a 99s duration) - it's consumed the moment the caster's deck lands its next hit: `dealDamage` applies the bonus, then immediately strips the buff back off, so it only ever affects exactly one attack no matter how long you wait to use it.

165. Pet Frog - Water | Spit Mud | Active (12s CD)
`effect: slow_recharge, effectValue: 50, effectDuration: 5`
**Damage:** 20 (water) | **Cooldown:** 12s
> *Wiki ability text - Spit Mud:* For 5s, the target's abilities recharge half as fast. Inflicts 20 Water damage.

Active ability, 12s cooldown. Deals 20 damage, then applies a `slow_cd` debuff to BOTH the target's active deck AND its benched partner deck for 5s, halving recharge speed (`speedMult *= 0.5`) for pet cooldowns and for ticking down that deck's Stun/Freeze/Sleep durations - per wiki trivia this partner-deck reach isn't stated outright but is implemented, and the CC-duration slowdown applies as well as ability recharge.

166. Green Pet Triceratops - Water | Defensive Gore W | Active (13s CD)
`effect: block, effectDuration: 4`
**Damage:** 16 (water) | **Cooldown:** 13s
> *Wiki ability text - Defensive Gore W:* Strike and then block attacks for 4s. Inflicts 16 Water damage.

Active ability, 13s cooldown. Deals 16 damage to the enemy, then grants the caster's own deck a self-Block status for 4s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

167. Green Pet Tyranotops - Water | Crushing Beak W | Active (13s CD)
`effect: trauma, effectDuration: 7`
**Damage:** 19 (water) | **Cooldown:** 13s
> *Wiki ability text - Crushing Beak W:* Causes Trauma, preventing the target from healing for 7 seconds. Inflicts 19 Water damage.

Active ability, 13s cooldown. Deals 19 damage, then inflicts Trauma on the target for 7s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

168. Robo Shark - Water | Laserbeam! | Active (13s CD)
`effect: none`
**Damage:** 30 (water) | **Cooldown:** 13s
> *Wiki ability text - Laserbeam!:* Fires a laserbeam that deals Water Damage! Inflicts 30 Water Damage.

Active ability, 13s cooldown. Deals 30 flat damage with no secondary effect attached - a plain, unconditional hit.

169. Green Pet Pterosaurus - Water | Critical Strike W | Active (15s CD)
`effect: anti_dodge_block_conditional, effectValue: 50`
**Damage:** 17 (water) | **Cooldown:** 15s
> *Wiki ability text - Critical Strike W:* If the opponent attempts to dodge or block, hits for 50% more damage. Inflicts 17 Water damage. Can't be dodged or blocked.

Active ability, 15s cooldown, the "Precision/Critical" dino line. Deals 17 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 50% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

170. Mini-Peng - Water | Trick Escape | Active (15s CD)
`effect: dodge_swap, effectDuration: 4`
**Cooldown:** 15s
> *Wiki ability text - Trick Escape:* If struck in the next 4s, avoid it completely and swap out if able to). Teammate gets ability to dodge for 4s. Is not visible to enemy.

Active ability, 15s cooldown ("Self Sacrifice"/"Trick Escape"). If the caster has a living backup deck and is currently allowed to swap, this instantly swaps them onto it (clearing Doom on the deck being left) and leaves a Dodge status worth 4s on the deck being SWAPPED INTO - the incoming teammate gets the dodge window, not the deck retreating (since the retreating deck is now inactive and can't be hit anyway). Per wiki trivia this move can in principle also be reactively triggered by any other self-targeted move the pet uses (Stoneform, Acrobatics, Hop, etc.), but the simulator only implements it for this pet's own direct cast of the move.

171. Muni's Caspiro - Water | Luck Be A Kitty | Active (15s CD)
`effect: none`
**Heal:** 30 | **Cooldown:** 15s
> *Wiki ability text - Luck Be A Kitty:* Good luck heals thy teammate. Heals 30 life.

Active ability, 15s cooldown. No damage, no special effect - restores 30 flat HP to the caster's own deck and nothing else.

172. Mysterious Synthoid - Water | Mysterious Beam | Active (15s CD)
`effect: ethereal, effectValue: 50, effectDuration: 4`
**Damage:** 10 (water) | **Cooldown:** 15s
> *Wiki ability text - Mysterious Beam:* Drain your molecules to shoot a beam. This leaves you ethereal for 4s, taking 50% less damage. Inflicts 10 Water damage.

Active ability, 15s cooldown ("Mist Form"/"Mysterious Beam"). Grants the caster's own deck an Ethereal status for 4s. This is explicitly NOT a dodge chance - `dealDamage` applies it as a flat damage-taken reduction of 50%, multiplying incoming damage by `1 - 50/100` after dodge/block checks but before other damage buffs/debuffs. Recasting replaces the existing Ethereal entry rather than stacking with it.

173. Reindeer Bell - Water | Winter Gift | Active (15s CD)
`effect: none`
**Heal:** 30 | **Cooldown:** 15s
> *Wiki ability text - Winter Gift:* Give a gift to your teammate, healing them. Heals 30 life.

Active ability, 15s cooldown. No damage, no special effect - restores 30 flat HP to the caster's own deck and nothing else.

174. Winter Flu Vaccine - Water | Influenza | Active (15s CD)
`effect: dot, effectValue: 2, effectDuration: 25, dotDamage: 2, dotDuration: 25`
**Cooldown:** 15s | **DoT:** 2/s for 25s
> *Wiki ability text - Influenza:* Infect the target for 2 Water damage per second over 25s.

Active ability, 15s cooldown. Applies a damage-over-time status to the target dealing 2 damage per tick for 25s (subject to the passive_resist_negative resist check and the passive_shorten_debuff duration reduction, same as any other negative effect). Per wiki trivia, DoTs (like all timed effects) don't tick on their very last second of duration, so the realized total damage over the full window ends up one tick short of the naive `2 × 25` calculation.

175. Green Pet Tyrannosaurus - Water | Crushing Jaws W | Active (16s CD)
`effect: trauma, effectDuration: 3`
**Damage:** 29 (water) | **Cooldown:** 16s
> *Wiki ability text - Crushing Jaws W:* Causes trauma, preventing the target for 3s. Inflicts 29 Water damage.

Active ability, 16s cooldown. Deals 29 damage, then inflicts Trauma on the target for 3s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

176. Green Pet Tricerus Rex - Water | Defensive Bite W | Active (18s CD)
`effect: block, effectDuration: 2`
**Damage:** 22 (water) | **Cooldown:** 18s
> *Wiki ability text - Defensive Bite W:* Strike and then block attacks for 2s. Inflicts 22 Water damage.

Active ability, 18s cooldown. Deals 22 damage to the enemy, then grants the caster's own deck a self-Block status for 2s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

177. Green Pet Pteranus Rex - Water | Precision Crush W | Active (19s CD)
`effect: anti_dodge_block_conditional, effectValue: 10`
**Damage:** 25 (water) | **Cooldown:** 19s
> *Wiki ability text - Precision Crush W:* If the opponent attempts to dodge or block, hits for 10% more damage. Inflicts 25 Water damage. Can't be dodged or blocked.

Active ability, 19s cooldown, the "Precision/Critical" dino line. Deals 25 damage. In `dealDamage`, this effect unconditionally sets both `ignoresDodge` and `ignoresBlock` to true for this attack - the hit always lands at full damage regardless of the target's dodge/block state. Note: the wiki describes this as "if the opponent attempts to dodge or block, hit for 10% more damage" - but the current code never reads `effectValue` for this effect anywhere; it simply pierces dodge/block for the pet's flat listed damage with no bonus multiplier applied. The one exception to the piercing itself: if the defender's dodge specifically came from a passive_dodge pet's passive (e.g. Yeonnalligi's Soaring), the pierce is turned back off and the attack is dodged normally - matching the wiki's Soaring-vs-Precision trivia.

178. Bride Of Reanimator Remote - Water | Reanimate | Active (20s CD)
`effect: revive, effectValue: 50, effectDuration: 5`
**Heal:** 50 | **Cooldown:** 20s
> *Wiki ability text - Reanimate:* If beaten within 5s, resurrect with 50% life.

Active ability, 20s cooldown, up to 5s cast window implied by its long cooldown. If the caster's own backup deck is currently at 0 HP, this revives it to 50% of its max HP. If the backup deck is still alive, casting this ability does nothing (no healing, no effect) - it only works as a genuine revive, not a heal.

179. Eldritch Spawn - Water | Cosmic Fear | Active (20s CD)
`effect: mess_up_undodgeable, effectValue: 20, effectDuration: 10`
**Damage:** 20 (water) | **Cooldown:** 20s
> *Wiki ability text - Cosmic Fear:* Curse the target's skills to fail 20% of the time for 10s. Inflicts 20 Water Damage. Can't be dodged.

Active ability, 20s cooldown. Deals 20 damage that can't be dodged (this effect is on the `ignoresDodge` list in `dealDamage`, though Block still works against it), then applies the same kind of `mess_up` debuff as above - 20% ability-fail chance for 10s - the only difference from the plain mess_up pets is that this one's own damage can't be sidestepped by Dodge.

180. Green Pet Tripatosaurus - Water | Defensive Bash W | Active (20s CD)
`effect: block, effectDuration: 7`
**Damage:** 10 (water) | **Cooldown:** 20s
> *Wiki ability text - Defensive Bash W:* Strike and then block attacks for 7s. Inflicts 10 Water damage

Active ability, 20s cooldown. Deals 10 damage to the enemy, then grants the caster's own deck a self-Block status for 7s (`applyStatusEffect` with `type: 'block'`). While Block is up, `dealDamage` intercepts any incoming hit that doesn't specifically ignore block (i.e. isn't `anti_block`/`anti_dodge_block`/a piercing Precision hit) and reduces it to 0 before the damage floor is applied.

181. Green Pet Tyranopatos - Water | Grinding Jaws W | Active (20s CD)
`effect: trauma, effectDuration: 30`
**Damage:** 3 (water) | **Cooldown:** 20s
> *Wiki ability text - Grinding Jaws W:* Causes Trauma, preventing the target from healing for 30s. Inflicts 3 Water damage.

Active ability, 20s cooldown. Deals 3 damage, then inflicts Trauma on the target for 30s. Trauma doesn't deal damage itself - it's a pure anti-heal flag: every heal source in the simulator (base pet heals, Hot ticks, weather healing like Soothing Mist, Life Drain, Absorb's heal-on-nullify, and Revive) explicitly checks for Trauma and skips the heal if it's present. Per wiki trivia, Trauma does NOT block Reanimate/Revive from bringing a deck back, and does NOT prevent Absorb from nullifying damage - it only blocks the healing portion of Absorb's payout.

182. Leashed Silkworm - Blue - Water | Toss Cookies | Active (20s CD)
`effect: cleanse`
**Damage:** 10 (water) | **Cooldown:** 20s
> *Wiki ability text - Toss Cookies:* Purge yourself of harmful effects, inflicting 10 Water damage per effect removed.

Active ability, 20s cooldown ("Toss Cookies"). Strips every negative status (Stun/Freeze/Trauma/DoT/Dance/Sleep) and every modifier debuff currently on the caster's OWN deck, counting how many were removed. For each one removed, the enemy takes 10 damage (so removing 3 effects deals `10 × 3`); if nothing was removed, no damage is dealt at all. Per wiki trivia, this purge triggers even if the ability's damage portion would otherwise be dodged/blocked, and even in cases where the base hit doesn't land, but it does NOT trigger if the cast itself was messed up by a mess_up-type effect.

183. Magnifying Glass - Water | Infect | Active (20s CD)
`effect: dot, effectValue: 6, effectDuration: 7, dotDamage: 6, dotDuration: 7`
**Cooldown:** 20s | **DoT:** 6/s for 7s
> *Wiki ability text - Infect:* Infect the target for 6 Water damage per second lasting 7s.

Active ability, 20s cooldown. Applies a damage-over-time status to the target dealing 6 damage per tick for 7s (subject to the passive_resist_negative resist check and the passive_shorten_debuff duration reduction, same as any other negative effect). Per wiki trivia, DoTs (like all timed effects) don't tick on their very last second of duration, so the realized total damage over the full window ends up one tick short of the naive `6 × 7` calculation.

184. Pet Slime - Water | Regrowth | Active (20s CD)
`effect: hot, effectDuration: 10, dotDamage: 4, dotDuration: 10`
**Heal:** 4 | **Cooldown:** 20s | **DoT:** 4/s for 10s
> *Wiki ability text - Regrowth:* Heal 4 health every second for 10s.

Active ability, 20s cooldown ("Regrowth"). No direct damage to anyone. Grants the caster's OWN deck a heal-over-time status ticking 4 HP per second for 10s. Despite sharing the database's `isDot: true` flag with genuine damage-over-time effects (so it ticks on the same timer machinery), the handler routes it to heal the caster instead of damaging a target - it's checked before the generic DoT branch specifically to avoid being misapplied as damage to the enemy. Blocked by Trauma on the caster, same as any other heal.

185. Rayman's Fist - Water | Super Slap | Active (20s CD)
`effect: stun, effectDuration: 5`
**Damage:** 20 (water) | **Cooldown:** 20s
> *Wiki ability text - Super Slap:* Stuns the target for 5s, making them unable to act. Inflicts 20 Water damage.

Active ability, 20s cooldown. Deals 20 damage, then stuns the target for 5s - disabling its ability to act, but (unlike Freeze) NOT blocking its Swap button, so a stunned deck can still retreat to its backup at any time.

186. Passionate Painter Paintbrush Pet - Water | Hit by Inspiration | Active (25s CD)
`effect: absorb, effectDuration: 4`
**Cooldown:** 25s
> *Wiki ability text - Hit by Inspiration:* For 4 seconds, you take in your surroundings to paint a picture converting any incoming damage to health.

Active ability, 25s cooldown. No direct damage. Raises an Absorb shield on the caster's own deck for 4s. While up, the very next hit(s) this deck would take are fully nullified (0 damage taken) in `dealDamage`, and - unless the deck is also carrying Trauma - the deck heals for exactly the amount of damage that would have been dealt. If Trauma is present, the damage is still nullified but the healing is blocked. Per wiki trivia, this move shares a "move class" with the other Absorb-type pet: recasting one before the other expires EXTENDS the remaining duration rather than resetting it, and one of the two variants reuses the same 2-second cast animation even though its actual shield window is longer, so there's no visual cue for the remaining time beyond the animation.

187. Black Cat Leash - Water | Bad Luck | Active (30s CD)
`effect: debuff_damage_both, effectValue: 25, effectDuration: 8`
**Cooldown:** 30s
> *Wiki ability text - Bad Luck:* Cross the target's path, making them do -25% damage, and suffer +25% damage, for 8s.

Active ability, 30s cooldown ("Bad Luck"), no direct damage. Applies a `bad_luck` debuff to the target for 8s that hits them twice over: -25% damage dealt AND +25% damage taken at the same time, both driven off the same `effectValue`.

188. Diamond Dragon - Water | Diamond Block | Active (30s CD)
`effect: anti_swap, effectDuration: 10`
**Cooldown:** 30s
> *Wiki ability text - Diamond Block:* Prevent the target from swapping out for 10s.

Active ability, 30s cooldown, no direct damage. Applies a `no_swap` debuff to the target deck for 10s (`applyModifiers`). While active, `recomputeModifiers` sets `canSwap = false` on that deck, so its Swap button is locked (`🔒 LOCKED` in the UI) and `swapDeck()` refuses to run for the duration - the target is stuck on its current deck even if it wants to retreat.

189. Ghost Wolf Monocle - Water | Mist Form | Active (30s CD)
`effect: ethereal, effectValue: 50, effectDuration: 10`
**Cooldown:** 30s
> *Wiki ability text - Mist Form:* Become ethereal for 10s, reducing all damage taken by 50%.

Active ability, 30s cooldown ("Mist Form"/"Mysterious Beam"). Grants the caster's own deck an Ethereal status for 10s. This is explicitly NOT a dodge chance - `dealDamage` applies it as a flat damage-taken reduction of 50%, multiplying incoming damage by `1 - 50/100` after dodge/block checks but before other damage buffs/debuffs. Recasting replaces the existing Ethereal entry rather than stacking with it.

190. Ice Calf Leash - Water | Ice Laser | Active (30s CD)
`effect: freeze, effectDuration: 5`
**Cooldown:** 30s
> *Wiki ability text - Ice Laser:* Freeze the target for 5s. Frozen targets can't even swap out.

Active ability, 30s cooldown. Deals 0 damage, then freezes the target for 5s. Freeze disables using abilities (it's in the stun/freeze/sleep/dance disable list) exactly like Stun, but additionally blocks that deck's Swap button outright - `swapDeck` explicitly refuses to run while a `freeze` status with remaining time is present, so a frozen deck can't retreat to its backup either.

191. Magical Carrot - Water | That's Cold | Active (30s CD)
`effect: throw_teammate`
**Cooldown:** 30s
> *Wiki ability text - 30s:* 

Active ability, 30s cooldown ("That's Cold"). If the caster's own backup deck is alive, it is instantly reduced to 0 HP, and half of that partner's HP (rounded down, as it stood right before the KO) is dealt to BOTH of the enemy's decks (their active deck via the normal damage path, and their backup deck too if it's alive). If the cast itself is messed up by a mess_up-type debuff, the whole handler never runs and the partner is spared - UNLESS the enemy happens to be dodging or blocking at that moment, in which case the partner is still instantly KO'd anyway despite the failed cast (a specific wiki-documented exception to the mess-up-spares-the-partner rule).

192. Royal Eldritch Spawn - Water | Cosmic Fear | Active (30s CD)
`effect: mess_up_undodgeable, effectValue: 20, effectDuration: 15`
**Damage:** 30 (water) | **Cooldown:** 30s
> *Wiki ability text - Cosmic Fear:* Curse the target's skills to fail 20% of the time for 15s. Inflicts 30 Water Damage. Can't be dodged.

Active ability, 30s cooldown. Deals 30 damage that can't be dodged (this effect is on the `ignoresDodge` list in `dealDamage`, though Block still works against it), then applies the same kind of `mess_up` debuff as above - 20% ability-fail chance for 15s - the only difference from the plain mess_up pets is that this one's own damage can't be sidestepped by Dodge.

193. Leashed Silkworm - Aqua - Water | Soothing Mist | Active (60s CD)
`effect: weather_hot, effectDuration: 30, dotDamage: 6, dotDuration: 30`
**Cooldown:** 60s | **DoT:** 6/s for 30s
> *Wiki ability text - Soothing Mist:* Weather Effect. Heal everyone for 6 life every 5s, lasting 30s. Replaces any other active weather effect.

Active ability, 60s cooldown ("Soothing Mist"). Sets the global battlefield weather for 30s: every 5 ticks, BOTH sides' current active decks are healed for 6 HP (blocked by Trauma on whichever deck has it). Per wiki trivia, like the damage-dealing weathers, it heals for less than the naive total on its last tick before expiring (30 instead of a nominal 36 over its full window, per the wiki's own example).

---


---

# Part 3: Developer / Function Reference

---

### 1. File Layout

`simulator.html` is a single self-contained file: `<style>` block, a `#team-builder` screen and
an empty `#battle-screen` div in the HTML body, then one big `<script>` block. There's no
build step, no modules, no framework - everything is plain global functions and globals.

```
<style>...</style>
<body>
  #team-builder   (deck-building UI, visible first)
  #battle-screen  (empty until "Start Battle" is clicked, then filled by JS)
  <script>
    // 1. Team-builder state & functions
    // 2. Battle engine (state, damage pipeline, effect dispatch, game loop)
    // 3. UI rendering functions
    // 4. Start-battle click handler (wires it all together)
  </script>
</body>
```

Pet data itself is NOT hard-coded in this file - it's fetched at load time from
`Data/pet_battle_database.json` into the global `allPets` array. **This is the file you edit to
add a new pet**. `simulator.html` only contains the *engine* that interprets whatever
`effect` string a pet has.

---

### 2. Data Model

#### 2.1 A pet's raw database record (one entry in `pet_battle_database.json`)

```js
{
  name: "Yeonnalligi",
  element: "Air",                 // Fire | Earth | Air | Water
  ability: "Soaring",
  is_passive: true,               // true = never manually cast, always-on
  cooldown: 0,                    // seconds; ignored if is_passive
  damage: 0, damageType: "",
  heal: 0,
  effect: "passive_dodge",        // this string picks which code branch runs it
  effectValue: 25,
  effectDuration: 0,
  isDot: false, dotDamage: 0, dotDuration: 0,
  img: "....png", hitSound: "....mp3", powerSound: "....mp3"
}
```

Every pet card in the team builder is a shallow copy (`{ ...pet }`) of one of these records.
Nothing about a specific pet is special-cased by name in most of the engine - the `effect`
field is what routes it through the code - **except** a small, deliberate list of by-name
checks for pets whose behavior can't be captured by `effect` alone:

| Special-cased by **name/ability**, not `effect` | Where | Why |
|---|---|---|
| `sourcePet.data.ability === 'Soaring'` | `dealDamage` | Precision/Critical dino line explicitly does NOT pierce this specific pet's dodge |
| `sourcePet.data.ability === 'Disco Fever'` | `applyModifiers` (`force_dance`) | Only this dancer also grants Dodge |
| `pet.data.name === 'Pinata Pal'` | `handlePetEffect` (`trap_swap`) | Only this trap hits both enemy decks |
| `['Strawberry Slime', 'Leashed Silkworm - Purple', 'Pet Present Goblin']` | `applyModifiers` (`stackingPets`) | Only these sources stack their buff/debuff instead of refreshing |
| `attackerPet.data.ability === 'Toasties'` | `dealDamage` (`isToasties`) | Flags a hit to skip nearly every modifier |

If you're adding a new pet that needs one of these special interactions, this table is where
to go add it - don't invent a new `effect` key just to special-case one pet's name.

#### 2.2 The `battle` object (all runtime state lives here)

```js
battle = {
  running: true, tick: 0,
  delayedHits: [], healBacks: [], swapTraps: [],   // queues, ticked down once per gameTick
  weather: { type, remaining, damage, tickRate, tickCounter, heal, fireDmgMod, otherDmgMod, sourceName, sourceImg },
  your:   { activeDeck: 1, activePet: 0, livingDeadUsed: false, decks: { 1: <deck>, 2: <deck> } },
  enemy:  { ...same shape... }
}
```

A **player** object (`battle.your` / `battle.enemy`) tracks which deck/pet is currently
selected and the "used Living Dead already" flag. A **deck** object:

```js
deck = {
  hp: 150, swapCd: 0, duckBackCd: 0,
  pets: [ {data, cd, charge?, growStacks?, buildPhase?}, ... up to 3 ],
  statusEffects: [ {type, duration, remaining, sourceName, sourceImg, ...extra fields} ],
  modifiers: {
    damageDealt, damageTaken, nextDamage, doubleDmgChance, rechargeSpeed,
    messUpChance, canSwap, canAct, buffs: [], debuffs: []
  }
}
```

`pet.data` is the static database record; everything else on `pet` (`cd`, `charge`,
`growStacks`, `buildPhase`) is per-battle mutable state bolted onto that pet instance.

`statusEffects` holds **timed, type-tagged entries** (stun/freeze/trauma/dodge/block/dot/hot/
ethereal/absorb/doom/dance/sleep) - anything with a `remaining` countdown that isn't a
percentage modifier. `modifiers.buffs`/`.debuffs` hold **percentage/flat modifiers** (damage %,
recharge speed, mess-up chance, etc.) that get summed into the flat fields above by
`recomputeModifiers`. This split exists because status effects gate *actions* (can this deck
act/swap right now?) while modifiers scale *numbers* (how much damage, how fast cooldowns
tick) - see the table below for exactly which is which.

---

### 3. Team Builder Functions

These only run before battle starts; none of them touch `battle` (which doesn't exist yet).

| Function | Role |
|---|---|
| `filterPets(query, containerId)` | Live-filters the pet-card grid as you type in the search box; hides whole element sections if nothing in them matches. |
| `buildTeamBuilder()` | Entry point once `Data/pet_battle_database.json` loads. Builds both sides' pet lists and both decks' empty slot rows. |
| `buildPetList(cid, owner)` | Renders one side's scrollable pet grid, grouped into Air/Earth/Fire/Water sections, one `.pet-card` per pet with a click handler wired to `addPet`. |
| `buildSlots(cid, deck, owner, dnum)` | Renders the 3 empty slot placeholders for one deck; each slot's click handler is `removePet`. |
| `countInTeam(team, petName)` | Counts how many copies of a named pet are already placed across both of a side's decks (used to enforce the 2-copy limit). |
| `updateCardState(owner, petName)` | Updates a pet card's visual state (`in-team`/`partial-team` class + "1/2"/"2/2" badge) after adding/removing a copy. |
| `addPet(pet, owner, card)` | Click handler for a pet card. Refuses outright if already at 2 copies for this side. Otherwise scans deck 1 then deck 2, **skipping any deck that already contains this pet name**, and drops a copy into the first empty slot of the first deck that both lacks it and has room - this is what enforces "at most one copy per deck, two total across both decks," not just the top-level 2-copy check. |
| `removePet(owner, dnum, slot)` | Click handler for an occupied slot - clears it back to empty. |
| `refreshAll()` / `refreshSlots(cid, deck)` | Re-renders all 4 deck-slot rows from the `yourTeam`/`enemyTeam` state after any add/remove. |
| `updateBtn()` | Shows/hides the "⚔️ START BATTLE" button - only visible once both sides have all 6 slots filled. |
| `playSound(soundPath)` | Cached `Audio` playback helper (also used during battle for hit/power sounds); silently swallows autoplay-blocked errors. |

**To add:** a new pet doesn't require touching any of these - they all read from `allPets`,
which comes straight from the JSON file.

---

### 4. Battle Setup

| Function | Role |
|---|---|
| `initBattle()` | Builds the fresh `battle` object from `yourTeam`/`enemyTeam`. For each deck: computes each pet's starting cooldown (`floor(cooldown/2)`, capped at 12, floored at 1, minus 2 more if the deck has Windspeed), and calls `resetModifiers`. |
| `resetModifiers(deck)` | Resets a deck's `modifiers` object to its all-zero/neutral defaults. Called once per deck at battle start - **not** a general-purpose "clear all buffs" function; mid-battle clearing (e.g. `cleanse`) manipulates `.buffs`/`.debuffs` arrays directly and calls `recomputeModifiers` instead. |
| `getDeck(player)` | `player.decks[player.activeDeck]` - the currently active deck. |
| `getPet(player)` | The currently selected pet within that active deck. |
| `inactiveDeck(player)` | The OTHER (benched) deck. |
| `getMaxHp(player, deckNum?)` | 195 if that deck contains a `passive_hp_boost` pet (Mammoth), else 150. Defaults to the active deck if `deckNum` omitted. |
| `getDeckElement(deck, activeIdx)` | The deck's "dominant element" for elemental-advantage purposes: majority vote across all 3 pet slots, tiebroken by whichever pet is at `activeIdx`. |

---

### 5. The Damage & Effect Pipeline

This is the part you'll touch most. Understanding the *order of operations* here is essential
before adding anything.

#### 5.1 `dealDamage(player, amount, attackerPet)`

The single funnel almost all damage goes through (exceptions: Toasties' flat hit, weather
ticks, and recoil-style self-damage all subtract HP directly, bypassing this). Applies, **in this exact order**:
Purple Haze weather mod → elemental ±25% → dodge check → block check → Ethereal % → attacker's
double-damage roll / next-hit bonus → attacker's ongoing damage-dealt modifier → defender's
damage-taken modifier → `passive_dmg_reduce` flat -25% → hit sound → **floor to minimum 1** →
Absorb nullify-and-heal check → subtract HP → Static Charge stacking → Counter/Thorns
reflection. Returns the actual damage number dealt (0 if dodged/blocked/absorbed) - **always
log this return value**, not the pet's raw `damage` field, since it reflects everything that
actually happened to the hit.

#### 5.2 The generic hit + effect split

Using an ability is **two separate steps**, not one function:
1. The click handler / `aiAct` deals the pet's base `damage`/`heal` generically (this is where
   `SPECIAL_DAMAGE_EFFECTS` gets checked - see below).
2. `handlePetEffect(pet, sourceDeck, targetDeck, owner)` is called afterward to apply whatever
   *secondary* effect the pet has (stun, buff, DoT, forced swap, etc.).

**`SPECIAL_DAMAGE_EFFECTS`** is the list of effect keys where step 1 must be skipped, because
`handlePetEffect` computes and deals damage itself in a non-generic way (a random roll, a
drain-then-heal, hitting a second deck, no immediate hit at all, etc.). If you add a new effect
that needs custom damage math instead of a flat `pet.data.damage` hit, **add its key to
`SPECIAL_DAMAGE_EFFECTS`** or the generic step will *also* fire a duplicate plain hit before
your handler runs.

#### 5.3 `handlePetEffect` - the dispatch table

This function is a long if/return chain (NOT a switch) checked top-to-bottom; the first
matching branch handles the effect and returns. Order matters for a few overlapping cases
(commented in the source) - most importantly `self_dot` and `hot` are checked **before** the
generic `isDot` branch, because both are flagged `isDot: true` in the database but need to
heal/burn the *caster* instead of the target.

Roughly, in the order they're checked:

1. **Weather effects** (`WEATHER_EFFECTS`) → `applyWeather`
2. `force_dance` (may also carry a DoT component)
3. `self_dot` - burns the caster, not the target
4. `hot` - heals the caster over time
5. Generic `isDot` - burns the target
6. `ENEMY_STATUS_EFFECTS` (stun/freeze/trauma) → `applyStatusEffect` on target
7. `SELF_STATUS_EFFECTS` (dodge/block) → `applyStatusEffect` on caster
8. Then one `if` block per remaining unique effect key: `revive`, `cleanse`, `slow_recharge`,
   `extend_cooldown`, `debuff_damage_dealt`, `SELF_BUFFS` (buff_damage_dealt /
   buff_chance_double_damage / buff_next_damage / buff_damage_and_speed / fast_recharge),
   `random_damage`, `bonus_on_debuff`, `desperate`, `random_element`, `life_drain`,
   `self_damage`, `self_stun`, `delayed_damage`, `counter`/`thorns`, `doom`, `banish`,
   `absorb`, `ethereal`, `hit_both`, `mind_swap`, `force_partner_forward`, `consume_buff`,
   `force_swap`, `chain_on_kill`, `heal_back`, `elemental_bonus`, `summon`, `throw_teammate`,
   `dodge_swap`, `stack_burst`, `stacking_build`, `force_swap_transfer`,
   `random_skill_wrong_target`, `trap_swap`, `mess_up_undodgeable`.
9. **Fallback:** anything not matched above falls through to the final line -
   `applyModifiers(targetDeck, pet.data, false, pet, ...)` - a generic enemy-targeted debuff.
   This is why some effects (like `anti_swap`) have no dedicated `if` block in
   `handlePetEffect` at all: they're simple enough that the generic fallback handles them, with
   their actual behavior living inside `applyModifiers`'s switch statement instead.

#### 5.4 `applyModifiers(deck, effectData, isSelf, sourcePet, owner)` and `applyStatusEffect(deck, effectData, sourcePet, owner)`

These two are the **actual mechanical implementations** for the simpler, more uniform effect
types (both status-effect-like things and modifier-like things). If your new effect is a
straightforward "+X% for Ys" buff/debuff or a plain stun/freeze/dodge/block, you add its case
to the `switch` here, not to `handlePetEffect` - `handlePetEffect` should only special-case
things that need genuinely custom logic (multi-deck hits, RNG, conditional branching, etc.).

Both functions log automatically - you don't need a separate `addLog` call at the
handlePetEffect level for anything routed through here.

**`owner` parameter convention (important, and easy to get backwards):** in both functions,
`owner` is always the owner-key of the `deck` argument being modified - NOT necessarily the
caster. For a debuff (`isSelf`/`isDebuff` false), the caster is the *opposite* key; for a
self-buff or self-status (dodge/block), the caster IS `owner`. Both functions derive a local
`casterOwner`/`casterType` at the top specifically to get logging attribution right - reuse
that pattern rather than assuming `owner` means "the caster."

#### 5.5 `recomputeModifiers(deck)`

Called after every buff/debuff array mutation. Resets the deck's flat numeric fields to zero,
then sums every entry in `modifiers.buffs`/`.debuffs` into `damageDealt`, `damageTaken`,
`doubleDmgChance`, `nextDamage`, `messUpChance`, `rechargeSpeed` (multiplicative - 2x per
fast source, 0.5x per slow source, stacks multiplicatively if both are present), `canSwap`,
`canAct`. **If you add a new buff/debuff `type` string, you must add a `case` for it here** or
it will sit in the array doing nothing.

#### 5.6 `tryResist` / `upsertModifierEntry`

- `tryResist(deck, owner)` - the single implementation of Duck's Back (`passive_resist_negative`):
  if the deck has that passive and it isn't on cooldown, blocks the incoming negative effect
  entirely and puts the passive on cooldown. Called at the top of `applyStatusEffect` (for
  stun/freeze/trauma) and in a couple of direct DoT/Toasties branches - **not** wired into
  weather or `cleanse` (see the discrepancy note in [Part 1](#part-1-core-mechanics) if you're
  trying to reconcile this with the wiki).
- `upsertModifierEntry(list, type, entry, isStackingPool)` - shared "add or refresh" logic for
  buff/debuff arrays. Same-source reapplication refreshes duration (or, for stacking pets,
  adds to the value and extends to the longer duration); different sources coexist as separate
  array entries. This is what makes multiple different debuffers not overwrite each other.

---

### 6. Ability Use, Swapping, Weather, Passives, AI

| Function | Role |
|---|---|
| `useAbility(player)` | Gatekeeper for casting: checks cooldown/disabled/passive, rolls the mess-up chance (logging "Ability failed!" and still applying the two mess-up carve-outs - Swoop's dodge, weather still firing - on a fail), then resets cooldown (factoring in Windspeed and any armed `extend_cd` debuff) and locks the deck's other ready pets to 2s. Returns `true`/`false`; callers only proceed to deal damage/call `handlePetEffect` on `true`. |
| `swapDeck(player)` | Gatekeeper for swapping: checks swap-cooldown/Freeze/`canSwap`, springs any armed swap traps targeting this player first, clears Doom on the deck being left, flips `activeDeck`, sets the new deck's 3s swap cooldown, and locks its ready pets to 2s (skipped if that deck has Windspeed). |
| `selectPet(player, index)` | Just moves `activePet` to a different (non-empty) slot in the same active deck - no cooldown cost, no effect resolution. |
| `applyWeather(effectData, sourcePet, owner)` | Sets the single global `battle.weather` slot to one of firestorm/toxic_cloud/purple_haze/soothing_mist/count_sheep based on the cast pet's fields, and logs the cast. Overwrites any weather already active. |
| `applyPassives(player)` | Runs every tick for always-on, non-selected-pet-dependent passives: heal-while-benched (`passive_heal_inactive`, every 5 ticks) and auto-attack-while-benched (`auto_attack_inactive`, every `pet.data.cooldown` ticks). `passive_dodge`/`passive_dmg_reduce`/etc. are NOT handled here - those are checked inline inside `dealDamage` against whichever pet is currently active, since they only matter at the moment of a hit. |
| `getPassiveDesc(pet, deck)` | Tooltip text generator for passive icons - one `case` per passive `effect` key. If you add a new passive effect, add its description here too, or its tooltip falls back to just the ability name. |
| `aiAct()` | Runs every other tick (`battle.tick % 2 === 0`) for `battle.enemy`. Priority: cast the active pet if ready → else switch to another ready pet in the same deck → else swap decks if the backup has something ready → else do nothing. |
| `tryLivingDeadRevive(player, deadDeckNum)` | Living Dead Remote's revive check (see [Part 1](#part-1-core-mechanics), "Deck Death & Revival" for the full behavior). Requires the Remote to be in the OTHER (still-alive) deck from the one that just died; once-per-battle via `player.livingDeadUsed`. |
| `gameTick()` | The 1-second heartbeat - see the ordered walkthrough below. |

#### 6.7 Deferred-effect queues

Three arrays on `battle` hold effects that don't resolve immediately when cast, each ticked
down once per `gameTick`:
- `delayedHits` - Zombie Hound's spore burst; fires flat damage once `ticksLeft` hits 0.
- `healBacks` - Phantom Pain's return-heal; spreads a total amount roughly evenly across its
  remaining ticks each tick.
- `swapTraps` - Lockjaw!/Surprise!'s hidden trap; springs (extra damage) the moment the
  *targeted* player calls `swapDeck`, not on its own timer (the timer only controls how long it
  stays armed if never triggered).

If you add a new "happens later" effect, follow one of these three patterns rather than
inventing a fourth queue shape unless genuinely necessary.

#### 6.8 `gameTick()` walkthrough (order matters)

1. Bump `battle.tick`.
2. Weather tick (damage/heal/sleep-onset, using its own internal `tickRate`, separate from the
   1s master clock).
3. For every deck on both sides: tick down swap-cooldown, Duck's Back cooldown, every pet's
   ability cooldown (scaled by `rechargeSpeed`); tick down every status effect's `remaining`
   (stun/freeze/sleep scaled by `rechargeSpeed`, everything else by flat 1); apply DoT/Hot
   ticks (note: these fire on the tick where `remaining` hits exactly 0, one tick before
   removal - see the last-tick trivia in [Part 1](#part-1-core-mechanics), "Status Effects"); resolve any Doom that
   just hit 0; prune expired status effects and buffs/debuffs; `recomputeModifiers`.
4. Advance `delayedHits`, `healBacks`, `swapTraps` queues.
5. `applyPassives` for both sides.
6. `aiAct()` on even ticks.
7. Per-side death check: if the active deck is at 0 HP, try `tryLivingDeadRevive`; if that
   fails, auto-switch to the surviving backup deck if one exists.
8. Compute `yDead`/`eDead` (both decks at 0) and end the battle if either is true.
9. `updateBattleUI()`.
10. Re-schedule itself via `setTimeout(gameTick, 1000)` if still running.

---

### 7. Logging

| Function | Role |
|---|---|
| `addLog(msg, type, owner)` | Writes a short-lived floating popup under the given side's active-deck area (auto-removed after 2.2s) AND forwards to `addDetailedLog`. `type` is `'your-dmg'` (green), `'enemy-dmg'` (red), or anything else (defaults to blue/"heal" styling). `owner` picks which side's popup ('your'/'enemy') - pass `null`/anything else and it defaults to the enemy box, so always pass an explicit key. |
| `addDetailedLog(msg, type, owner)` | Appends a permanent line (capped at the last 100) to the scrollable side panel, prefixed with the current tick and a 🟢/🔴 marker (blank if `owner` isn't `'your'`/`'enemy'` - used for owner-less global events like weather ticks). |
| `toggleDetailLog()` | Click handler for the panel header - collapses/expands the detail log body. |

**Convention:** always log using the `casterOwner`/`casterType` derived at the top of whichever
function is doing the logging - don't reuse the raw `owner` parameter directly for
attribution, since its meaning flips depending on isSelf/isDebuff. Always include the actual
number (damage/heal/duration/%) in the message - see the logging conventions used throughout Part 1;
past bugs came from effects applying completely silently because this was skipped.

---

### 8. UI Rendering

These all read from `battle` and rebuild DOM every call - none of them mutate battle state.

| Function | Role |
|---|---|
| `updatePassiveIndicators()` | Renders the passive-icon row for each side. Most passives show only while their pet is in the *active* deck; `passive_heal_inactive` and `passive_revive` are the exception - they only show while **benched**, mirroring their actual trigger condition (and `passive_revive` additionally shows a "USED" badge once `livingDeadUsed` is true). |
| `updateDebuffIndicators()` / `updateBuffIndicators()` | Render the debuff/buff icon rows from `statusEffects` + `modifiers.debuffs`/`.buffs`, plus the two special per-pet counters (`stack_burst`'s `charge`, `stacking_build`'s `growStacks`). If you add a new status/modifier `type`, add its display name to the `names`/`typeNames` map here or it won't render an icon (it'll still function correctly, just invisibly). |
| `updateWeatherUI()` | Renders the top-center weather badge (icon, name, remaining seconds) from `battle.weather`. |
| `updateBattleUI()` | The big one - rebuilds each side's deck label, status/buff/debuff tag row, pet row (with the click handler that actually calls `useAbility`/`handlePetEffect` for the human player - see below), HP bar, backup-deck sidebar, and swap button; then calls all four of the above. This is called after every `gameTick()` and after every manual player action. |

**The human player's click handler** lives inline inside `updateBattleUI`'s pet-row rendering
(around the `pr.appendChild(el)` loop) - it's the mirror of `aiAct()`'s cast logic but wired to
a DOM click instead of the AI's automatic decision, and includes the same
"generic hit, unless `SPECIAL_DAMAGE_EFFECTS`" + `handlePetEffect` two-step described earlier.

---

### 9. Recipe: Adding a New Pet

1. Add a new object to `Data/pet_battle_database.json` following the schema described earlier
   in this document ("A pet's raw database record"). Reuse an
   existing `effect` key if the pet's ability matches an existing mechanic exactly (just with
   different numbers) - you don't need any code changes for that case at all.
2. Drop its sprite into `Sprites/` matching the `img` field, and sounds into wherever
   `hitSound`/`powerSound` point (missing images fall back to `Sprites/0.png` automatically via
   `onerror`; missing sounds fail silently via `playSound`'s `.catch(() => {})`).
3. If it needs a genuinely new mechanic, see the next recipe instead.
4. That's it - the team builder, pet list, and grid all read `allPets` dynamically; nothing
   else needs to know the pet exists.

### 10. Recipe: Adding a New Ability Effect

1. **Pick the smallest existing pattern that's close to what you need** and read that section
   of `handlePetEffect`, `applyModifiers`, or `applyStatusEffect` first - most new abilities are
   a variation on an existing one (a new debuff percentage, a different target, an extra
   secondary hit) rather than something wholly novel.
2. **Decide where it belongs:**
   - Simple timed status (disables acting, or a guaranteed dodge/block) → add a case to
     `applyStatusEffect`'s switch, and add its type to `ENEMY_STATUS_EFFECTS` or
     `SELF_STATUS_EFFECTS` so `handlePetEffect` routes to it.
   - Simple percentage/flat buff or debuff → add a case to `applyModifiers`'s switch (and to
     `SELF_BUFFS` if it's a self-buff so `handlePetEffect` routes to it), AND add a matching
     case to `recomputeModifiers` so the stored value actually does something.
   - Anything with custom damage math, multi-deck targeting, RNG, or conditional logic → add a
     new `if (pet.data.effect === 'your_new_effect') { ...; return; }` block directly in
     `handlePetEffect`, following the existing blocks as templates.
3. **If it deals damage in a non-generic way** (random amount, hits a second target, no
   immediate hit, drains-then-heals, etc.), add its effect key to `SPECIAL_DAMAGE_EFFECTS` -
   otherwise the generic click/AI handler will ALSO fire a duplicate flat-damage hit using
   `pet.data.damage` before your custom code runs.
4. **If it bypasses normal damage rules entirely** (like Toasties ignoring elemental/dodge/
   block/buffs), apply damage by subtracting `deck.hp` directly instead of calling
   `dealDamage` - don't try to thread new bypass flags through `dealDamage` unless multiple
   future effects will need the same bypass.
5. **Log it.** Every branch should end with an `addLog(...)` call stating the actual
   number(s) involved (damage dealt, % applied, duration) - see the logging conventions covered earlier. This is the
   single most common thing that gets missed when adding a new effect.
6. **Add a UI label if it's a status effect or modifier**, not just a one-off `if` block:
   - `updateDebuffIndicators`/`updateBuffIndicators`'s `names`/`typeNames` maps (icon tooltip)
   - `getPassiveDesc` if it's a passive
7. **Test the interactions, not just the happy path:** does it need to respect Trauma (if it
   heals)? Duck's Back / `tryResist` (if it's a debuff)? `exoReduction` (if it's timed)? Does it
   need to check `canAct`/`canSwap` before acting? Does an existing pet's passive (Windspeed,
   Stubborn's `passive_dmg_reduce`, Soaring's dodge) interact with it in a way the wiki calls
   out? Cross-check against [Part 2](#part-2-pet-ability-documentation) and [Part 1](#part-1-core-mechanics) for any
   documented trivia/edge case that touches your new effect's category.

### 11. Quick Reference: Constants

| Constant | Contents | Used for |
|---|---|---|
| `SELF_BUFFS` | `buff_damage_dealt, buff_chance_double_damage, buff_next_damage, buff_damage_and_speed, fast_recharge` | Routes these effect keys to `applyModifiers(sourceDeck, ..., isSelf=true, ...)` in `handlePetEffect` |
| `ENEMY_STATUS_EFFECTS` | `stun, freeze, trauma` | Routes to `applyStatusEffect(targetDeck, ...)` |
| `SELF_STATUS_EFFECTS` | `dodge, block` | Routes to `applyStatusEffect(sourceDeck, ...)` |
| `WEATHER_EFFECTS` | `weather, weather_hot, sleep` | Routes to `applyWeather(...)` |
| `SPECIAL_DAMAGE_EFFECTS` | `cleanse, random_damage, life_drain, counter, thorns, desperate, random_element, elemental_bonus, summon, stack_burst, stacking_build` | Suppresses the generic flat-damage hit in the click/AI handlers so `handlePetEffect` can deal damage its own way without a duplicate hit |
| `ELEMENT_BEATS` | `{Fire: Earth, Earth: Air, Air: Water, Water: Fire}` | Elemental advantage lookup in `dealDamage`, `getDeckElement`-based checks, and `elemental_bonus` |

---

### 12. Known Rough Edges (things to be aware of before "fixing" them)

- `desperate` (Desperate Bite) and `random_skill_wrong_target` (Possession) are explicitly
  commented in the source as best-guess implementations - the wiki doesn't give exact numbers/
  targeting rules for them. If you get authoritative numbers, update the code AND
  the affected pets' entries in [Part 2](#part-2-pet-ability-documentation).
- Weather is **not** currently resistible (`tryResist`) or purgeable (`cleanse`) even though the
  wiki implies it should be - `battle.weather` is a separate global object neither function
  looks at. Fix this in both `tryResist`'s call sites (would need a weather-aware branch) and
  `cleanse`'s removal logic if you want to match the wiki.
- Toasties (`summon`) bypasses `dealDamage` entirely, so it also skips Purple Haze's modifier -
  the wiki claims Purple Haze is the one exception to "Toasties ignores everything." If you
  want to honor that, you'd need to manually apply Purple Haze's `fireDmgMod`/`otherDmgMod`
  inside the `summon` branch in `handlePetEffect` rather than relying on `dealDamage`.