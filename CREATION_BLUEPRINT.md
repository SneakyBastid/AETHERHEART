# AETHERHEART Creation Blueprint
## Comprehensive System Architecture & Implementation Guide

---

## SECTION 1: PROJECT FOUNDATION

### 1.1 System Identity
- **Name:** AETHERHEART
- **Purpose:** Complete RP system for AI Dungeon (standalone, no dependencies)
- **Design Philosophy:** Clean, user-friendly, Realmheart-style layout and language
- **License:** MIT (original code, fully original naming)
- **Base Reference:** Built in spirit of Realmheart, not as extension

### 1.2 Core Architecture Principles
- All new systems are completely self-contained
- No shared state with other scripts
- Card layout matches Realmheart aesthetic (clean, toggle-based, next-turn apply)
- State stored in: `state.aetherheart` (isolated namespace)
- Context injection is additive and ordered
- Undo/erase/retry properly rolls back state (no permanent corruption)
- All toggles: OFF by default or ON (user choice), apply next turn

### 1.3 Language & User Experience
- User-friendly wording only (no jargon)
- All abbreviations explained (curt explanations in tooltips)
- Matches Realmheart command structure: `/help` teaches everything else
- No "escalating adverbs" language—just plain terms
- `[SYSTEM]` and `[SHOP]` use bracket notation consistently
- Configuration cards match Realmheart format exactly

---

## SECTION 2: CONFIGURATION CARD STRUCTURE

### 2.1 Master Configure Card
**Card Name:** `CONFIGURE AETHERHEART`

**Top Section (Non-negotiable Format):**
```
== CONFIGURE AETHERHEART ==
[Changes apply on the next turn.]
```

**Sections in order:**
1. `-- SYSTEM --` (core toggles: Script, Inventory, Currency, Time, Weather, Holidays, Events, Bills)
2. `-- RP SYSTEM --` (Vitals, Survival, Leveling, Skills, Perks, Powers, Inner Thoughts, Inner Opinions, Quests, Tickers, [SYSTEM], Card Creator)
3. `-- STORY CARDS --` (list of all secondary cards: Master, Inventory, Configure Aetherheart, Configure Time, Vitals, Character Sheet, etc.)

**Rule:** Each toggle is a single line. Format: `Feature Name:                Value`

### 2.2 Secondary Card Categories
All secondary cards follow this structure:
- **Header:** `== CARD NAME ==`
- **Status line:** Current values/state
- **Configuration section:** Editable toggles and parameters
- **Action line:** `[Edit X directly. Changes apply next turn.]` or similar

**Secondary Cards Required:**
1. Vitals & Survival
2. Character Sheet
3. Drugs & Consumables
4. Character Minds (Inner Memory system)
5. Relationship Web (Drama/Opinions system)
6. Mind Journal (Tracked summary log)
7. Quests
8. Universal Tickers
9. System Shop
10. Card Creator Config (Auto-Card generation)
11. Configure Time (Time progression controls)
12. Master (Inventory/Currency/Bills tracking)

---

## SECTION 3: FEATURE BREAKDOWN BY SYSTEM

### 3.1 SYSTEM (Core Gameplay Loop)
**Toggles in Configure Card:**
- Script: ON/OFF
- Inventory: ON/OFF
- Currency: ON/OFF (called "SP" or game currency)
- Time: ON/OFF
- Weather: ON/OFF
- Holidays: ON/OFF
- Events: ON/OFF
- Bills: ON/OFF

**Behavior:**
- Each toggle controls whether that subsystem is active
- Changes apply next turn
- All must work independently if one is toggled off

### 3.2 RP SYSTEM (Vitals & Survival)
**Toggles:**
- Vitals: ON/OFF (HP / MP / Stamina)
- Survival: ON/OFF (Hunger / Thirst / Tiredness)
- Link Stamina → Tiredness: ON/OFF
- Link MP → Tiredness: ON/OFF
- Eating restores vitals: ON/OFF

**Live Tracking (Vitals & Survival Card):**
- Current: HP, MP, Stamina (with max values)
- Current: Hunger %, Thirst %, Tiredness % (with descriptors: Satisfied/Mild/Severe)
- Settings: Max values, drain rates (normal/fast/slow), linkage toggles
- Edit capability: User can adjust current or max values directly

**Hard Links:**
- Stamina depletion increases Tiredness
- MP depletion increases Tiredness (if enabled)
- Eating from Inventory automatically restores vitals (if enabled)
- Using Skills can drain Stamina
- Casting Powers can drain MP
- Resting recovers all three

### 3.3 RP SYSTEM (Leveling & Progression)
**Toggles:**
- Leveling: ON/OFF
- Skills: ON/OFF
- Perks: ON/OFF
- Powers: ON/OFF
- Auto-generate starters: ON/OFF (3 skills + 3 perks + 5 powers on first turn)

**Character Sheet Card:**
- Level (numeric)
- XP (current / max to next level)
- Core Stats: STR, DEX, CON, INT, WIS, CHA (D&D standard, numeric 3-18)
- Skills list: name, rank (Novice → Adept → Expert → Master → Grandmaster → Legendary), linked stat
- Perks list: name (no rank, yes/no active)
- Powers list: name, rank, MP/SP cost, linked stat

**Behavior:**
- Level-up creates a related stronger power automatically (if enabled)
- Rank escalation is automatic (plain language, no fancy terms)
- Enchanted loot chance: percentage (default 15%) for random loot generation
- Skills linked to stats affect their effectiveness
- Powers linked to stats affect their effectiveness

**Adding New Entries:**
- `[ADD SKILL: Name | linked-stat | starting rank]`
- `[ADD PERK: Name]`
- `[ADD POWER: Name | cost | linked-stat]`

### 3.4 RP SYSTEM (Consumables & Drugs)
**Toggles:**
- Drug System: ON/OFF
- Injury Tracking: ON/OFF (for future use)

**Drugs & Consumables Card:**
- Format per drug: `name | cost | aliases | effects | duration`
- Example: `weed | 20 | bowl, bong, blunt, joint | -10 Stamina, -10 INT, -20 Hunger, -40 Thirst | 3 hours`
- Effects can target: HP, MP, Stamina, Hunger, Thirst, Tiredness, any Core Stat
- User adds/edits/deletes lines directly
- Changes apply next turn

**Behavior:**
- When consumed, effects apply immediately
- Duration tracked automatically
- After duration, effects wear off
- Can stack (up to reasonable limit or no limit, user's choice)

### 3.5 RP SYSTEM (Inner Thoughts)
**Toggle:** Inner Thoughts: ON/OFF

**Character Minds Card:**
- Tracked characters: list (user-editable)
- Current Memories: character name + quoted memory text (updated automatically by AI)
- Format: `Name: "Memory text here."`
- User can add/delete tracked characters
- Memories auto-update each turn (if enabled)

**Behavior:**
- AI receives list of tracked characters and their current thoughts about the player
- Thoughts inform NPC behavior and dialogue
- System cleans properly on undo

### 3.6 RP SYSTEM (Inner Opinions)
**Toggle:** Inner Opinions: ON/OFF

**Relationship Web Card:**
- Format: `Character Name > Target : emotion tags (comma-separated)`
- Example: `Father Aldric > Player : wary, protective, slightly disappointed`
- User can add/edit/delete relationships
- Opinions persist and update over time

**Behavior:**
- AI receives relationship web and factors it into NPC reactions
- Emotional tags guide AI tone and interaction style
- Opinions can shift based on events (AI-driven within narrative)

### 3.7 RP SYSTEM (Mind Journal)
**Toggle:** Journal auto-summaries: ON/OFF, `Journal every: [X] turns`

**Mind Journal Card:**
- Last summary (turn #): multi-line text recap
- Next auto-summary in: [X] turns (countdown)
- Action buttons: `[Force Summary]` `[Clear Journal]`

**Behavior:**
- Every X turns, AI generates a summary of recent events
- Summaries are stored and injected into context
- Helps AI maintain long-term memory of plot threads
- User can force a summary immediately or wipe the journal

### 3.8 RP SYSTEM (Quests)
**Toggles:**
- Story Arc Engine: ON/OFF
- Auto-update quests: ON/OFF
- Max active quests: [number] (default 5)

**Quests Card:**
- Header: `Active: [X] / [Max]`, `Auto-update: ON/OFF`
- List format: `[TYPE] Quest Title (progress %)` (one per line)
- Types: `[MAIN]` `[SIDE]` `[PERSONAL]`
- User can add new quests: `[ADD QUEST: Title | type | description]`

**Behavior:**
- Quests display in player's context each turn
- If auto-update ON: AI updates progress % based on narrative (realism check)
- If auto-update OFF: user updates manually
- Quest completion can trigger events
- Completed quests can be archived or removed

### 3.9 RP SYSTEM (Tickers)
**Toggle:** Universal Tickers: ON/OFF

**Universal Tickers Card:**
- List format: `Name | current% | interval | effect at 100%`
- Example: `Madness (Lovecraft) | 37% | interval: 4h | → at 100%: permanent Insanity`
- User can add: `[ADD TICKER: Name | interval | effect-at-100%]`
- Accepted intervals: 1m, 5m, 15m, 1h, 4h, 8h, 1d, 3d, 1w, 1mo, 1y, 2y

**Behavior:**
- Tickers increment automatically based on time progression
- When a ticker reaches 100%, its effect triggers (event, stat change, etc.)
- After triggering, ticker resets to 0 or stops (configurable)
- Multiple independent tickers can run simultaneously
- Used for decay, doom, anticipation, or recurring events

### 3.10 [SYSTEM] (Meta Shop Interface)
**Toggle:** [SYSTEM]: ON/OFF

**System Shop Card:**
- Current SP: [number]
- Multiplier (cashback on game currency): ON/OFF
- Multiplier value: [number] (default 1.5)

**Behavior:**
- User types `[SHOP]` in an action
- Story suspends, outputs holographic shop interface
- Categories: `[STATS]` `[SKILLS]` `[PERKS]` `[POWERS]` `[ITEMS]`
- User selects category (e.g., types `[SKILLS]`)
- Shop lists 5 upgradeable/buyable options with SP cost
- User selects item or `[SHOP]` to return to menu
- If Multiplier ON and user spends game currency: they get cashback
- Story resumes after purchase

**Pricing Rules:**
- SP cost is fixed per item and displayed
- Multiplier is optional: if ON, spending in-game currency gives X% cashback
- No infinite money; SP is earned through gameplay/achievements

### 3.11 Card Creator (Card Forge)
**Toggles:**
- Card Creator: ON/OFF
- Auto-detect card type: ON/OFF
- Pin config near top: ON/OFF
- Min turns cooldown: [number] (default 12)
- New cards do memory updates: ON/OFF
- Exclude ALL-CAPS titles: ON/OFF (default true)
- Detect from player input: ON/OFF (default true)
- Min age for detection: [number] (default 20)

**Card Forge Config Card:**
- Format Templates section (one per card type)
- Each template has:
  - Card type name
  - Rules (max character limit, focus area)
  - Format (exact fields required in the card)
  
**Card Types & Rules:**
1. **Character**
   - Max 450 characters
   - Focus: personality, current goals, secrets, emotional state
   - Fields: Name, Appearance, Personality, Goals, Secrets, Current Mood

2. **Location**
   - Max 400 characters
   - Focus: atmosphere, notable features, current conditions
   - Fields: Name, Description, Atmosphere, Notable Features, Current State

3. **Race**
   - Max 350 characters
   - Fields: Name, Physical Traits, Cultural Notes, Abilities

4. **Class/Faction**
   - Max 350 characters
   - Fields: Name, Role, Beliefs, Resources

5. **Other**
   - Max 400 characters
   - Use when type cannot be clearly detected
   - Fields: Name, Description, Relevance

**Behavior:**
- AI auto-detects card type from player or narrative context (if enabled)
- User can manually invoke: `/card Character "Sister Maren" she is starting to suspect the journal`
- System generates only the requested card (suspends story)
- Generation respects user-defined Rules + Format
- Cooldown prevents spam (min turns between auto-cards)
- ALL-CAPS titles can be excluded from detection
- New cards update Memory/Relationship systems (if enabled)

---

## SECTION 4: TIME & WORLD SYSTEM

### 4.1 Time Progression
**Toggle:** Time: ON/OFF

**Configure Time Card:**
- Starting date: [month] [day], [year]
- Adventure day count: auto-tracked
- Hours per turn: [number] (default 2, adjustable 1-4 for pacing)
- Current time: [hour:minute] (auto-calculated)

**Behavior:**
- Time advances automatically each turn
- Time is injected into AI context every turn (hidden block)
- Format in context: `Time: [hour:minute]`, `Date: [month name + day, year (Adventure day X)]`
- Used for scheduling, NPC routines, event timing

### 4.2 Weather & Seasons
**Toggles:** Weather: ON/OFF, Seasons: ON/OFF

**Behavior:**
- Seasons auto-calculated from in-game date
- Weather rolled once per adventure day (from seasonal tables)
- Injected into AI context same as time
- Format in context: `Season: [Season] | Weather: [Weather type]`
- Affects narrative tone, travel speed, available actions

### 4.3 Holidays & Events
**Toggles:** Holidays: ON/OFF, Events: ON/OFF

**Holidays Card (if enabled):**
- Format per holiday: `ADD: [Name] | [month] | [day] | [type] | [description]`
- Types: holiday, birthday, seasonal, religious, custom
- Example: `ADD: Christmas | 12 | 25 | religious | Nativity of Our Lord. Solemnity.`
- User can toggle: `[ON]` / `[OFF]` or `DEL: [Name]` to delete

**Behavior:**
- Fixed-date holidays are set-and-forget
- Moveable holidays require manual updates or approximate dates
- Only today's + upcoming holidays injected into context
- Tagged with type so AI treats them appropriately

### 4.4 Bills (if enabled)
**Toggle:** Bills: ON/OFF

**Behavior:**
- If enabled, periodic currency drain for maintenance/rent/upkeep
- Frequency and amount configurable in Configure card
- Forces player resource management

---

## SECTION 5: CONTEXT INJECTION FORMAT

### 5.1 The Hidden Block
**When sent to AI every turn (player cannot see):**

```
[CURRENT WORLD STATE -- use this as ground truth for the scene.
Answer any player questions about time, date, money, inventory, or weather from this block.
Prioritize any [EVENT] entries below when narrating the scene.]

[VITALS]
HP: X / MAX
MP: X / MAX
Stamina: X / MAX
Hunger: X%
Thirst: X%
Tiredness: X%

[PROGRESSION]
Level: X
XP: X / X_MAX
STR X  DEX X  CON X  INT X  WIS X  CHA X

[INVENTORY]
Item (qty) | Item (qty) | ...

[CURRENCY]
Gold: X | Silver: X | [etc]

[TIME & DATE]
Time: HH:MM
Date: Month Name + Day, Year (Adventure day X)
Season: [Season] | Weather: [Weather]

[ACTIVE HOLIDAYS]
Holiday Name (type): description

[ACTIVE QUESTS]
[TYPE] Quest Title (progress%)

[CHARACTER MINDS]
Character Name: "Current thought about player"

[RELATIONSHIP WEB]
Character Name > Player: emotion, emotion, emotion

[ACTIVE TICKERS]
Name: X% (effect at 100%)

[ACTIVE DRUGS/EFFECTS]
Effect Name: X turns remaining
```

### 5.2 Injection Rules
- Block is completely invisible to player
- Treated as ground truth by AI
- AI prioritizes injected data over narrative confusion
- Time/date/weather consistency enforced here
- All subsystems contribute their current state
- Order matters: VITALS → PROGRESSION → INVENTORY → CURRENCY → TIME → HOLIDAYS → QUESTS → MINDS → RELATIONSHIPS → TICKERS → EFFECTS

---

## SECTION 6: COMMAND STRUCTURE

### 6.1 Primary Help Command
**User Types:** `/help`

**Output:**
- Lists all available commands (short format)
- Explanation of each (one sentence)
- Points to specific card for deeper details

**Commands Available:**
- `/help` – Shows this guide
- `/keywords` – Lists all special terms (explanation of each)
- `/ah` – Aetherheart system status (all toggles, all card links)
- `/vitals` – Show current vitals in plain text
- `/sheet` – Show character sheet in plain text
- `/minds` – Show character minds summary
- `/memory` – Show mind journal summary
- `/quests` – Show active quests
- `/tickers` – Show active tickers
- `/shop` – Enter the [SYSTEM] shop
- `/card [type] [name] [details]` – Generate new card manually
- `/time` – Show current time and date
- `/weather` – Show current weather and season

### 6.2 Keywords Command
**User Types:** `/keywords`

**Output:**
- List of special terms used in Aetherheart
- One-line plain-English explanation per term
- Example: "Adept = mid-level rank. Comes after Novice."

---

## SECTION 7: IMPLEMENTATION CONSTRAINTS & RULES

### 7.1 State Management
- All state lives in `state.aetherheart` (JSON object)
- Never modify `state.ubis` or other systems' state
- State must cleanly serialize/deserialize
- On undo: state reverts to previous turn's snapshot
- On erase: state object deleted, fresh state on next turn
- On retry: state reverts to turn start

### 7.2 Editing Cards
- User can edit any value directly on any card
- Changes take effect "on the next turn"
- No immediate feedback, just update on turn parse
- Edits must be robust against typos/invalid input
- If invalid: ignore silently or default to last-good value

### 7.3 Auto-Generation
- Skills, Perks, Powers generated only on first turn (if enabled)
- Auto-cards generated on turn when conditions met (if enabled)
- No spam of cards; cooldown enforced
- Generated content respects card format rules

### 7.4 Undo/Erase/Retry Behavior
- State rolls back cleanly (no orphaned data)
- Card data reverts to last turn's snapshot
- Timers/tickers revert
- Memory/relationships revert
- No permanent side effects

---

## SECTION 8: VALIDATION & ERROR HANDLING

### 8.1 Type Checking
- Numeric fields must be numbers (clamp to min/max if needed)
- Boolean fields must be true/false only
- Percentage fields must be 0-100
- String fields stripped of leading/trailing whitespace

### 8.2 Range Clamping
- HP/MP/Stamina: cannot exceed max
- Hunger/Thirst/Tiredness: 0-100%
- Level: cannot exceed max (e.g., 20)
- XP: auto-level-up when max reached
- Stats (STR/DEX/etc.): 3-18 range

### 8.3 Missing Data
- If a card value is missing, use sensible default
- HP = 100, MP = 80, Stamina = 100 (if not specified)
- Level = 1, XP = 0 (if not specified)
- Stats = 10 (neutral D&D value, if not specified)
- Time = in-game sunrise (if not specified)

---

## SECTION 9: SCRIPT STRUCTURE OUTLINE

### 9.1 Library Script
- State initialization (new game setup)
- All subsystem functions (vitals, leveling, quests, tickers, etc.)
- Helper functions (clamp, calculate, format)
- Card generation (auto-cards, shop, commands)
- Linkage logic (stamina→tiredness, etc.)

### 9.2 Input Script
- Parse user input for `/commands`
- Parse direct card edits
- Route to appropriate subsystem handler
- Queue state changes for next turn

### 9.3 Context Script
- Build [CURRENT WORLD STATE] block
- Pull latest values from `state.aetherheart`
- Format each section (vitals, progression, inventory, etc.)
- Inject into AI context in correct order
- Handle visibility (player sees only story, AI sees hidden block)

### 9.4 Output Script
- After AI generates narrative, parse for special triggers
- Check for level-ups, quest completions, ticker triggers, etc.
- Update state accordingly
- Queue any auto-card generation
- Return modified narrative to player

---

## SECTION 10: READY FOR NEXT PHASE

Once this blueprint is locked in (no changes needed), we proceed with:

1. **Phase 1:** Full Library Script code
2. **Phase 2:** Input Script code
3. **Phase 3:** Context Script code
4. **Phase 4:** Output Script code
5. **Phase 5:** Integration testing and refinement

---

**END OF BLUEPRINT**

All information needed to code Aetherheart is now in this document.
Next step: Confirm blueprint is correct, then proceed with implementation.
