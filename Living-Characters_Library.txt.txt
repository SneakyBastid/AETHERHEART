/*
LIVING CHARACTERS - AUTONOMOUS SOCIAL ENGINE
A LivingNarratives project. Everything in this file is part of Living Characters.

Docs, install, configuration, model tips, pressure presets, and troubleshooting:
https://github.com/LivingNarratives/LivingCharacters
(GitHub is the single source of truth -- always use it for the current version.)

This file contains TWO independent systems that belong to Living Characters:
  1. Life Cards   - the autonomous NPC social-pressure engine (the bulk of this file).
  2. Thought Cards- optional, player-facing thought journals ("Name - Thoughts"; a
                    temporary 💭 marks the card that was just updated).
                    Thought Cards are NOT brain cards and NOT memory: they never enter
                    story context, are never read by the AI narrator, and never affect
                    behavior, Life Card creation, targeting, pressure, momentum, or any
                    story logic. The system only asks the model for a thought, captures
                    it, and stores it for the player to read later. See the "THOUGHT
                    CARD SYSTEM" module further down for full details.

Paste this whole block in: LIBRARY

CONTEXT TAB:
const modifier = (text) => {
  text = LivingCharacters("context", text);
  return { text };
};
modifier(text);

OUTPUT TAB:
const modifier = (text) => {
  text = LivingCharacters("output", text);
  return { text };
};
modifier(text);

INPUT TAB (optional):
const modifier = (text) => {
  text = LivingCharacters("input", text);
  return { text };
};
modifier(text);

(Existing adventures whose tabs still call ChaosGoblinV2(...) keep working via a
 compatibility alias at the bottom of this file.)
*/

function LivingCharacters(hook, hookText) {
  "use strict";

  const CFG = {
    VERSION: "2.63-thought-order-option-2026-07-13",

    // All cast / protagonist / pressures / pacing come from the editable config
    // Story Card below. No scenario-specific names live in engine logic.
    CONFIG_CARD_TITLE: "LIVING CHARACTERS CONFIG",
    CONFIG_CARD_KEY: "living-characters-config",
    CONFIG_CARD_TYPE: "Config",

    // RELATIONSHIPS live on their OWN optional Story Card, kept separate from the
    // engine-settings config card above (engine config vs. story relationship data).
    // The key is non-matching so the card is NEVER injected into context; the card is
    // found by key OR title, so a user only needs to match the TITLE. It is auto-created
    // as a comments-only template (zero active rules = today's behavior) so users can
    // discover and edit it. See parseRelationshipRules for the line format.
    REL_CARD_TITLE: "LIVING CHARACTERS RELATIONSHIPS",
    REL_CARD_KEY: "living-characters-relationships",
    REL_CARD_TYPE: "Relationships",
    // Old config cards used this exact NOTES text (not a roster). Recognized so it
    // is not mistaken for character names during the NOTES-roster migration.
    LEGACY_CONFIG_NOTES: "Living Characters setup. Edit cast, protagonist, pressures, and pacing here.",

    // User-facing card naming is "Life". The key prefix VALUE ("chaos-v2:") is kept
    // unchanged for save/card compatibility so existing Life cards (and legacy
    // "Chaos - " cards) are still found by key and auto-migrated to Life titles.
    LIFE_CARD_TYPE: "Life",
    LIFE_CARD_TITLE_PREFIX: "Life - ",
    LIFE_CARD_KEY_PREFIX: "chaos-v2:",

    // Seedling indicator shown while a Life card is unresolved; dropped on resolve.
    SEEDLING: "🌱",
    SEEDLING_IN_TITLE: true,

    DEBUG: false,
    DEBUG_CARD_TITLE: "Living Characters Debug",
    DEBUG_CARD_KEY: "lc-debug",
    DEBUG_CARD_TYPE: "Debug",

    // Safe last-resort output. AI Dungeon reports "empty response" for an empty OR
    // whitespace-only return; this zero-width space is non-empty and invisible to the
    // player. Centralized here so every hook's final boundary uses the same fallback.
    EMPTY_OUTPUT_FALLBACK: "\u200B",

    AUTONOMY_ENABLED: true,
    AUTONOMY_MAX_PENDING_AGE: 1,

    // Internal timing constants (NOT exposed on the public config card -- adjust here if
    // needed). Dormancy cadence: threads age toward dormant on this turn interval.
    THREAD_REMINDER_EVERY: 7,   // slower aging so cards do not burn out too quickly
    THREAD_REMINDER_MAX: 3,
    // Write-back checkpoint cadence: how often the LC_MEMORY write-back is offered.
    // INDEPENDENT of dormancy -- it never ages cards or affects lifespan.
    CHECKPOINT_EVERY: 3,        // more frequent chances for Life Cards to develop

    // Which active Life Cards are injected into context:
    //   "strict" - only scene-relevant cards (an involved NPC is present)
    //   "off"    - any active card (off-screen threads usable as world-state)
    //   "hybrid" - scene-relevant first, then fill remaining slots off-screen
    SCENE_RELEVANCE_MODE: "off",

    // HARD cap on simultaneously-open Life threads.
    // Config can choose 1 or 2 active cards.
    // When active slots are full, no new Life Card is rolled, seeded, replaced, or deleted.
    // The system waits until an active card archives naturally.
    MAX_ACTIVE_LIFE_CARDS: 2,
    ACTIVE_LIFE_STATUSES: ["active", "simmering", "surfaced"],
    // After this many dormancy-cadence appearances, an unresolved card goes dormant.
    THREAD_REMINDERS_BEFORE_DORMANT: 3,
    // No Life Card may be created before this turn (1 = allow from the first turn).
    MIN_TURN_BEFORE_FIRST_SEED: 1,
    // A card is flagged FRESH PRESSURE (urgent "reflect this now") for this many turns
    // after it fires. 0 = only the turn it fires.
    FRESH_THREAD_WINDOW: 1,

    // If the narrator shows a Life Card's actor but returns no <LC_MEMORY>, complete
    // the card from the seed instead of leaving the placeholder forever.
    AUTO_COMPLETE_ONSCREEN_SEEDS: true,

    // Whether a Life Card also triggers on its NPC target (not just its owner).
    // The protagonist is never added as a trigger. Override in the config card.
    TRIGGER_ON_TARGET: true,

    // Legacy/optional. Adds a shared trigger token to active cards + injects it into
    // the block. Only helps systems that RE-SCAN the modified context for triggers;
    // Hearthfire does not, so this is off by default now that the block carries the
    // full card details directly. Leave false unless you know your front-end re-scans.
    FORCE_ACTIVE_CARD_TRIGGER: false,
    ACTIVE_SHARED_TRIGGER: "life-thread",

    MAX_CONTEXT_CARDS: 4,
    MAX_EVENT_LOG: 12,
    // How many trailing context characters the scene detector scans (current scene).
    SCENE_SCAN_CHARS: 2000,
    // In second-person stories the protagonist is "you" and the name rarely appears,
    // but the protagonist is always present. When true (and a PROTAGONIST_NAME is set),
    // the protagonist always counts as in-scene so protagonist-targeted threads are
    // not disadvantaged. Set false for third-person stories that name the protagonist.
    PROTAGONIST_ALWAYS_PRESENT: true,

    // How often NPC Life Cards TARGET the protagonist. This is a TARGET-side
    // selection bias ONLY -- it does NOT let the protagonist OWN a Life Card and
    // never has the narrator author the player's private thoughts/interiority.
    //   "off"    - protagonist is excluded from target selection (pure NPC-to-NPC)
    //   "normal" - default; protagonist can be targeted with no special weighting
    //   "high"   - protagonist gets extra target weight (~3x an NPC); NPC-to-NPC still happens
    //   "always" - new cards target the protagonist when legal; safe fallback otherwise
    // If no PROTAGONIST_NAME is configured, every mode degrades safely to "normal".
    PROTAGONIST_INVOLVEMENT: "normal",

    // ---- Thought Card system (SEPARATE from Life Cards) --------------------
    // Player-facing thought journals. Thought Cards never enter context, are never
    // read by the AI, and never affect Life Card logic. Own config card, own state.
    THOUGHT_CONFIG_CARD_TITLE: "THOUGHT CARDS CONFIG",
    THOUGHT_CONFIG_CARD_KEY: "living-thoughts-config",
    THOUGHT_CONFIG_CARD_TYPE: "Config",
    // Title format is character-first for easy scanning. Base title: "Name - Thoughts".
    // The 💭 is a TEMPORARY "recently updated" marker prepended to the title (see below),
    // NOT a permanent part of it -- so the player can see which card just changed.
    THOUGHT_CARD_TITLE_SUFFIX: " - Thoughts",
    THOUGHT_MARKER: "💭",            // temporary "new thought / recently updated" title marker
    THOUGHT_MARKER_TURNS: 3,         // turns the 💭 marker stays before it clears
    THOUGHT_CARD_KEY_PREFIX: "lc-thoughts:",                // non-matching key -> never injected (UNCHANGED)
    THOUGHT_CARD_TYPE: "Custom",
    THOUGHTS_ENABLED_DEFAULT: false,    // opt-in; off by default
    THOUGHT_INTERVAL_DEFAULT: 5,        // turns between thought attempts
    THOUGHT_CHANCE_DEFAULT: 50,         // % chance per eligible turn
    // Thought Cards are NO LONGER limited to a fixed number of thoughts per character.
    // Storage is bounded by CHARACTER COUNT across two fields of the Story Card:
    //   - Entry holds the NEWEST thoughts up to ~THOUGHT_ENTRY_MAX_CHARS.
    //   - Notes holds the OLDER overflow up to ~THOUGHT_NOTES_MAX_CHARS.
    //   - Thoughts too old to fit in Notes are dropped (oldest-first). Numbers are
    //     PERMANENT and never reused, so dropping old thoughts never renumbers the rest.
    MAX_THOUGHTS_DEFAULT: 10,           // legacy default kept for save/config compatibility; NOT a cap anymore
    THOUGHT_ENTRY_MAX_CHARS: 1700,      // newest thoughts live in the Story Card Entry up to ~this many chars
    THOUGHT_NOTES_MAX_CHARS: 1700,      // older overflow lives in the Story Card Notes up to ~this many chars
    // Default is "scene": a character only gets a thought when they are actually in the
    // current scene. No silent fallback to the roster (that is the opt-in "roster" mode).
    THOUGHT_SCENE_MODE_DEFAULT: "scene", // scene | recent | roster
    // Display order of the numbered thoughts on a character's card. DISPLAY ONLY --
    // storage stays oldest->newest so the overflow/trim logic is unaffected.
    //   ascending  -> 1, 2, 3, 4  (oldest first; the original behavior)
    //   descending -> 4, 3, 2, 1  (newest first)
    THOUGHT_ORDER_DEFAULT: "ascending", // ascending | descending
    THOUGHT_SCENE_TIGHT_CHARS: 700,     // "scene" mode: how many trailing chars count as the CURRENT scene

    // Fallback pressures, used ONLY if the config card lists none. Generic and
    // scenario-agnostic; users override these in the config card's PRESSURES.
    DEFAULT_PRESSURES: [
      "attraction", "fondness", "friendship", "protectiveness",
      "curiosity", "envy", "jealousy", "rivalry",
      "betrayal", "resentment", "trust", "suspicion"
    ],

    // LIFE_CARD_INTERVAL default: turns between Life Card generation ATTEMPTS.
    // 0 = Off. Lower = more social activity, higher = less.
    DEFAULT_ACTIVITY_TURNS: 15,
    // Legacy SOCIAL_ACTIVITY (deprecated) -> LIFE_CARD_INTERVAL. Auto-converted
    // when an old config card has no LIFE_CARD_INTERVAL line.
    LEGACY_ACTIVITY_MAP: { off: 0, chaos: 5, busy: 10, balanced: 15, quiet: 20 },
    // TARGET_COOLDOWN is counted in Life Cards (how many must occur before the same
    // character can be selected again), not in turns.
    DEFAULT_TARGET_COOLDOWN: 3,

    // Seed dedup TTL: a used owner|target|pressure signature blocks re-seeding for
    // this many TURNS, then expires. Turn-based ON PURPOSE (not seed-count-based):
    // with fixed RELATIONSHIPS pairs the blocked signature can be the ONLY possible
    // seed, and a blocked seed never advances the seed count -- count-based aging
    // would deadlock. Turns always advance.
    SEED_DEDUP_TTL_TURNS: 30,

    // ---- World Event cards -------------------------------------------------
    // A world event is a Life Card whose owner is the WORLD, not a person. It rides the
    // SAME machinery as any other card -- slot cap, pacing, seeding, injection block,
    // round-robin -- so it is not a second system. It is excluded from every CHARACTER
    // path via kind === "event" (never an actor, never a target, never "in scene"), and
    // because it takes a normal card slot, MAX_ACTIVE_CARDS alone decides whether it can
    // coexist with a character card (set 1 to make the event the only card in play).
    // The bucket key contains ":" -- cleanName can never produce that, so it can never
    // collide with a real character's bucket, and the narrator can never address it.
    EVENT_BUCKET_KEY: "event:world",
    EVENT_OWNER_LABEL: "The World",
    EVENT_CARD_TITLE: "Event - World",
    EVENT_CARD_TYPE: "Event",
    // Round-robin cycle:
    //   relationship -> event -> random -> relationship -> random -> repeat
    // Events take 1 seat in 5; relationship and random take 2 each. Re-tune the cadence
    // anytime by editing this array. A seat that cannot produce is SKIPPED by the fallback
    // loop in maybeCreateSeed, so an empty WORLD_EVENTS list simply degrades to the
    // relationship/random seats and nothing stalls.
    SEED_CYCLE: ["relationship", "event", "random", "relationship", "random"],

    STATUS_VALUES: ["active", "simmering", "surfaced", "dormant", "resolved"]
  };

  function getGlobalText(fallback) {
    if (typeof hookText === "string") return hookText;
    if (typeof globalThis.text === "string") return globalThis.text;
    return fallback || "";
  }

  function setGlobalText(value) {
    if (typeof globalThis.text === "string") globalThis.text = String(value || "");
  }

  // ---- Generic configuration card -----------------------------------------
  // All cast / protagonist / pressures / pacing are read from an editable Story
  // Card. The engine holds no scenario-specific names.
  function defaultConfigEntry() {
    return [
      "LIVING CHARACTERS",
      "",
      "For installation, configuration options, model recommendations, pressure",
      "presets, updates, and troubleshooting, visit:",
      "https://github.com/LivingNarratives/LivingCharacters",
      "Always refer to the GitHub documentation for the most current version.",
      "",
      "( Characters go in this card's NOTES, one name per line. )",
      "",
      "PROTAGONIST_NAME:",
      "None",
      "",
      "SCENE_RELEVANCE_MODE:",
      "off",
      "",
      "PRESSURES:",
      "friendship",
      "trust",
      "curiosity",
      "protectiveness",
      "jealousy",
      "rivalry",
      "attraction",
      "teasing",
      "gossip",
      "misunderstanding",
      "",
      "WORLD_EVENTS:",
      "( Optional. One event per line -- type your own. )",
      "( The narrator decides which characters get pulled in. )",
      "( An event takes a card slot like any other Life Card. )",
      "( Leave empty for no events. )",
      "( A gun duel erupts in the street )",
      "( A brawl breaks out in the saloon )",
      "",
      "LIFE_CARD_INTERVAL:",
      "15",
      "",
      "TARGET_COOLDOWN:",
      "3",
      "",
      "MAX_ACTIVE_CARDS:",
      "2",
      "",
      "PROTAGONIST_INVOLVEMENT:",
      "normal",
      "Options: off | normal | high | always",
      "",
      "( Relationship steering is OPTIONAL and lives on a SEPARATE Story Card )",
      "( titled \"LIVING CHARACTERS RELATIONSHIPS\". Create that card to steer who )",
      "( targets whom; leave it out for fully random behavior. See GitHub. )"
    ].join("\n");
  }

  // The character roster lives in the config card's NOTES (description).
  function defaultConfigNotes() {
    return [
      "( Add one character name per line below. See GitHub for help. )",
      "Characters:"
    ].join("\n");
  }

  // Parse a roster from NOTES text: one name per line, trimmed, blanks and
  // ( comments ) ignored, multi-word names preserved.
  function isConfigSectionLabel(value) {
    const line = String(value || "").trim();
    const colon = line.indexOf(":");
    if (colon === -1 || line.slice(colon + 1).trim()) return false;
    const head = line.slice(0, colon).trim().toLowerCase().replace(/\s+/g, "_");
    const labels = [
      "protagonist_name", "protagonist_involvement", "characters", "pressures",
      "world_events",
      "life_card_interval", "social_activity", "target_cooldown", "max_active_cards",
      "scene_relevance_mode", "scene_relevance", "trigger_on_target",
      "force_active_card_trigger", "protagonist_always_present", "world_events"
    ];
    return labels.indexOf(head) !== -1;
  }

  function parseRoster(notes) {
    const lines = String(notes || "").replace(/\r/g, "").split("\n");
    const out = [];
    for (let i = 0; i < lines.length; i++) {
      const raw = lines[i].trim();
      if (!raw || raw.charAt(0) === "(" || isConfigSectionLabel(raw)) continue;
      const v = cleanName(raw);
      if (v && out.indexOf(v) === -1) out.push(v);
    }
    return out;
  }

  function ensureConfigCard() {
    const existing = findStoryCardByKeys(CFG.CONFIG_CARD_KEY) || findStoryCardByTitle(CFG.CONFIG_CARD_TITLE);
    if (existing) return existing;
    return createOrPatchStoryCard(
      CFG.CONFIG_CARD_TITLE,
      CFG.CONFIG_CARD_TYPE,
      CFG.CONFIG_CARD_KEY,
      defaultConfigEntry(),
      defaultConfigNotes()
    );
  }

  // Auto-created template for the SEPARATE relationships card. Only lines BELOW the
  // "Relationships:" header are parsed as rules (see parseRelationshipRules), so the
  // preamble and the two example lines above it are NEVER treated as live rules -- a
  // freshly created card carries ZERO active rules and behavior is identical to before
  // until the user adds a line under "Relationships:".
  function defaultRelationshipsEntry() {
    return [
      "Per-character targeting",
      "",
      "Add your relationships under the \"Relationships:\" heading.",
      "",
      "Example:",
      "Jessica>Sam=jealousy,attraction",
      "or",
      "Jessica > Sam = jealousy,attraction",
      "",
      "Relationships:",
      ""
    ].join("\n");
  }

  // Auto-create the relationships card if it is missing, so users can discover and edit
  // it -- mirroring ensureConfigCard/ensureThoughtConfigCard. The template has no rules
  // under its "Relationships:" header, so it leaves behavior exactly as before.
  function ensureRelationshipsCard() {
    const existing = findStoryCardByKeys(CFG.REL_CARD_KEY) || findStoryCardByTitle(CFG.REL_CARD_TITLE);
    if (existing) return existing;
    return createOrPatchStoryCard(
      CFG.REL_CARD_TITLE,
      CFG.REL_CARD_TYPE,
      CFG.REL_CARD_KEY,
      defaultRelationshipsEntry(),
      "( Optional per-character targeting. Add rules under the Relationships: line. This card never enters the story. )"
    );
  }

  function parseConfigText(text, keysOverride) {
    const KEYS = keysOverride || ["protagonist_name", "characters", "pressures", "life_card_interval", "social_activity", "target_cooldown", "max_active_cards", "scene_relevance_mode", "scene_relevance", "trigger_on_target", "force_active_card_trigger", "protagonist_always_present", "protagonist_involvement", "world_events"];
    const lines = String(text || "").replace(/\r/g, "").split("\n");
    const sections = {};
    let current = "";
    for (let i = 0; i < lines.length; i++) {
      const line = lines[i].trim();
      if (!line) continue;
      const colon = line.indexOf(":");
      const head = (colon !== -1 ? line.slice(0, colon) : line).trim().toLowerCase().replace(/\s+/g, "_");
      if (KEYS.indexOf(head) !== -1) {
        current = head;
        if (!sections[current]) sections[current] = [];
        const after = colon !== -1 ? line.slice(colon + 1).trim() : "";
        if (after) sections[current].push(after);
        continue;
      }
      if (current) sections[current].push(line);
    }
    return sections;
  }

  function configFirst(sections, key, dflt) {
    const arr = sections[key];
    if (arr) {
      for (let i = 0; i < arr.length; i++) {
        if (arr[i] && arr[i].charAt(0) !== "(") return arr[i].trim();
      }
    }
    return dflt;
  }

  function configList(sections, key) {
    const arr = sections[key] || [];
    const out = [];
    for (let i = 0; i < arr.length; i++) {
      if (!arr[i] || arr[i].charAt(0) === "(" || (key === "characters" && isConfigSectionLabel(arr[i]))) continue;
      const v = cleanName(arr[i]);
      if (v && out.indexOf(v) === -1) out.push(v);
    }
    return out;
  }

  // Like configList, but PRESERVES the raw line text. configList runs cleanName, which
  // strips punctuation and truncates at 50 chars -- right for names, wrong for an event
  // sentence like "A gun duel erupts in the street."
  function configTextList(sections, key) {
    const arr = sections[key] || [];
    const out = [];
    for (let i = 0; i < arr.length; i++) {
      const line = String(arr[i] || "").trim();
      if (!line || line.charAt(0) === "(" || isConfigSectionLabel(line)) continue;
      const v = cleanText(line);
      if (v && out.indexOf(v) === -1) out.push(v);
    }
    return out;
  }

  function toIntOr(value, dflt) {
    const n = parseInt(String(value).replace(/[^0-9-]/g, ""), 10);
    return isNaN(n) ? dflt : n;
  }

  function toBoolOr(value, dflt) {
    if (value == null) return dflt;
    const s = String(value).trim().toLowerCase();
    if (s === "true" || s === "yes" || s === "on" || s === "1") return true;
    if (s === "false" || s === "no" || s === "off" || s === "0") return false;
    return dflt;
  }

  function buildRuntimeConfig() {
    ensureConfigCard();
    const card = findStoryCardByKeys(CFG.CONFIG_CARD_KEY) || findStoryCardByTitle(CFG.CONFIG_CARD_TITLE);
    const sections = parseConfigText(card && card.entry);

    let protagonist = cleanName(configFirst(sections, "protagonist_name", ""));
    if (/^none$/i.test(protagonist) || /^n\/?a$/i.test(protagonist)) protagonist = "";

    // Roster: prefer the card NOTES (description). The old default NOTES string is
    // not a roster, so ignore it. Fall back to the legacy ENTRY CHARACTERS section
    // for older cards so they do not break.
    const notes = (card && card.description) || "";
    let rosterSource = "NOTES";
    let characters = (String(notes).trim() === CFG.LEGACY_CONFIG_NOTES) ? [] : parseRoster(notes);
    if (!characters.length) {
      characters = configList(sections, "characters");
      rosterSource = characters.length ? "ENTRY (legacy)" : "none";
    }
    characters = characters.filter(function(n) { return n && n !== protagonist; });

    let pressures = configList(sections, "pressures").map(function(p) { return p.toLowerCase(); });
    if (!pressures.length) pressures = CFG.DEFAULT_PRESSURES.slice();

    // WORLD_EVENTS: optional authored list, one event per line. Raw text is preserved.
    // Empty (the default) -> the "event" seat in the round-robin falls through to the
    // character pools, so an existing story behaves exactly as before.
    const worldEvents = configTextList(sections, "world_events");

    // LIFE_CARD_INTERVAL -> turns between Life Card attempts. 0 disables generation.
    // SOCIAL_ACTIVITY is deprecated; if an old card has it but no LIFE_CARD_INTERVAL,
    // auto-convert the keyword to the matching interval (no silent Balanced fallback).
    const intervalRaw = configFirst(sections, "life_card_interval", null);
    const legacyRaw = configFirst(sections, "social_activity", null);
    let legacyActivityUsed = false;
    let interval;
    if (intervalRaw != null) {
      interval = toIntOr(intervalRaw, CFG.DEFAULT_ACTIVITY_TURNS);
    } else if (legacyRaw != null) {
      legacyActivityUsed = true;
      const key = String(legacyRaw).toLowerCase().replace(/\s+/g, "");
      interval = CFG.LEGACY_ACTIVITY_MAP[key] != null
        ? CFG.LEGACY_ACTIVITY_MAP[key]
        : toIntOr(legacyRaw, CFG.DEFAULT_ACTIVITY_TURNS);
    } else {
      interval = CFG.DEFAULT_ACTIVITY_TURNS;
    }
    const activityOff = (interval <= 0);
    const activityTurns = activityOff ? 0 : Math.max(1, interval);

    const targetCooldown = Math.max(0, toIntOr(configFirst(sections, "target_cooldown", CFG.DEFAULT_TARGET_COOLDOWN), CFG.DEFAULT_TARGET_COOLDOWN));
    // Configurable, clamped to 1-2 (3+ not allowed for now).
    //   1 = focused mode (one storyline at a time)
    //   2 = layered mode (two interwoven threads)
    const maxActive = Math.max(1, Math.min(2, toIntOr(configFirst(sections, "max_active_cards", CFG.MAX_ACTIVE_LIFE_CARDS), CFG.MAX_ACTIVE_LIFE_CARDS)));
    const triggerOnTarget = toBoolOr(configFirst(sections, "trigger_on_target", null), CFG.TRIGGER_ON_TARGET);
    const forceActiveCardTrigger = toBoolOr(configFirst(sections, "force_active_card_trigger", null), CFG.FORCE_ACTIVE_CARD_TRIGGER);
    const protagonistAlwaysPresent = toBoolOr(configFirst(sections, "protagonist_always_present", null), CFG.PROTAGONIST_ALWAYS_PRESENT);

    // Scene relevance mode: strict | off | hybrid (validated; default strict).
    // Accept both SCENE_RELEVANCE_MODE and the SCENE_RELEVANCE alias. Prefer
    // SCENE_RELEVANCE_MODE if present, else SCENE_RELEVANCE, else the default.
    let sceneMode = String(
      configFirst(
        sections,
        "scene_relevance_mode",
        configFirst(sections, "scene_relevance", CFG.SCENE_RELEVANCE_MODE)
      )
    ).toLowerCase().replace(/\s+/g, "");
    if (sceneMode !== "strict" && sceneMode !== "off" && sceneMode !== "hybrid") sceneMode = CFG.SCENE_RELEVANCE_MODE;

    // Two INDEPENDENT cadences (decoupled), hardwired internally and NOT exposed on the
    // public config card. Adjust the CFG constants if needed.
    //   CHECKPOINT_EVERY      -> write-back (LC_MEMORY) opportunity only. No lifespan effect.
    //   THREAD_REMINDER_EVERY -> dormancy/reminder aging only (card lifespan).
    const checkpointEvery = CFG.CHECKPOINT_EVERY;
    const threadReminderEvery = CFG.THREAD_REMINDER_EVERY;

    // Protagonist involvement: off | normal | high | always (validated; default normal).
    // TARGET-side bias only -- never makes the protagonist an owner. With no protagonist
    // configured the bias is meaningless, so it degrades safely to "normal".
    let involvement = String(configFirst(sections, "protagonist_involvement", CFG.PROTAGONIST_INVOLVEMENT)).toLowerCase().replace(/\s+/g, "");
    if (involvement !== "off" && involvement !== "normal" && involvement !== "high" && involvement !== "always") involvement = CFG.PROTAGONIST_INVOLVEMENT;
    if (!protagonist) involvement = "normal";

    // RELATIONSHIPS: optional directed steering rules (seed-time only), read from a
    // SEPARATE Story Card (engine config vs. story relationship data). If the card is
    // absent, rel is empty and the engine behaves exactly as before. Rules are read
    // from the card's ENTRY and NOTES (either place works), parsed AFTER the roster and
    // protagonist so names can canonicalize against the roster.
    ensureRelationshipsCard();
    const relCard = findStoryCardByKeys(CFG.REL_CARD_KEY) || findStoryCardByTitle(CFG.REL_CARD_TITLE);
    const relationshipsCardFound = !!relCard;
    const relText = relCard ? (String(relCard.entry || "") + "\n" + String(relCard.description || "")) : "";
    const relLines = relText ? relText.replace(/\r/g, "").split("\n") : [];
    const rel = parseRelationshipRules(relLines, characters, protagonist);

    // A RELATIONSHIPS section left over in the OLD location (the main config card) is
    // no longer read here -- flag it so the debug card can tell the user to move it.
    const mainConfigHadRelationships = /(^|\n)\s*relationships\s*:/i.test(String(card && card.entry) || "");

    // Expand the cast so relationship rules do NOT require duplicating names in the
    // roster: runtime actors = roster actors + relationship owners + relationship
    // targets. Relationship-only names become valid characters (scene detection,
    // random target pools, etc.); the protagonist is still excluded from the NPC cast.
    // rosterCharacters is captured BEFORE expansion: it is the set eligible for RANDOM
    // owner selection. Relationship-only TARGETS are added to the cast (valid targets +
    // scene-detectable) but are NOT random owners unless they also appear here or hold
    // their own rule -- see ownerCandidates().
    const rosterActorCount = characters.length;
    const rosterCharacters = characters.slice();
    const relNameSet = {};
    Object.keys(rel.rules).forEach(function(owner) {
      relNameSet[owner] = true;
      rel.rules[owner].forEach(function(r) { relNameSet[r.target] = true; });
    });
    const relNames = Object.keys(relNameSet).filter(function(n) { return n && n !== protagonist; });
    let relActorCount = 0;
    for (let ri = 0; ri < relNames.length; ri++) {
      if (characters.indexOf(relNames[ri]) === -1) relActorCount++;
    }
    characters = unique(characters.concat(relNames)).filter(function(n) { return n && n !== protagonist; });

    return {
      protagonist: protagonist,
      characters: characters,
      rosterCharacters: rosterCharacters,
      rosterSource: rosterSource,
      pressures: pressures,
      worldEvents: worldEvents,
      activityOff: activityOff,
      activityTurns: activityTurns,
      legacyActivityUsed: legacyActivityUsed,
      targetCooldown: targetCooldown,
      maxActive: maxActive,
      sceneRelevanceMode: sceneMode,
      triggerOnTarget: triggerOnTarget,
      forceActiveCardTrigger: forceActiveCardTrigger,
      protagonistAlwaysPresent: protagonistAlwaysPresent,
      protagonistInvolvement: involvement,
      relationships: rel.rules,
      relationshipNotes: rel.notes,
      relationshipCount: rel.count,
      relationshipsCardFound: relationshipsCardFound,
      mainConfigHadRelationships: mainConfigHadRelationships,
      rosterActorCount: rosterActorCount,
      relActorCount: relActorCount,
      checkpointEvery: checkpointEvery,
      threadReminderEvery: threadReminderEvery
    };
  }

  function ensureState() {
    if (!globalThis.state || typeof state !== "object") globalThis.state = {};
    // NOTE: the persistent save key is intentionally kept as `chaosGoblinV2` for
    // backward compatibility — existing adventures store their Life Threads here.
    // Renaming it would orphan every saved thread, so the key stays; only the
    // surrounding code/naming is modernized. `lc` below is the live engine state.
    if (!state.chaosGoblinV2) {
      state.chaosGoblinV2 = {
        version: CFG.VERSION,
        turn: 0,
        pendingSeed: null,
        cards: {},
        actorSeedIndex: {},
        seedCount: 0,
        lastSeedAttemptTurn: 0,
        recentSeeds: [],
        recentMemory: [],
        seedPhase: "relationship"
      };
    }
    const cg = state.chaosGoblinV2;
    cg.version = CFG.VERSION;
    if (!cg.cards || typeof cg.cards !== "object") cg.cards = {};
    if (!cg.actorSeedIndex || typeof cg.actorSeedIndex !== "object") cg.actorSeedIndex = {};
    if (typeof cg.seedCount !== "number") cg.seedCount = 0;
    // Round-robin phase: which owner type is TRIED first next seed. Default to
    // "relationship" so the very first generated card prefers a relationship rule.
    if (cg.seedPhase !== "relationship" && cg.seedPhase !== "random" && cg.seedPhase !== "event") cg.seedPhase = "relationship";
    // Round-robin CYCLE index into CFG.SEED_CYCLE. Migrated from the legacy 2-way
    // seedPhase toggle so existing saves keep their place in the rotation.
    if (typeof cg.seedCycleIndex !== "number") cg.seedCycleIndex = (cg.seedPhase === "random") ? 1 : 0;
    if (!Array.isArray(cg.recentSeeds)) cg.recentSeeds = [];
    // Migrate legacy recentSeeds entries (plain signature strings, never-expiring)
    // to { sig, turn } stamped with the current turn so they age out normally.
    for (let i = 0; i < cg.recentSeeds.length; i++) {
      if (typeof cg.recentSeeds[i] === "string") {
        cg.recentSeeds[i] = { sig: cg.recentSeeds[i], turn: cg.turn || 0 };
      }
    }
    if (!Array.isArray(cg.recentMemory)) cg.recentMemory = [];
    return cg;
  }

  function cleanText(value) {
    return String(value || "")
      .replace(/\r/g, "")
      .replace(/[ \t]+\n/g, "\n")
      .replace(/\n{3,}/g, "\n\n")
      .trim();
  }

  // Leading whitespace run (spaces/tabs/newlines) of a string, or "". Used to PRESERVE the
  // separator the model puts before a continuation -- cleanText's trim() would remove it and
  // jam the new narration onto the prior story text.
  function leadingWhitespace(s) {
    const m = /^[ \t\r\n]+/.exec(String(s || ""));
    return m ? m[0] : "";
  }

  // Normalize a leading-whitespace run to exactly ONE safe separator so restoring it never
  // reintroduces sloppy spacing: a paragraph break, a single newline, or a single space.
  function leadSeparator(ws) {
    if (/\n[ \t]*\n/.test(ws)) return "\n\n";
    if (/\n/.test(ws)) return "\n";
    if (/[ \t]/.test(ws)) return " ";
    return "";
  }

  function cleanName(value) {
    let s = String(value || "").replace(/[^A-Za-z0-9 _'-]/g, " ").trim();
    s = s.replace(/\s+/g, " ");
    return s.slice(0, 50);
  }

  function keyName(name) {
    return cleanName(name).toLowerCase().replace(/[^a-z0-9]+/g, "-").replace(/^-+|-+$/g, "");
  }

  function containsWholeWord(text, word) {
    text = String(text || "").toLowerCase();
    word = String(word || "").toLowerCase();
    if (!word) return false;
    let pos = text.indexOf(word);
    while (pos !== -1) {
      const before = pos === 0 ? " " : text.charAt(pos - 1);
      const after = pos + word.length >= text.length ? " " : text.charAt(pos + word.length);
      const bWord = /[a-z0-9_]/.test(before);
      const aWord = /[a-z0-9_]/.test(after);
      if (!bWord && !aWord) return true;
      pos = text.indexOf(word, pos + 1);
    }
    return false;
  }

  function unique(list) {
    const out = [];
    for (let i = 0; i < list.length; i++) {
      const item = cleanName(list[i]);
      if (item && out.indexOf(item) === -1) out.push(item);
    }
    return out;
  }

  function playerName() {
    return LC.protagonist || "";
  }

  function npcRoster() {
    const p = playerName();
    return unique(LC.characters || []).filter(function(name) {
      return name !== p;
    });
  }

  // Scene detector: which configured characters (protagonist + roster) appear in
  // the recent context. Scans the TAIL of the context (the current scene), matches
  // whole words/phrases so multi-word names work, and includes the protagonist so
  // the engine knows when the player is present.
  // Text for the CURRENT scene. Prefer recent story actions (history), which EXCLUDE
  // story-card entries, memory, author's note, and injected notes — so strict scene
  // relevance means "in the visible scene", not "named somewhere in context".
  function recentSceneText(contextFallback) {
    const cg = ensureState();
    if (Array.isArray(globalThis.history) && history.length) {
      let s = "";
      for (let i = history.length - 1; i >= 0 && s.length < CFG.SCENE_SCAN_CHARS; i--) {
        const h = history[i];
        const t = (h && typeof h.text === "string") ? h.text : (typeof h === "string" ? h : "");
        if (t) s = t + "\n" + s;
      }
      if (s.replace(/\s/g, "")) { cg.sceneSource = "history"; return s; }
    }
    cg.sceneSource = "context-tail";
    const text = String(contextFallback || "");
    return text.length > CFG.SCENE_SCAN_CHARS ? text.slice(-CFG.SCENE_SCAN_CHARS) : text;
  }

  function sceneActors(text) {
    const recent = recentSceneText(text);
    const p = playerName();
    const found = [];
    // Protagonist is present by default (second-person "you" rarely names them).
    if (p && LC.protagonistAlwaysPresent) found.push(p);
    // NPCs are detected by name in the recent scene text.
    const names = npcRoster();
    for (let i = 0; i < names.length; i++) {
      if (names[i] && containsWholeWord(recent, names[i])) found.push(names[i]);
    }
    // Also detect the protagonist by name (third-person stories that name them).
    if (p && found.indexOf(p) === -1 && containsWholeWord(recent, p)) found.push(p);
    return unique(found);
  }

  function randomInt(max) {
    return Math.floor(Math.random() * Math.max(1, max));
  }

  function choose(list) {
    if (!list || !list.length) return "";
    return list[randomInt(list.length)];
  }

  function weightedChoice(pairs) {
    let total = 0;
    for (let i = 0; i < pairs.length; i++) total += Math.max(0, Number(pairs[i][1]) || 0);
    if (total <= 0) return pairs.length ? pairs[0][0] : "";
    let roll = Math.random() * total;
    for (let j = 0; j < pairs.length; j++) {
      roll -= Math.max(0, Number(pairs[j][1]) || 0);
      if (roll <= 0) return pairs[j][0];
    }
    return pairs[pairs.length - 1][0];
  }

  function ensureCardBucket(owner) {
    owner = cleanName(owner);
    if (!owner) return null;
    const cg = ensureState();
    if (!cg.cards[owner]) {
      cg.cards[owner] = {
        owner: owner,
        target: "",
        pressure: "",
        momentum: "",
        event: "",
        status: "simmering",
        reminderCount: 0,
        eventLog: []
      };
    }
    const b = cg.cards[owner];
    if (!Array.isArray(b.eventLog)) b.eventLog = [];
    if (typeof b.reminderCount !== "number") b.reminderCount = 0;
    return b;
  }

  function eventFingerprint(line) {
    return cleanText(line).toLowerCase().replace(/[^a-z0-9]+/g, " ").trim().slice(0, 120);
  }

  function appendEventLog(bucket, line) {
    line = cleanText(line);
    if (!bucket || !line) return;
    const fp = eventFingerprint(line);
    for (let i = 0; i < bucket.eventLog.length; i++) {
      if (eventFingerprint(bucket.eventLog[i]) === fp) return;
    }
    bucket.eventLog.push(line);
    while (bucket.eventLog.length > CFG.MAX_EVENT_LOG) bucket.eventLog.shift();
  }

  function applyMemory(memory) {
    const owner = cleanName(memory.owner);
    if (!owner) return false;
    const bucket = ensureCardBucket(owner);
    if (!bucket) return false;

    // Narrator authors the specific EVENT. System keeps PRESSURE/TARGET/MOMENTUM,
    // but fill them from memory if the card has none yet.
    if (memory.event) bucket.event = cleanText(memory.event);
    if (memory.target && !cleanText(bucket.target)) bucket.target = cleanName(memory.target);
    if (memory.pressure && !cleanText(bucket.pressure)) bucket.pressure = cleanText(memory.pressure);
    if (memory.momentum && !cleanText(bucket.momentum)) bucket.momentum = cleanText(memory.momentum);
    // Optional first-person thought: update only when the narrator supplies a non-empty
    // value; never blank an existing thought. Display-only, never used in engine logic.
    // NOTE: this legacy OWNER_THOUGHT lives on the Life Card and is fed by the LC_MEMORY
    // write-back (XML-style block authored by the narrator). It is SEPARATE from the
    // Thought Card system: Thought Cards do NOT use LC_MEMORY, XML, or narrator write-back
    // -- they use the leading-parenthetical capture and store to their own "💭 Thoughts"
    // cards. The two are independent; see the THOUGHT CARD SYSTEM module below.
    if (cleanText(memory.owner_thought)) bucket.ownerThought = cleanText(memory.owner_thought);

    const status = cleanText(memory.status).toLowerCase();
    bucket.status = CFG.STATUS_VALUES.indexOf(status) !== -1 ? status : (bucket.status || "active");
    bucket.reminderCount = 0; // narrator touched it; restart the dormancy clock

    appendEventLog(bucket, memory.log || memory.event);

    // Resolved threads archive permanently: drop the Story Card. HISTORY stays in
    // state so the character keeps a long-term social record.
    if (bucket.status === "resolved") {
      removeStoryCardByKeys(storyCardIdToken(owner));
    }

    const cg = ensureState();
    cg.recentMemory.push(owner + ": " + (memory.event || memory.log || ""));
    while (cg.recentMemory.length > 12) cg.recentMemory.shift();

    if (cg.pendingSeed && (owner === cg.pendingSeed.actor || owner === cg.pendingSeed.target)) {
      cg.pendingSeed = null;
    }
    return true;
  }

  function renderCardEntry(bucket) {
    const status = cleanText(bucket.status) || "active";
    // Dormant = archived record: STATUS only. HISTORY is in the card description.
    if (status.toLowerCase() === "dormant") return "STATUS: dormant";
    if (!cardHasContent(bucket)) return "";
    const target = cleanText(bucket.target) || "someone";
    const pressure = cleanText(bucket.pressure) || "unspoken social pressure";
    const momentum = cleanText(bucket.momentum) || "low";
    const occurrence = cleanText(bucket.event);
    const ownerThought = cleanText(bucket.ownerThought);
    // Seedling shows only while the thread is active; dropped when dormant/resolved.
    const statusLine = lifeStatusIsActive(status) ? (CFG.SEEDLING + " " + status) : status;
    // Compact one-line-per-field format (no blank lines) to keep Story Cards small.
    const out = ["TARGET: " + target, "PRESSURE: " + pressure];
    if (occurrence) out.push("OCCURRENCE: " + occurrence); // only if narrator authored it
    if (ownerThought) out.push("OWNER_THOUGHT: " + ownerThought); // LC_MEMORY-fed (legacy)
    out.push("MOMENTUM: " + momentum, "STATUS: " + statusLine);
    return out.join("\n");
  }

  function renderCardDescription(bucket) {
    const lines = bucket.eventLog || [];
    if (!lines.length) return "HISTORY:\n";
    const out = ["HISTORY:"];
    for (let i = 0; i < lines.length; i++) out.push("[" + (i + 1) + "] " + lines[i]);
    return out.join("\n");
  }

  // Stable per-character identity token. Never changes across a character's life
  // (even when the target changes). Used ONLY for lookup/removal.
  function storyCardIdToken(owner) {
    return CFG.LIFE_CARD_KEY_PREFIX + keyName(owner);
  }

  // Visible trigger list = identity token + owner name + (NPC target, if enabled).
  // Joined with "," and no trailing space. The protagonist, empty targets, and
  // "someone" placeholders are never added as triggers.
  function storyCardTriggers(bucket) {
    const owner = cleanName(bucket.owner);
    let triggers = storyCardIdToken(owner) + "," + owner;
    if (LC.triggerOnTarget) {
      const target = cleanName(bucket.target);
      const p = playerName();
      if (target && target !== owner && target !== p && target.toLowerCase() !== "someone") {
        triggers += "," + target;
      }
    }
    // Shared trigger token: only added here, and syncCards only runs for ACTIVE
    // cards, so dormant/resolved cards never carry it.
    if (LC.forceActiveCardTrigger && CFG.ACTIVE_SHARED_TRIGGER) {
      triggers += "," + CFG.ACTIVE_SHARED_TRIGGER;
    }
    // Experiment: add the second-person word "you" as a trigger on every card so that in a
    // second-person story (where "you" is ever-present) AI Dungeon matches every card.
    triggers += ",you";
    return triggers;
  }

  // Identity = the FIRST trigger token only (normalized), never the full list.
  function keyIdToken(keys) {
    const s = String(keys || "");
    const comma = s.indexOf(",");
    return (comma === -1 ? s : s.slice(0, comma)).replace(/\s+/g, "").toLowerCase();
  }

  function cardTitle(owner, status) {
    const prefix = (CFG.SEEDLING_IN_TITLE && lifeStatusIsActive(status)) ? (CFG.SEEDLING + " ") : "";
    return prefix + CFG.LIFE_CARD_TITLE_PREFIX + owner;
  }

  function isUnnamedish(value) {
    const s = String(value || "").trim().toLowerCase();
    return !s || s === "unnamed" || s === "untyped" || s === "first name";
  }

  function findStoryCardByTitle(title) {
    if (!Array.isArray(globalThis.storyCards)) return null;
    for (let i = 0; i < storyCards.length; i++) {
      const card = storyCards[i];
      if (card && card.title === title) return card;
    }
    return null;
  }

  // Stable identity lookup by the FIRST trigger token (not the full trigger list),
  // so adding/removing a target trigger never changes a card's identity. Titles can
  // change (seedling on/off, Chaos->Life); the token stays constant.
  function findStoryCardByKeys(keys) {
    if (!Array.isArray(globalThis.storyCards)) return null;
    const token = keyIdToken(keys);
    for (let i = 0; i < storyCards.length; i++) {
      const card = storyCards[i];
      if (card && keyIdToken(card.keys) === token) return card;
    }
    return null;
  }

  // Remove a Story Card by its identity token so an archived thread leaves the card
  // list and stays gone (syncCards will not recreate a non-active bucket).
  function removeStoryCardByKeys(keys) {
    if (!keys || !Array.isArray(globalThis.storyCards)) return;
    const token = keyIdToken(keys);
    for (let i = storyCards.length - 1; i >= 0; i--) {
      if (storyCards[i] && keyIdToken(storyCards[i].keys) === token) {
        try {
          if (typeof removeStoryCard === "function") removeStoryCard(i);
          else storyCards.splice(i, 1);
        } catch (e) {
          try { storyCards.splice(i, 1); } catch (e2) {}
        }
      }
    }
  }

  function createOrPatchStoryCard(title, type, keys, entry, description) {
    if (isUnnamedish(title) || isUnnamedish(type) || !keys || !entry) return null;
    let card = findStoryCardByKeys(keys) || findStoryCardByTitle(title);
    const beforeLength = Array.isArray(globalThis.storyCards) ? storyCards.length : -1;

    if (!card && typeof addStoryCard === "function") {
      try {
        const created = addStoryCard(keys, entry, type);
        if (created && typeof created === "object") card = created;
        else if (typeof created === "number" && Array.isArray(globalThis.storyCards) && storyCards[created]) card = storyCards[created];
      } catch (e) {
        card = null;
      }
    }

    if (!card && Array.isArray(globalThis.storyCards) && storyCards.length > beforeLength) {
      card = storyCards[storyCards.length - 1];
    }

    if (!card && Array.isArray(globalThis.storyCards)) {
      card = {};
      storyCards.push(card);
    }

    if (!card || typeof card !== "object") return null;
    card.title = title;
    card.type = type;
    card.keys = keys;
    card.entry = entry;
    card.description = description || "";
    return card;
  }

  function syncCards() {
    const cg = ensureState();
    const owners = Object.keys(cg.cards || {});
    for (let i = 0; i < owners.length; i++) {
      const owner = owners[i];
      const bucket = cg.cards[owner];
      // Only live threads sync. Dormant/resolved buckets are archived in state
      // (HISTORY preserved) and must NOT be recreated as Story Cards.
      if (!cardHasContent(bucket) || !lifeStatusIsActive(bucket.status)) continue;
      // World events render as an event, not as an owner/target pressure card.
      if (isEventBucket(bucket)) { syncWorldEventCard(bucket); continue; }
      const entry = renderCardEntry(bucket);
      if (!entry) continue;
      createOrPatchStoryCard(
        cardTitle(owner, bucket.status),
        CFG.LIFE_CARD_TYPE,
        storyCardTriggers(bucket),
        entry,
        renderCardDescription(bucket)
      );
    }
  }

  // Read a labeled field's value from a rendered Story Card entry. Handles the
  // compact inline form ("LABEL: value") and the older "LABEL:" + next-line form.
  function extractEntryField(entry, label) {
    const lines = String(entry || "").replace(/\r/g, "").split("\n");
    const want = label.toUpperCase() + ":";
    for (let i = 0; i < lines.length; i++) {
      const t = lines[i].trim();
      if (t.toUpperCase().indexOf(want) === 0) {
        const inline = t.slice(want.length).trim();
        if (inline) return inline;
        for (let j = i + 1; j < lines.length; j++) {
          if (lines[j].trim()) return lines[j].trim();
        }
      }
    }
    return "";
  }

  // Per active card: compare the INTERNAL bucket (source for injection + render)
  // against the VISIBLE Story Card entry, and flag stale (non-config) pressures.
  function cardAuditLines() {
    const cg = ensureState();
    const owners = Object.keys(cg.cards || {});
    const out = [];
    for (let i = 0; i < owners.length; i++) {
      const b = cg.cards[owners[i]];
      if (!cardHasContent(b) || !lifeStatusIsActive(b.status)) continue;
      if (isEventBucket(b)) continue; // world events have no owner/target/pressure to audit
      const bP = cleanText(b.pressure), bT = cleanText(b.target);
      const card = findStoryCardByKeys(storyCardIdToken(owners[i]));
      const cP = card ? extractEntryField(card.entry, "PRESSURE") : "";
      const cT = card ? extractEntryField(card.entry, "TARGET") : "";
      const inConfig = LC.pressures.indexOf(bP.toLowerCase()) !== -1 || isRulePressure(bP);
      const match = (cP.toLowerCase() === bP.toLowerCase() && cT.toLowerCase() === bT.toLowerCase());
      out.push("AUDIT " + owners[i] +
        " | bucket(injected): T=" + (bT || "-") + " P=" + (bP || "-") +
        " | card(visible): T=" + (cT || "-") + " P=" + (cP || "-") +
        " | source=internalCache | inConfig=" + (inConfig ? "yes" : "NO-stale") +
        " | match=" + (match ? "yes" : "NO"));
    }
    return out.length ? out.join("\n") : "AUDIT: no active cards";
  }

  function updateDebugCard() {
    if (!CFG.DEBUG) { removeStoryCardByKeys(CFG.DEBUG_CARD_KEY); return; }
    const cg = ensureState();
    const owners = Object.keys(cg.cards || {});
    let live = 0, activeLife = 0, dormant = 0, resolved = 0;
    for (let i = 0; i < owners.length; i++) {
      const b = cg.cards[owners[i]];
      const st = cleanText(b && b.status).toLowerCase();
      if (st === "dormant") { dormant++; continue; }
      if (st === "resolved") { resolved++; continue; }
      if (!cardHasContent(b)) continue;
      live++;
      if (lifeStatusIsActive(b.status)) activeLife++;
    }
    const pending = cg.pendingSeed
      ? cg.pendingSeed.actor + " -> " + cg.pendingSeed.target +
        " (" + cg.pendingSeed.pressure + ", age " +
        (cg.turn - (cg.pendingSeed.turnCreated || 0)) + ")"
      : "none";
    const activityLabel = LC.activityOff ? "Off" : ("every ~" + LC.activityTurns + " turns");
    const entry = [
      "LIFE CARDS DEBUG",
      "version: " + CFG.VERSION,
      "turn: " + (cg.turn || 0),
      "config protagonist: " + (LC.protagonist || "(none)"),
      "config cast: " + LC.characters.length + " total (" + (LC.rosterActorCount || 0) + " roster from " + LC.rosterSource + " + " + (LC.relActorCount || 0) + " relationship-only) | pressures: " + LC.pressures.length,
      "config pacing: LIFE_CARD_INTERVAL " + activityLabel + " / cooldown " + LC.targetCooldown + " cards / cap " + LC.maxActive,
      (LC.legacyActivityUsed
        ? "NOTE: SOCIAL_ACTIVITY is deprecated and was auto-converted to LIFE_CARD_INTERVAL=" + (LC.activityOff ? 0 : LC.activityTurns) + ". Add a LIFE_CARD_INTERVAL line to silence this."
        : "config source: LIFE_CARD_INTERVAL"),
      "relationshipsCard: " + (LC.relationshipsCardFound ? "FOUND (\"" + CFG.REL_CARD_TITLE + "\")" : "not found -- optional; random targeting in use"),
      (LC.mainConfigHadRelationships
        ? "NOTE: a RELATIONSHIPS section in LIVING CHARACTERS CONFIG is now IGNORED -- move it to the \"" + CFG.REL_CARD_TITLE + "\" card."
        : "relationships source: dedicated card"),
      "relationships: " + (LC.relationshipCount || 0) + " rules / " + Object.keys(LC.relationships || {}).length + " ruled owners",
      "cast: " + (LC.rosterActorCount || 0) + " roster + " + (LC.relActorCount || 0) + " relationship-only = " + LC.characters.length + " total (valid targets + scene-detectable)",
      "eligibleOwners: " + ownerCandidates().length + " (roster NPCs + ruled owners; relationship-only targets are targets-only, never random owners)",
      "relationshipRules: " + relationshipRuleLines(),
      "relationshipNotes (parse errors): " + ((LC.relationshipNotes && LC.relationshipNotes.length) ? LC.relationshipNotes.join(" | ") : "(none)"),
      "lastRoll: " + (cg.lastRoll || "n/a"),
      "seedSource: " + (cg.seedSource || "n/a"),
      "seedCandidates: " + ((cg.seedCandidates && cg.seedCandidates.length) ? cg.seedCandidates.join(", ") : "(none)"),
      "selectedSeed: " + (cg.selectedSeed || "(none)"),
      "seedReason: " + (cg.seedReason || "n/a"),
      "seedSteering: " + (cg.seedTargeting || "n/a") + " | pressureSource: " + (cg.seedPressureSource || "n/a"),
      "roundRobin: next-phase=" + (cg.seedPhase || "relationship") + " | last=" + (cg.seedRoundRobin || "n/a"),
      "pendingSeed: " + pending,
      "lastSeedAttemptTurn: " + (cg.lastSeedAttemptTurn || 0) + " | totalLifeCards: " + (cg.seedCount || 0),
      "lastThreadReminderTurn: " + (cg.lastThreadReminderTurn || 0),
      "activeLifeCards/cap: " + activeLife + "/" + LC.maxActive,
      "sceneMode: " + LC.sceneRelevanceMode + " (" + (LC.sceneRelevanceMode === "off" ? "scene IGNORED" : LC.sceneRelevanceMode === "strict" ? "scene ENFORCED" : "scene PREFERRED") + ")",
      "protagonistInvolvement: " + LC.protagonistInvolvement + " (target-side bias; protagonist never owns a card)" + (LC.protagonist ? "" : " [no protagonist set -> normal]"),
      "sceneRelevance applied -> seed: " + (cg.seedSource === "skipped" ? "enforced(skipped)" : cg.seedSource === "sceneActors" ? "enforced" : "ignored") + " | inject: " + (LC.sceneRelevanceMode === "off" ? "ignored" : LC.sceneRelevanceMode === "strict" ? "enforced" : "preferred"),
      "injected " + (cg.lastActiveSelected || 0) + " of " + activeLife + " active | sceneSource: " + (cg.sceneSource || "?"),
      "sceneActors" + (LC.protagonistAlwaysPresent && LC.protagonist ? " (protagonist always-on)" : "") + ": " + ((cg.sceneActors && cg.sceneActors.length) ? cg.sceneActors.join(", ") : "(none)"),
      "candidateCards: " + ((cg.lastCandidateTrace && cg.lastCandidateTrace.length) ? cg.lastCandidateTrace.join(" | ") : "(none)"),
      "selectedCards / injectedCards: " + ((cg.lastSelectedTrace && cg.lastSelectedTrace.length) ? cg.lastSelectedTrace.join(" | ") : "(none)"),
      "freshThreads (marked URGENT): " + ((cg.lastFreshThreads && cg.lastFreshThreads.length) ? cg.lastFreshThreads.join(", ") : "(none)"),
      "worldEvents configured: " + (LC.worldEvents ? LC.worldEvents.length : 0) +
        " | live: " + (function () {
          const b = ensureState().cards[CFG.EVENT_BUCKET_KEY];
          return (b && cardHasContent(b) && lifeStatusIsActive(b.status))
            ? ("\"" + cleanText(b.worldEvent) + "\" (since turn " + (b.createdTurn || 0) + ", normal life-card lifespan)")
            : "(none)";
        })(),
      "seedCycle: [" + CFG.SEED_CYCLE.join(", ") + "] | next=" + CFG.SEED_CYCLE[((cg.seedCycleIndex || 0) % CFG.SEED_CYCLE.length)],
      cardAuditLines(),
      "archivedLifeCards (dormant/resolved): " + dormant + "/" + resolved,
      "cards (live/total): " + live + "/" + owners.length,
      "lastOccurrence: " + (cg.recentMemory && cg.recentMemory.length ? cg.recentMemory[cg.recentMemory.length - 1] : "none"),
      "lastEmptyFallback: " + (cg.lastFallback || "(none)")
    ].join("\n");
    createOrPatchStoryCard(
      CFG.DEBUG_CARD_TITLE,
      CFG.DEBUG_CARD_TYPE,
      CFG.DEBUG_CARD_KEY,
      entry,
      "Diagnostic only. Set DEBUG = false to disable."
    );
  }

  // Names eligible to OWN a Life Card:
  //   - roster NPCs (the classic random owners), and
  //   - any name with its OWN relationship rule (a ruled owner; may be the protagonist).
  // Relationship-only TARGETS are deliberately excluded: being named as someone's target
  // makes a character a valid target and scene-detectable, but NEVER a random owner. A
  // target-only name only becomes an owner if it also appears in the roster or gains its
  // own rule.
  function ownerCandidates() {
    const p = playerName();
    const set = {};
    const roster = LC.rosterCharacters || [];
    for (let i = 0; i < roster.length; i++) {
      if (roster[i] && roster[i] !== p) set[roster[i]] = true;
    }
    // Ruled owners (the keys of the relationships map). The protagonist is allowed here
    // ONLY via an explicit rule; random selection still never makes them an owner.
    const owners = Object.keys(LC.relationships || {});
    for (let i = 0; i < owners.length; i++) {
      if (owners[i]) set[owners[i]] = true;
    }
    return Object.keys(set);
  }

  // Eligible actors = ownerCandidates, minus characters who already hold an active Life
  // thread (one per character -- so a ruled owner with a live card is skipped, and the
  // engine does NOT compensate by promoting that owner's target to a random owner), minus
  // characters still inside their TARGET_COOLDOWN (counted in Life Cards, not turns). When
  // the owner's card archives, they return here and may seed again from their rule targets.
  function actorPool() {
    const cg = ensureState();
    return unique(ownerCandidates()).filter(function(name) {
      if (!name) return false;
      const b = cg.cards[name];
      if (b && cardHasContent(b) && lifeStatusIsActive(b.status)) return false;
      const lastIdx = cg.actorSeedIndex[name];
      if (lastIdx != null && (cg.seedCount - lastIdx) < LC.targetCooldown) return false;
      return true;
    });
  }

  function targetPool(actor) {
    const p = playerName();
    return unique([p].concat(npcRoster())).filter(function(name) {
      return name && name !== actor;
    });
  }

  // ---- RELATIONSHIPS: directed steering rules (seed-time only) -------------
  // Optional "Owner > Target [xN] [= pressure, pressure]" lines from the dedicated
  // LIVING CHARACTERS RELATIONSHIPS card. Rules narrow WHAT seeds (target set and
  // pressure pool), never WHEN: pacing, cooldowns, slot caps, dormancy, and
  // existing/pending cards are untouched. An owner with at least one rule ONLY
  // targets listed characters (fail closed when none is currently legal -- never a
  // random fallback); owners without rules keep the original random behavior exactly.

  // Parse the RELATIONSHIPS card lines. STRUCTURE is parsed FIRST (">", "=", ",",
  // trailing " xN") and names are cleaned AFTER -- cleanName strips those symbols,
  // so it must never see the raw line. Invalid rules are skipped and reported;
  // for duplicate directed pairs the LAST valid rule wins.
  //
  // NAME RESOLUTION is liberal (no roster membership required):
  //   - "You" / "Protagonist" (and the protagonist's own name) -> PROTAGONIST_NAME.
  //   - A name matching the roster (case-insensitive) canonicalizes to the roster
  //     spelling so "jessica" and "Jessica" are the same character.
  //   - ANY OTHER non-empty name is accepted as a new relationship-only character;
  //     the caller folds these into the runtime cast, so users never duplicate names.
  // The protagonist MAY own a rule. Only self-targeting is rejected.
  function parseRelationshipRules(rawLines, characters, protagonist) {
    const rules = {};  // owner -> [{ target, pressures: array|null, weight }]
    const notes = [];
    let count = 0;

    // Canonical-name resolver. Roster spellings are authoritative; the first spelling
    // seen wins for any relationship-only name. Returns "" only for empty input or an
    // unresolvable You/Protagonist (no PROTAGONIST_NAME set).
    const canon = {};
    for (let i = 0; i < (characters || []).length; i++) {
      const c = cleanName(characters[i]);
      if (c) canon[c.toLowerCase()] = c;
    }
    const pName = cleanName(protagonist);
    const pKey = pName.toLowerCase();
    function resolve(label) {
      const c = cleanName(label);
      if (!c) return "";
      const low = c.toLowerCase();
      if (low === "you" || low === "protagonist" || (pKey && low === pKey)) return pName; // "" if no protagonist
      if (canon[low]) return canon[low];
      canon[low] = c;
      return c;
    }

    // SECTIONING: rules are only read AFTER a "Relationships:" (or "RELATIONSHIPS")
    // header line. This lets the card carry a title, an "Example:" label, and example
    // lines ABOVE the header without them ever being parsed as live rules. If no header
    // line exists, every line is considered (backward compatible with header-less cards).
    let lines = rawLines || [];
    for (let h = 0; h < lines.length; h++) {
      if (/^\s*relationships\s*:?\s*$/i.test(String(lines[h] || ""))) { lines = lines.slice(h + 1); break; }
    }
    for (let i = 0; i < lines.length; i++) {
      const raw = String(lines[i] || "").trim();
      if (!raw || raw.charAt(0) === "(") continue;
      const gt = raw.indexOf(">");
      if (gt === -1) {
        // A label line ending in ":" or a stray "relationships" header is silent; other
        // non-rule lines are noted so typos are visible on the debug card.
        if (!/:$/.test(raw) && !/^relationships$/i.test(raw)) notes.push("ignored (no '>'): " + raw.slice(0, 60));
        continue;
      }
      const ownerRaw = raw.slice(0, gt);
      let targetRaw = raw.slice(gt + 1);
      let listRaw = null;
      const eq = targetRaw.indexOf("=");
      if (eq !== -1) { listRaw = targetRaw.slice(eq + 1); targetRaw = targetRaw.slice(0, eq); }
      // Optional trailing weight on the target segment: "Target x3".
      let weight = 1;
      const wm = /\sx(\d+)\s*$/i.exec(targetRaw);
      if (wm) { weight = Math.max(1, toIntOr(wm[1], 1)); targetRaw = targetRaw.slice(0, wm.index); }

      const owner = resolve(ownerRaw);
      if (!owner) { notes.push("ignored (unresolved owner \"" + (cleanName(ownerRaw) || "?") + "\" -- set PROTAGONIST_NAME to use You/Protagonist)"); continue; }
      const target = resolve(targetRaw);
      if (!target) { notes.push("ignored (unresolved target \"" + (cleanName(targetRaw) || "?") + "\" for " + owner + ")"); continue; }
      if (target === owner) { notes.push("ignored (self-target): " + owner); continue; }

      // Pressure list: cleaned the same way as the global PRESSURES pool
      // (cleanName + lowercase) so comparisons and audits line up. Pressures NOT
      // in the global pool are allowed on purpose (pair-specific pressures).
      let pressures = null;
      if (listRaw != null) {
        const parts = String(listRaw).split(",");
        const list = [];
        for (let j = 0; j < parts.length; j++) {
          const pr = cleanName(parts[j]).toLowerCase();
          if (pr && list.indexOf(pr) === -1) list.push(pr);
        }
        if (list.length) pressures = list;
        else notes.push("empty pressure list (global pool used): " + owner + " > " + target);
      }

      if (!rules[owner]) rules[owner] = [];
      let replaced = false;
      for (let j = 0; j < rules[owner].length; j++) {
        if (rules[owner][j].target === target) {
          rules[owner][j] = { target: target, pressures: pressures, weight: weight };
          notes.push("duplicate pair (last rule used): " + owner + " > " + target);
          replaced = true;
          break;
        }
      }
      if (!replaced) {
        rules[owner].push({ target: target, pressures: pressures, weight: weight });
        count++;
      }
    }
    while (notes.length > 12) notes.pop();
    return { rules: rules, notes: notes, count: count };
  }

  function ownerRules(owner) {
    const list = (LC.relationships || {})[owner];
    return (list && list.length) ? list : null;
  }

  function ruleFor(owner, target) {
    const list = ownerRules(owner);
    if (!list) return null;
    for (let i = 0; i < list.length; i++) {
      if (list[i].target === target) return list[i];
    }
    return null;
  }

  // True when `owner` may target `target`: unruled owners may target anyone,
  // ruled owners only characters in their configured set.
  function canTarget(owner, target) {
    return !ownerRules(owner) || !!ruleFor(owner, target);
  }

  // True when `owner` could seed SOME target right now: unruled, or at least one
  // configured target is currently in their legal target pool.
  function hasLegalRuleTarget(owner) {
    const list = ownerRules(owner);
    if (!list) return true;
    const pool = targetPool(owner);
    for (let i = 0; i < list.length; i++) {
      if (pool.indexOf(list[i].target) !== -1) return true;
    }
    return false;
  }

  // True when a pressure appears in ANY relationship rule's list, so the debug
  // audit does not flag legitimate pair-specific pressures as stale.
  function isRulePressure(p) {
    p = cleanText(p).toLowerCase();
    if (!p) return false;
    const owners = Object.keys(LC.relationships || {});
    for (let i = 0; i < owners.length; i++) {
      const list = LC.relationships[owners[i]] || [];
      for (let j = 0; j < list.length; j++) {
        if (list[j].pressures && list[j].pressures.indexOf(p) !== -1) return true;
      }
    }
    return false;
  }

  // Compact resolved-rule summary for the debug card, e.g.
  // "Jessica > [Sam x3 (=jealousy, attraction), Tristan]".
  function relationshipRuleLines() {
    const owners = Object.keys(LC.relationships || {});
    const parts = [];
    for (let i = 0; i < owners.length; i++) {
      const list = LC.relationships[owners[i]] || [];
      const t = [];
      for (let j = 0; j < list.length; j++) {
        t.push(list[j].target +
          (list[j].weight > 1 ? " x" + list[j].weight : "") +
          (list[j].pressures ? " (=" + list[j].pressures.join(", ") + ")" : ""));
      }
      parts.push(owners[i] + " > [" + t.join(", ") + "]");
    }
    return parts.length ? parts.join(" ; ") : "(none)";
  }

  // Choose a TARGET for a given (already-selected NPC) owner, applying the
  // PROTAGONIST_INVOLVEMENT bias. This only ever influences who is TARGETED; owner
  // selection (actorPool) and the never-an-owner rule are untouched. With no
  // protagonist, LC.protagonistInvolvement is forced to "normal" upstream, so this
  // behaves exactly like the original choose(targetPool(actor)).
  function chooseTarget(actor) {
    const p = playerName();
    const mode = LC.protagonistInvolvement || "normal";
    const pool = targetPool(actor);
    if (!pool.length) return "";
    // RELATIONSHIPS steering: a ruled owner's target ALWAYS comes from their rules
    // (weighted via weightedChoice), overriding PROTAGONIST_INVOLVEMENT for that
    // owner. If no configured target is currently legal, fail closed with "" --
    // the caller skips this seed; a ruled owner NEVER falls back to a random target.
    const rules = ownerRules(actor);
    if (rules) {
      const pairs = [];
      for (let i = 0; i < rules.length; i++) {
        if (pool.indexOf(rules[i].target) !== -1) pairs.push([rules[i].target, rules[i].weight || 1]);
      }
      return pairs.length ? weightedChoice(pairs) : "";
    }
    const protagInPool = !!p && pool.indexOf(p) !== -1;

    if (mode === "off") {
      // Exclude the protagonist absolutely. If no NPC target remains, choose([])
      // returns "" and the caller safely skips the card (never falls back to the
      // protagonist) -- "off" means the protagonist is never targeted.
      const npcOnly = pool.filter(function(n) { return n !== p; });
      return choose(npcOnly);
    }
    if (mode === "always" && protagInPool) {
      // Use the protagonist as target when legal. (Owner stays an NPC; cooldown and
      // scene relevance are still enforced by the surrounding selection logic.)
      return p;
    }
    if (mode === "high" && protagInPool) {
      // Protagonist gets ~3x the weight of a single NPC; NPC-to-NPC still happens.
      const pairs = pool.map(function(n) { return [n, n === p ? 3 : 1]; });
      return weightedChoice(pairs);
    }
    // normal (and high/always with no eligible protagonist target): current behavior.
    return choose(pool);
  }

  function cardReason(actor, target, category) {
    const cg = ensureState();
    const actorCard = cg.cards[actor];
    if (cardHasContent(actorCard)) {
      const detail = cleanText(actorCard.event) ||
        (cleanText(actorCard.pressure) + " toward " + (cleanText(actorCard.target) || target));
      return actor + " has unresolved " + category + " pressure connected to " + target + ": " + detail;
    }
    return actor + " has enough social momentum around " + target + " for a " + category + " move.";
  }

  function seedSignature(seed) {
    return [seed.actor, seed.target, seed.category].join("|").toLowerCase();
  }

  // Seed-dedup bookkeeping. Entries are { sig, turn } and EXPIRE after
  // SEED_DEDUP_TTL_TURNS turns (see the CFG comment for why turn-based).
  function pruneRecentSeeds(cg) {
    const keep = [];
    for (let i = 0; i < cg.recentSeeds.length; i++) {
      const e = cg.recentSeeds[i];
      if (e && typeof e.sig === "string" && (cg.turn - (e.turn || 0)) < CFG.SEED_DEDUP_TTL_TURNS) keep.push(e);
    }
    cg.recentSeeds = keep;
  }

  function isRecentSeed(sig) {
    const cg = ensureState();
    pruneRecentSeeds(cg);
    for (let i = 0; i < cg.recentSeeds.length; i++) {
      if (cg.recentSeeds[i].sig === sig) return true;
    }
    return false;
  }

  function rememberSeed(seed) {
    const cg = ensureState();
    pruneRecentSeeds(cg);
    const sig = seedSignature(seed);
    let known = false;
    for (let i = 0; i < cg.recentSeeds.length; i++) {
      if (cg.recentSeeds[i].sig === sig) { cg.recentSeeds[i].turn = cg.turn; known = true; break; }
    }
    if (!known) cg.recentSeeds.push({ sig: sig, turn: cg.turn });
    while (cg.recentSeeds.length > 20) cg.recentSeeds.shift();
    // Count this Life Card and stamp the actor's card index for TARGET_COOLDOWN.
    cg.seedCount = (cg.seedCount || 0) + 1;
    cg.actorSeedIndex[seed.actor] = cg.seedCount;
  }

  function momentumForIntensity(intensity) {
    if (intensity === "major") return "high";
    if (intensity === "medium") return "medium";
    return "low";
  }

  // True for a WORLD EVENT bucket: an event, not a person. Every character path checks
  // this (never an actor, target, or scene presence). Keyed off `kind`, NOT the display
  // name, so a character called "The World" could never be mistaken for the event card.
  function isEventBucket(b) {
    return !!(b && b.kind === "event");
  }

  // A card is meaningful once it carries system pressure or a narrator event. A world
  // event carries its event text instead.
  function cardHasContent(b) {
    if (isEventBucket(b)) return !!cleanText(b.worldEvent);
    return !!(b && (cleanText(b.pressure) || cleanText(b.event)));
  }

  function worldEventSignature(text) {
    return CFG.EVENT_BUCKET_KEY + "||" + cleanText(text).toLowerCase();
  }

  // Seed a WORLD EVENT: a Life Card owned by the world rather than a person. It takes a
  // normal card slot (so MAX_ACTIVE_CARDS governs whether a character card can coexist)
  // and rides the same injection/lifecycle machinery. Returns a seed-shaped object, or
  // null when no events are configured -- the caller then falls through to characters.
  function seedWorldEvent() {
    const cg = ensureState();
    const list = LC.worldEvents || [];
    if (!list.length) return null;
    // Only ONE world event at a time. All events share a single bucket key, so seeding
    // while one is live would silently overwrite it -- return null instead and let the
    // round-robin fall through to a character seat.
    const live = cg.cards[CFG.EVENT_BUCKET_KEY];
    if (live && cardHasContent(live) && lifeStatusIsActive(live.status)) return null;
    // Prefer an event that is not still inside the dedup TTL, so the same one does not
    // repeat back to back; if they are all recent, allow a repeat rather than stall.
    const unused = list.filter(function(t) { return !isRecentSeed(worldEventSignature(t)); });
    const text = cleanText(choose(unused.length ? unused : list));
    if (!text) return null;

    cg.cards[CFG.EVENT_BUCKET_KEY] = {
      kind: "event",
      owner: CFG.EVENT_OWNER_LABEL, // display only -- never used for selection
      worldEvent: text,
      target: "", pressure: "", momentum: "", event: "",
      status: "active",
      reminderCount: 0,
      createdTurn: cg.turn,
      eventLog: []
    };

    cg.recentSeeds.push({ sig: worldEventSignature(text), turn: cg.turn });
    while (cg.recentSeeds.length > 20) cg.recentSeeds.shift();
    cg.seedCount = (cg.seedCount || 0) + 1; // an event counts as a produced card
    cg.lastSeedAttemptTurn = cg.turn;
    cg.seedSource = "worldEvent";
    cg.seedCandidates = [];
    cg.selectedSeed = "WORLD EVENT";
    cg.seedTargeting = "event";
    cg.seedPressureSource = "event list";
    cg.seedReason = "world event seeded (narrator chooses who is involved)";
    cg.lastRoll = "WORLD EVENT: " + text;
    return { id: "event-" + cg.turn, turnCreated: cg.turn, kind: "event", worldEvent: text };
  }

  // Story card for a live world event. Separate from the character-card renderer: no
  // owner/target/pressure, and the trigger list carries no person's name.
  function syncWorldEventCard(bucket) {
    createOrPatchStoryCard(
      CFG.EVENT_CARD_TITLE,
      CFG.EVENT_CARD_TYPE,
      CFG.EVENT_BUCKET_KEY + ",you",
      "WORLD EVENT: " + cleanText(bucket.worldEvent) + "\nSTATUS: " + (cleanText(bucket.status) || "active"),
      "A world event in play. The narrator decides who is pulled into it."
    );
  }

  // Materialize a pressure card the moment a Life Card seed fires. The system
  // writes only the abstract pressure (TARGET / PRESSURE / MOMENTUM); it never
  // invents the specific behavior and never overwrites a narrator-authored EVENT.
  function bootstrapSeedCard(seed) {
    if (!seed) return;
    const bucket = ensureCardBucket(seed.actor);
    if (!bucket) return;

    bucket.target = seed.target;
    bucket.pressure = seed.pressure;
    bucket.momentum = seed.momentum;
    bucket.reminderCount = 0; // fresh pressure restarts the dormancy clock
    bucket.status = "active";
    bucket.createdTurn = ensureState().turn; // for FRESH PRESSURE detection in the block
    appendEventLog(bucket, "[pressure] " + seed.pressure + " toward " + seed.target + " (" + seed.momentum + ")");
  }

  // A Life card counts as "active" only in these statuses. dormant/resolved do not.
  function lifeStatusIsActive(status) {
    const list = CFG.ACTIVE_LIFE_STATUSES || ["active", "simmering", "surfaced"];
    return list.indexOf(cleanText(status).toLowerCase()) !== -1;
  }

  function countActiveLifeCards() {
    const cg = ensureState();
    const owners = Object.keys(cg.cards || {});
    let n = 0;
    for (let i = 0; i < owners.length; i++) {
      const b = cg.cards[owners[i]];
      if (cardHasContent(b) && lifeStatusIsActive(b.status)) n++;
    }
    return n;
  }


  // Dormancy ARCHIVES a thread: keep the card and HISTORY, but clear the active
  // thread fields so the character fully releases this storyline and can be
  // reseeded later with a brand-new TARGET / PRESSURE / MOMENTUM.
  function makeDormant(bucket) {
    if (!bucket) return;
    const wasEvent = isEventBucket(bucket);
    bucket.target = "";
    bucket.pressure = "";
    bucket.momentum = "";
    bucket.event = "";
    if (wasEvent) bucket.worldEvent = ""; // release the event text too
    bucket.status = "dormant";
    bucket.reminderCount = 0;
    // eventLog (HISTORY) is intentionally left untouched.

    // Drop the Story Card; the bucket + HISTORY remain in state as the archive. A world
    // event's card is keyed by EVENT_BUCKET_KEY, not by an owner token, so it needs its
    // own lookup or the card would linger after the thread archived.
    removeStoryCardByKeys(wasEvent ? CFG.EVENT_BUCKET_KEY : storyCardIdToken(bucket.owner));
  }

  // Select an (owner -> target) pair from a GIVEN owner pool, honoring scene relevance.
  // PURE: returns { actor, target, seedSource } or null and sets no state. The round-robin
  // caller runs it once per pool (the preferred type first, then the other as fallback).
  // This is the same scene/strict/hybrid/off selection the engine has always used, just
  // parameterized by which owner pool it may draw from.
  function pickSeedPair(pool, mode, scene, p) {
    if (!pool || !pool.length) return null;
    if (mode === "strict" || mode === "hybrid") {
      const sceneNPCs = unique(npcRoster()).filter(function(n) { return n !== p && scene.indexOf(n) !== -1; });
      if (sceneNPCs.length) {
        const anchor = choose(sceneNPCs);                        // the present side
        // A ruled anchor may only OWN when a configured target is currently legal
        // (fail closed); owners considered for the target-present branch must be ALLOWED
        // to target the anchor (unruled, or anchor is in their configured target set).
        const anchorCanOwn = pool.indexOf(anchor) !== -1 && hasLegalRuleTarget(anchor);
        const otherOwners = pool.filter(function(n) { return n !== anchor && canTarget(n, anchor); });
        const anchorAsOwner = anchorCanOwn && (LC.protagonistInvolvement === "always" || otherOwners.length === 0 || Math.random() < 0.5);
        let actor = null, target = null;
        if (anchorAsOwner) { actor = anchor; target = chooseTarget(actor); }
        else if (otherOwners.length) { target = anchor; actor = choose(otherOwners); }
        else if (anchorCanOwn) { actor = anchor; target = chooseTarget(actor); }
        if (actor && target) return { actor: actor, target: target, seedSource: "sceneActors" };
      }
      if (mode === "strict") return null; // strict: this pool has no scene-eligible pair
      // hybrid: fall through to off-scene selection from this same pool
    }
    const seedable = pool.filter(hasLegalRuleTarget);
    if (!seedable.length) return null;
    const actor = choose(seedable);
    const target = chooseTarget(actor);
    if (!target) return null;
    return { actor: actor, target: target, seedSource: "fullRoster" };
  }

  function maybeCreateSeed(text) {
    const cg = ensureState();
    if (!CFG.AUTONOMY_ENABLED || LC.activityOff) { cg.lastRoll = "disabled (Social Activity Off)"; return null; }

    // First-card turn gate: no Life Card until the story has had time to form.
    if (cg.turn < CFG.MIN_TURN_BEFORE_FIRST_SEED) {
      cg.lastRoll = "blocked: before first card turn";
      return null;
    }

    if (cg.pendingSeed && cg.turn - (cg.pendingSeed.turnCreated || 0) > CFG.AUTONOMY_MAX_PENDING_AGE) {
      cg.pendingSeed = null;
    }
    if (cg.pendingSeed) {
      cg.lastRoll = "pending reused: " + cg.pendingSeed.actor + " -> " + cg.pendingSeed.target;
      return cg.pendingSeed;
    }
    // HARD SLOT CAP: while the active slots are full, do NOT roll, seed, or replace.
    // A new card is only considered after an active card finishes its full reminder
    // cycle and archives (dormant), which frees a slot.
    if (countActiveLifeCards() >= LC.maxActive) {
      cg.lastRoll = "blocked: slot cap full (" + countActiveLifeCards() + "/" + LC.maxActive + ") -- waiting for a card to archive";
      return null;
    }
    // Pacing: the FIRST card ever (seedCount 0) ignores the interval and fires as
    // soon as MIN_TURN allows. After that, wait LIFE_CARD_INTERVAL turns since the
    // last SUCCESSFUL card, then retry every turn until one is created (a failed
    // attempt does not waste the interval). lastSeedAttemptTurn is stamped on success.
    if ((cg.seedCount || 0) > 0 && cg.turn - (cg.lastSeedAttemptTurn || 0) < LC.activityTurns) {
      cg.lastRoll = "waiting (next attempt ~turn " + ((cg.lastSeedAttemptTurn || 0) + LC.activityTurns) + ")";
      return null;
    }

    // Scene-aware OWNER selection. The owner holds the pressure, so in strict mode it
    // must be an NPC currently in scene; hybrid prefers in-scene owners but falls back
    // to the full roster; off uses the full roster. (Protagonist is never an owner, so
    // the protagonist being always-present does not make every NPC eligible.)
    // Scene eligibility (strict/hybrid): a card is eligible if AT LEAST ONE side --
    // owner OR target -- is a NON-protagonist NPC in scene. The other side may be
    // off-scene (owner expresses pressure toward an absent target; or others react to
    // a present target whose owner is absent). The always-present protagonist never
    // makes a card eligible by itself:
    //   NPC -> NPC:          owner OR target in scene
    //   NPC -> Protagonist:  owner must be in scene
    //   Protagonist -> NPC:  N/A (protagonist is never an owner)
    const mode = LC.sceneRelevanceMode || "off";
    const scene = cg.sceneActors || [];
    const p = playerName();
    const poolAll = actorPool();

    // ROUND-ROBIN with FULL FALLBACK. Each produced card advances one seat in
    // CFG.SEED_CYCLE (relationship -> event -> random -> repeat), so every kind gets a
    // fair turn. CRITICALLY: a seat that cannot produce this turn is SKIPPED and the next
    // seat is tried, wrapping all the way around the cycle.
    //
    // This is the fix for the queue stalling: the index used to advance only on SUCCESS,
    // so a blocked character seat (everyone already holding a card, on cooldown, or the
    // pair being a recent duplicate) froze the rotation on that seat and events simply
    // stopped firing. Worse, an EVENTS-ONLY setup (no characters configured) could never
    // advance off seat 0 and never reached the event seat at all. Now every seat acts as
    // the fallback for every other seat, so whatever CAN fire, does.
    const cycle = CFG.SEED_CYCLE;
    let cycleIdx = (typeof cg.seedCycleIndex === "number" ? cg.seedCycleIndex : 0);
    cycleIdx = ((cycleIdx % cycle.length) + cycle.length) % cycle.length;
    const ruledPool = poolAll.filter(function(n) { return !!ownerRules(n); });
    const randomPool = poolAll.filter(function(n) { return !ownerRules(n); });
    const tried = [];
    const dupSeats = []; // seats whose pick was blocked by the dedup TTL (kept for debug)

    for (let step = 0; step < cycle.length; step++) {
      const idx = (cycleIdx + step) % cycle.length;
      const seat = cycle[idx];
      cg.seedPhase = seat; // debug display
      const seatLabel = "seat=" + seat +
        (step === 0 ? " (preferred)" : " (fallback after " + tried.join(",") + ")");
      tried.push(seat);

      // ---- EVENT seat ------------------------------------------------------
      if (seat === "event") {
        const evSeed = seedWorldEvent();
        if (evSeed) {
          cg.seedCycleIndex = (idx + 1) % cycle.length;
          cg.seedRoundRobin = seatLabel;
          return evSeed;
        }
        continue; // no events configured, or one already live -> try the next seat
      }

      // ---- CHARACTER seats (relationship / random) -------------------------
      const pool = (seat === "relationship") ? ruledPool : randomPool;
      const pick = pickSeedPair(pool, mode, scene, p);
      if (!pick) continue;

      const actor = pick.actor, target = pick.target, seedSource = pick.seedSource;
      // Pressure resolution: exact Owner > Target rule list, else the global pool.
      const rule = ruleFor(actor, target);
      const rulePressures = (rule && rule.pressures && rule.pressures.length) ? rule.pressures : null;
      const pressure = choose(rulePressures || LC.pressures) || "tension";
      const intensity = weightedChoice([["small", 65], ["medium", 28], ["major", 7]]);

      const seed = {
        id: "seed-" + cg.turn + "-" + randomInt(100000),
        turnCreated: cg.turn,
        actor: actor,
        target: target,
        category: pressure,
        intensity: intensity,
        pressure: pressure,
        momentum: momentumForIntensity(intensity),
        reason: cardReason(actor, target, pressure)
      };
      // A duplicate signature must not stall the queue either -- try the next seat.
      if (isRecentSeed(seedSignature(seed))) {
        dupSeats.push(seat + ":" + actor + "->" + target);
        cg.lastRoll = "attempt: duplicate, skipped (signature expires after " + CFG.SEED_DEDUP_TTL_TURNS + " turns)";
        continue;
      }

      const ownerIn = actor !== p && scene.indexOf(actor) !== -1;
      const tgtIn = target !== p && scene.indexOf(target) !== -1;
      cg.seedSource = seedSource;
      cg.seedCandidates = poolAll.slice(0, 12);
      cg.selectedSeed = actor + " -> " + target;
      cg.seedRoundRobin = seatLabel;
      cg.seedReason = (seedSource === "sceneActors")
        ? (ownerIn && tgtIn ? "owner & target in scene" : ownerIn ? "owner in scene" : "target in scene")
        : "off-scene seed allowed (mode=" + mode + ")";
      cg.seedTargeting = ownerRules(actor) ? "rule" : "random";
      cg.seedPressureSource = rulePressures ? "rule" : "global";
      cg.seedRule = rule
        ? "rule: " + actor + " > " + target + " | pressures: " + (rulePressures ? rulePressures.join(", ") : "global pool")
        : "";
      if (cg.seedRule) cg.seedReason += " | " + cg.seedRule;

      cg.pendingSeed = seed;
      cg.lastSeedAttemptTurn = cg.turn;            // start the next interval only on SUCCESS
      cg.seedCycleIndex = (idx + 1) % cycle.length; // advance past the seat that produced
      cg.lastRoll = "LIFE CARD " + actor + " -> " + target + " [" + pressure + "/" + intensity + "]";
      rememberSeed(seed);
      bootstrapSeedCard(seed);
      return seed;
    }

    // Nothing in the entire cycle could produce a card this turn.
    cg.seedSource = "skipped";
    cg.seedCandidates = poolAll.slice(0, 12);
    cg.selectedSeed = "(none)";
    cg.seedRoundRobin = "no seat produced (tried " + tried.join(",") + ")";
    cg.seedReason = poolAll.length
      ? (mode === "strict"
          ? "strict: no scene-eligible pair, and no event available"
          : "no eligible owner/target pair, and no event available")
      : ((LC.worldEvents && LC.worldEvents.length)
          ? "no eligible owners; an event is already live or none available"
          : "no eligible owners");
    // Keep the duplicate reason visible -- otherwise a dedup block would be masked by the
    // generic "no seat produced" message and the debug trail would lose why nothing fired.
    cg.lastRoll = dupSeats.length
      ? ("attempt: duplicate, skipped (" + dupSeats.join("; ") + "; signature expires after " +
         CFG.SEED_DEDUP_TTL_TURNS + " turns)")
      : ("attempt: no card from any seat (" + tried.join(",") + ")");
    return null;
  }

  function buildContextDirective(text) {
    const seed = maybeCreateSeed(text);
    if (!seed) return "";
    // A world event names nobody -- the narrator picks who gets pulled in.
    if (seed.kind === "event") {
      return "\n\n<LC_PRIVATE>\n" +
        "### Stop the story. Use this now.\n" +
        "A world event is happening RIGHT NOW: " + seed.worldEvent + "\n" +
        "Bring it into the scene immediately and make it the centre of what happens next. YOU decide which characters are pulled into it. Do not mention this note.\n" +
        "</LC_PRIVATE>";
    }
    // Compact: a new pressure formed. Let it color behavior; the narrator MAY record
    // a concrete development with a hidden memory block (never auto-control the player).
    return "\n\n<LC_PRIVATE>\n" +
      "### Stop the story. Use this card now.\n" +
      "New social pressure: " + seed.actor + " feels " + (seed.pressure || seed.category) +
      " toward " + seed.target + ". In your next response, " + seed.actor + " MUST act on this now so it visibly changes the scene. Do not state the label or mention this note.\n" +
      "If something concrete happens with " + seed.actor + ", record it after the story on its own lines. The system removes it before the player sees it, so it will not break the scene. Minimum is OWNER + OCCURRENCE; OWNER_THOUGHT and STATUS are optional (STATUS defaults to active):\n" +
      "<LC_MEMORY>\n" +
      "OWNER: " + seed.actor + "\n" +
      "OCCURRENCE: one sentence of what happened\n" +
      "OWNER_THOUGHT: (optional) brief first-person or close-third thought from " + seed.actor + "\n" +
      "STATUS: (optional) active | resolved\n" +
      "</LC_MEMORY>\n" +
      "</LC_PRIVATE>";
  }

  // Most recent concrete development for a card, kept to one compact line.
  function latestThreadLine(b) {
    let latest = cleanText(b.event);
    if (!latest && Array.isArray(b.eventLog) && b.eventLog.length) {
      latest = cleanText(b.eventLog[b.eventLog.length - 1]);
    }
    if (latest.length > 160) latest = latest.slice(0, 157) + "...";
    return latest;
  }

  // A thread is scene-relevant when a NON-protagonist participant is in scene.
  // (The protagonist is always "present", so target-is-protagonist alone does not
  // count; the NPC owner must be present.)
  function isSceneRelevant(b, scene) {
    // A world event is not a person -- it is always in play, whatever the scene mode.
    if (isEventBucket(b)) return true;
    const p = playerName();
    const owner = cleanName(b.owner);
    const tgt = cleanName(b.target);
    const ownerIn = owner && owner !== p && scene.indexOf(owner) !== -1;
    const targetIn = tgt && tgt !== p && scene.indexOf(tgt) !== -1;
    return !!(ownerIn || targetIn);
  }

  // Always-on block. Injects compact full details of the selected active Life Cards
  // every turn. SCENE_RELEVANCE_MODE controls selection: strict = scene-relevant only,
  // off = any active, hybrid = scene-relevant first then fill.
  function buildActiveThreadsBlock() {
    const cg = ensureState();
    const scene = cg.sceneActors || [];
    const mode = LC.sceneRelevanceMode || "off";
    const owners = Object.keys(cg.cards || {});

    const active = [];
    for (let i = 0; i < owners.length; i++) {
      const b = cg.cards[owners[i]];
      if (!cardHasContent(b) || !lifeStatusIsActive(b.status)) continue;
      active.push({ owner: owners[i], bucket: b });
    }
    if (!active.length) { cg.lastCandidateTrace = []; cg.lastSelectedTrace = []; cg.lastActiveSelected = 0; return ""; }

    // Debug trace: every active card + whether it is scene-relevant this turn.
    cg.lastCandidateTrace = active.map(function(c) {
      return c.owner + "->" + (cleanText(c.bucket.target) || "?") +
        (isSceneRelevant(c.bucket, scene) ? " [in-scene]" : " [off-scene]");
    });

    let selected;
    if (mode === "off") {
      selected = active.slice(0, LC.maxActive);
    } else if (mode === "hybrid") {
      const relevant = active.filter(function(c) { return isSceneRelevant(c.bucket, scene); });
      const rest = active.filter(function(c) { return !isSceneRelevant(c.bucket, scene); });
      selected = relevant.concat(rest).slice(0, LC.maxActive);
    } else { // strict
      selected = active.filter(function(c) { return isSceneRelevant(c.bucket, scene); }).slice(0, LC.maxActive);
    }
    cg.lastActiveSelected = selected.length;
    cg.lastSelectedTrace = selected.map(function(c) {
      return c.owner + "->" + (cleanText(c.bucket.target) || "?");
    });
    if (!selected.length) return "";

    // Render each card as a distinct multi-line block (more salient than one line).
    const renderEntry = function(c) {
      const b = c.bucket;
      // World event: no owner/target/pressure -- just the event and who-decides framing.
      if (isEventBucket(b)) {
        return "WORLD EVENT (in play now -- YOU decide which characters are pulled into it):\n" +
          cleanText(b.worldEvent);
      }
      let e = "OWNER: " + c.owner +
        "\nTARGET: " + (cleanText(b.target) || "someone") +
        "\nPRESSURE: " + (cleanText(b.pressure) || "unspoken pressure") +
        "\nMOMENTUM: " + (cleanText(b.momentum) || "low") +
        "\nSTATUS: " + (cleanText(b.status) || "active");
      const latest = latestThreadLine(b);
      if (latest) e += "\nLATEST: " + latest;
      return e;
    };

    // Split: a card is FRESH for FRESH_THREAD_WINDOW turns after it fires.
    const fresh = [], ongoing = [];
    for (let i = 0; i < selected.length; i++) {
      const created = (typeof selected[i].bucket.createdTurn === "number") ? selected[i].bucket.createdTurn : 0;
      if ((cg.turn - created) <= CFG.FRESH_THREAD_WINDOW) fresh.push(selected[i]);
      else ongoing.push(selected[i]);
    }
    cg.lastFreshThreads = fresh.map(function(c) { return c.owner; });

    // Write-back is a periodic CHECKPOINT, shown only on reminder/surface turns (the
    // same cadence as advanceThreadDormancy) -- NOT every turn. Framed as a look-back so
    // a development from a recent turn can still be recorded here. Conditional only:
    // never required, no no-op blocks (the owner && event gate still drops empty ones).
    const checkpointTurn = isCheckpointTurn();
    if (checkpointTurn) cg.lastCheckpointTurn = cg.turn; // stamp the checkpoint's OWN counter

    // Forceful, active framing (Dynamic Large responds better when the card reads as a
    // live scene driver, not background info). Not subtle.
    let body = "### Stop the story. Use these cards now.\n" +
      "Every active life thread below is in play in the CURRENT scene. Your next response must use them now -- make them visibly drive the characters' behavior, choices, dialogue, and reactions. Do not state the labels.\n";

    // PRIMARY: the Life Card details lead. This is story-driving context the narrator
    // should USE in the prose. (Front-loading the write-back task made the model
    // deprioritize the card, so the card and its nudge always come first now.)
    if (fresh.length) {
      body += "\nFRESH PRESSURE (new this turn -- you MUST act on this in THIS scene now):\n" +
        fresh.map(renderEntry).join("\n\n") + "\n";
    }
    if (ongoing.length) {
      body += "\nONGOING THREADS:\n" + ongoing.map(renderEntry).join("\n\n") + "\n";
    }

    // PRIMARY: single forceful story-driver nudge. Not subtle.
    body += (mode === "strict")
      ? "\n### Use these now. Each active life thread is REQUIRED to create a visible consequence in this scene -- dialogue, gossip, tension, attraction, avoidance, conflict, reactions, choices, or a relationship shift. Do not state the labels.\n"
      : "\n### Use these now. These pressures are live and drive the current story even when the characters are off-screen -- they are REQUIRED to surface as visible consequences: dialogue, gossip, shifting moods, attitudes, alliances, tension, attraction, avoidance, conflict, reactions, choices, relationship shifts. Do not state the labels.\n";

    // SECONDARY: trailing write-back, only on reminder/checkpoint turns, and visibly
    // SEPARATED from the story context above (a divider + "does not change the story")
    // so it reads as optional bookkeeping, never as competing with using the card.
    // Conditional only; the owner && event gate still drops empty/no-occurrence blocks.
    if (checkpointTurn) {
      const triggerList = selected.map(function(c) {
        const b = c.bucket;
        return c.owner + " (" + (cleanText(b.pressure) || "pressure") + " toward " + (cleanText(b.target) || "someone") + ")";
      }).join("; ");
      body += "\n---\n" +
        "Separate bookkeeping (does NOT change the story above, and is not part of the scene): if any active Life Card has visibly developed recently, then AFTER the story add one hidden block. Only if it actually developed -- otherwise skip it entirely. The system removes the block before the player sees it.\n" +
        "Minimum is OWNER + one OCCURRENCE line. OWNER_THOUGHT and STATUS are optional (STATUS defaults to active).\n" +
        "<LC_MEMORY>\n" +
        "OWNER: an exact owner name from the list above\n" +
        "OCCURRENCE: one sentence of what happened\n" +
        "OWNER_THOUGHT: (optional) brief first-person or close-third thought from that owner\n" +
        "STATUS: (optional) active | resolved\n" +
        "</LC_MEMORY>\n" +
        "Example (fictional -- copy the FORMAT, not these names; use an owner from above):\n" +
        "<LC_MEMORY>\n" +
        "OWNER: Mara\n" +
        "OCCURRENCE: Mara cut Jonah off mid-sentence and turned her back on him.\n" +
        "OWNER_THOUGHT: He always has to be right -- I'm done giving him the last word.\n" +
        "STATUS: active\n" +
        "</LC_MEMORY>\n" +
        "Cards that may have developed: " + triggerList + ".\n";
    }

    return "\n\n<LC_PRIVATE>\n" +
      (LC.forceActiveCardTrigger && CFG.ACTIVE_SHARED_TRIGGER ? CFG.ACTIVE_SHARED_TRIGGER + "\n" : "") +
      body +
      "</LC_PRIVATE>";
  }

  // True when THIS turn is a write-back CHECKPOINT turn -- on the CHECKPOINT_EVERY
  // cadence (its OWN counter cg.lastCheckpointTurn, independent of dormancy), and only
  // when at least one active (non-dormant/resolved) card exists. READ-ONLY: it does NOT
  // advance the counter, age cards, or change dormancy. The caller stamps
  // cg.lastCheckpointTurn when it actually shows the checkpoint. Does not affect lifespan.
  function isCheckpointTurn() {
    const cg = ensureState();
    if ((cg.turn - (cg.lastCheckpointTurn || 0)) < (LC.checkpointEvery || CFG.CHECKPOINT_EVERY)) return false;
    const owners = Object.keys(cg.cards || {});
    for (let i = 0; i < owners.length; i++) {
      const b = cg.cards[owners[i]];
      if (!cardHasContent(b)) continue;
      if (isEventBucket(b)) continue; // events take no narrator write-back
      const st = cleanText(b.status).toLowerCase();
      if (st === "resolved" || st === "dormant") continue;
      return true;
    }
    return false;
  }

  // Dormancy progression on the THREAD_REMINDER_EVERY cadence (its own counter,
  // cg.lastThreadReminderTurn). Counts appearances and archives threads that go stale.
  // INDEPENDENT of the write-back checkpoint cadence. Injects NO context text.
  function advanceThreadDormancy() {
    const cg = ensureState();
    if (cg.turn - (cg.lastThreadReminderTurn || 0) < (LC.threadReminderEvery || CFG.THREAD_REMINDER_EVERY)) return;

    const owners = Object.keys(cg.cards || {});
    const candidates = [];
    for (let i = 0; i < owners.length; i++) {
      const b = cg.cards[owners[i]];
      if (!cardHasContent(b)) continue;
      const st = cleanText(b.status).toLowerCase();
      if (st === "resolved" || st === "dormant") continue;
      candidates.push(b);
    }
    if (!candidates.length) return;

    cg.lastThreadReminderTurn = cg.turn;
    const show = candidates.slice(0, CFG.THREAD_REMINDER_MAX);
    for (let i = 0; i < show.length; i++) {
      const b = show[i];
      b.reminderCount = (Number(b.reminderCount) || 0) + 1;
      if (b.reminderCount >= CFG.THREAD_REMINDERS_BEFORE_DORMANT) makeDormant(b);
    }
  }

  function seedAppearsInOutput(seed, outputText) {
    if (!seed || !outputText) return false;
    if (!containsWholeWord(outputText, seed.actor)) return false;
    if (seed.target === playerName()) return true;
    return containsWholeWord(outputText, seed.target);
  }

  function maybeAutoCompleteOnscreenSeed(outputText) {
    if (!CFG.AUTO_COMPLETE_ONSCREEN_SEEDS) return false;

    const cg = ensureState();
    const seed = cg.pendingSeed;
    if (!seed) return false;
    if (!seedAppearsInOutput(seed, outputText)) return false;

    const bucket = ensureCardBucket(seed.actor);
    if (!bucket) return false;

    // The narrator showed actor + target but wrote no memory. Keep the abstract
    // pressure card and just mark it active. Do NOT invent canned event text;
    // a story-specific EVENT only ever comes from a narrator memory block.
    bucket.target = seed.target;
    if (!cleanText(bucket.pressure)) bucket.pressure = seed.pressure;
    if (!cleanText(bucket.momentum)) bucket.momentum = seed.momentum;
    bucket.status = "active";

    appendEventLog(bucket, "[active] " + seed.pressure + " toward " + seed.target + " is in play.");

    cg.pendingSeed = null;
    cg.lastRoll = "AUTO active: " + seed.actor + " / " + seed.pressure + " -> " + seed.target;
    return true;
  }

  function parseMemoryBlock(block) {
    const lines = String(block || "").replace(/\r/g, "").split("\n");
    const data = {};
    let current = "";
    for (let i = 0; i < lines.length; i++) {
      const line = lines[i];
      const idx = line.indexOf(":");
      if (idx !== -1) {
        const key = line.slice(0, idx).trim().toLowerCase();
        const value = line.slice(idx + 1).trim();
        if (["owner", "event", "occurrence", "pressure", "momentum", "target", "status", "log", "owner_thought"].indexOf(key) !== -1) {
          current = key;
          data[current] = value;
          continue;
        }
      }
      if (current && line.trim()) {
        data[current] = cleanText((data[current] || "") + "\n" + line.trim());
      }
    }
    data.owner = cleanName(data.owner);
    // OCCURRENCE is the preferred label; EVENT remains a backwards-compatible alias.
    data.event = cleanText(data.occurrence || data.event);
    data.pressure = cleanText(data.pressure);
    data.target = cleanName(data.target);
    data.momentum = cleanText(data.momentum).toLowerCase();
    data.status = cleanText(data.status || "active").toLowerCase();
    data.log = cleanText(data.log || data.event);
    // Optional narrator-authored first-person interior line. Display-only; the engine
    // never reads it for targeting, pressure, momentum, status, or any logic.
    data.owner_thought = cleanText(data.owner_thought);
    return data;
  }

  function extractBlocks(text, tag) {
    const blocks = [];
    const open = "<" + tag + ">";
    const close = "</" + tag + ">";
    let src = String(text || "");
    let start = src.indexOf(open);
    while (start !== -1) {
      const end = src.indexOf(close, start + open.length);
      if (end === -1) break;
      blocks.push(src.slice(start + open.length, end));
      start = src.indexOf(open, end + close.length);
    }
    return blocks;
  }

  function stripBlocks(text) {
    let out = String(text || "");
    // LC_* are current; CG_* are legacy and still stripped so an in-flight story
    // that emits an old tag never leaks it into the visible output.
    const tags = ["LC_PRIVATE", "LC_SEED", "LC_MEMORY", "LC_CARDS", "CG_PRIVATE", "CG_SEED", "CG_MEMORY", "CG_CARDS"];
    for (let t = 0; t < tags.length; t++) {
      const open = "<" + tags[t] + ">";
      const close = "</" + tags[t] + ">";
      let guard = 0;
      while (guard < 20) {
        const start = out.indexOf(open);
        if (start === -1) break;
        const end = out.indexOf(close, start + open.length);
        if (end === -1) {
          out = out.slice(0, start);
          break;
        }
        out = out.slice(0, start) + out.slice(end + close.length);
        guard++;
      }
    }
    return cleanText(out);
  }

  function stripSelectedBlocks(text, tags) {
    let out = String(text || "");
    for (let t = 0; t < tags.length; t++) {
      const open = "<" + tags[t] + ">";
      const close = "</" + tags[t] + ">";
      let guard = 0;
      while (guard < 20) {
        const start = out.indexOf(open);
        if (start === -1) break;
        const end = out.indexOf(close, start + open.length);
        if (end === -1) {
          out = out.slice(0, start);
          break;
        }
        out = out.slice(0, start) + out.slice(end + close.length);
        guard++;
      }
    }
    return out;
  }

  // Output-safety fallback. After the script removes hidden/private/thought blocks, the
  // visible text can end up empty (e.g. an opening <LC_PRIVATE> with no closing tag strips
  // to end-of-string). AI Dungeon treats an empty OR whitespace-only return (including a
  // plain space) as "The AI service returned an empty response". This guarantees a
  // non-empty return WITHOUT ever exposing private tags: it strips any remaining blocks,
  // and if nothing visible is left, returns a zero-width space the player never sees.
  // For normal output (real visible text present) it returns that text unchanged.
  function safeVisibleOutput(value) {
    const out = cleanText(stripBlocks(value));
    return out || CFG.EMPTY_OUTPUT_FALLBACK;
  }

  // True when `s` has no VISIBLE content: null/undefined, or only whitespace and/or
  // zero-width characters. NOTE: \u200B (zero-width space) is NOT matched by \s, so it is
  // listed explicitly here -- otherwise a "fallback" return would look non-empty and a
  // genuinely empty turn would slip through unlogged.
  function isBlank(s) {
    if (s == null) return true;
    return String(s).replace(/[\s\u200B\u200C\u200D\uFEFF]/g, "").length === 0;
  }

  // Records + logs WHICH return branch produced the safe Unicode fallback, so an empty
  // response can be traced to its source. Always emits a console line (visible in the AID
  // script console) and stashes the reason in state for the debug card. Never throws.
  function debugFallback(branch, detail) {
    const msg = "LivingCharacters EMPTY-FALLBACK [" + branch + "]" + (detail ? " :: " + detail : "");
    try {
      if (typeof log === "function") log(msg);
      else if (typeof console !== "undefined" && console && typeof console.log === "function") console.log(msg);
    } catch (e) {}
    try {
      const cg = ensureState();
      cg.lastFallback = msg + " (turn " + (cg.turn || 0) + ")";
    } catch (e) {}
  }

  // FINAL OUTPUT BOUNDARY for every hook return. Guarantees a non-empty, non-whitespace,
  // non-null string so AI Dungeon never reports an empty response (output) or "Unable to
  // run scenario scripts" (input). This is the single authoritative place the Unicode
  // fallback is applied -- helpers may return "" freely; this reconciles it.
  //   - output: must carry NO private tags AND keep its LEADING separator (the space/newline
  //     that joins this continuation to the prior story); blank -> fallback.
  //   - input/context: blank -> restore the untouched original if it has content, else fallback.
  function finalizeResult(hook, result, original) {
    const out = (typeof result === "string") ? result : (result == null ? "" : String(result));
    if (hook === "output") {
      // Remove private tags WITHOUT trimming (stripSelectedBlocks does not call cleanText),
      // capture the model's leading separator, then tidy the BODY and re-attach one
      // normalized separator. This stops the narrator's text jamming onto the previous text.
      const stripped = stripSelectedBlocks(out, ["LC_PRIVATE", "LC_SEED", "LC_MEMORY", "LC_CARDS", "CG_PRIVATE", "CG_SEED", "CG_MEMORY", "CG_CARDS"]);
      const visible = leadSeparator(leadingWhitespace(stripped)) + cleanText(stripped);
      if (!isBlank(visible)) return visible;
      debugFallback("finalize:output", "blank visible output after strip");
      return CFG.EMPTY_OUTPUT_FALLBACK;
    }
    if (!isBlank(out)) return out;
    // Blank input/context: prefer the original text (don't destroy a real turn), else fallback.
    if (!isBlank(original)) {
      debugFallback("finalize:" + hook + ":restore-original", "blank result; restored original text");
      return String(original);
    }
    debugFallback("finalize:" + hook + ":unicode", "blank result and blank original");
    return CFG.EMPTY_OUTPUT_FALLBACK;
  }


  // THOUGHT CARD SYSTEM
  // --------------------------------------------------------------------------
  // Part of LIVING CHARACTERS, a LivingNarratives project.
  // (https://github.com/LivingNarratives/LivingCharacters)
  //
  // This module belongs to Living Characters. Thought Cards are a feature of
  // Living Characters, kept deliberately separate from the Life Card engine.
  //
  // WHAT THOUGHT CARDS ARE:
  //   Player-facing thought journals. One numbered, human-readable log per
  //   character (titled "Name - Thoughts"; a temporary 💭 marks recent updates) to read
  //   to follow a character's emotional progression over many turns.
  //
  // WHAT THOUGHT CARDS ARE NOT:
  //   - NOT memory. NOT a brain. NOT Inner Self.
  //   - They do NOT enter the story context (the Thought Card uses a non-matching
  //     key, so the AI Dungeon front-end never injects it).
  //   - They are NOT read by the AI narrator.
  //   - They do NOT affect character behavior, Life Card creation, targeting,
  //     pressure, momentum, dormancy, selection, or any story logic.
  //
  // WHAT THIS MODULE DOES (the entire job):
  //   1. ASK   - inject ONE temporary first-person parenthetical request into
  //              context (the only thing this system ever puts in context).
  //   2. CAPTURE- read that leading parenthetical back from the model's output
  //              and strip it from the visible story.
  //   3. STORE - append it (numbered) to the character's Thought Card for the
  //              player to read later. Nothing reads it back.
  //
  // The PARENTHETICAL REQUEST is the only temporary instruction injected into
  // context. The stored Thought Card itself is never injected (non-matching key).
  // No XML, no LC_MEMORY, no write-back blocks, no detectors. Runs independently
  // of Life Cards (a character can have thoughts with no active Life Card).
  // ==========================================================================
  const THOUGHT_KEYS = ["thoughts_enabled", "thought_characters", "thought_interval", "thought_formation_chance", "thought_scene_mode", "thought_order"];

  // THOUGHT CARDS CONFIG
  // The editable Story Card that controls the separate Thought Card system. It is
  // distinct from the LIVING CHARACTERS CONFIG (Life Cards) card on purpose, so the
  // two systems never share settings. Defaults below are conservative and opt-in.
  function defaultThoughtConfigEntry() {
    return [
      "THOUGHT CARDS (separate from Life Cards)",
      "",
      "Important:",
"Thought Cards are not compatible with AI Dungeon's Optimized Context feature.",
"Disable Optimized Context when using Thought Cards.",
      "",
      "THOUGHTS_ENABLED:",
      "false",
      "",
      "THOUGHT_CHARACTERS:",
      "( one name per line )",
      "",
      "THOUGHT_INTERVAL:",
      "5",
      "",
      "THOUGHT_FORMATION_CHANCE:",
      "50",
      "",
      "THOUGHT_SCENE_MODE:",
      "scene",
      "",
      "THOUGHT_ORDER:",
      "ascending",
      "( ascending = 1,2,3,4  |  descending = 4,3,2,1 )"
    ].join("\n");
  }

  function ensureThoughtConfigCard() {
    const existing = findStoryCardByKeys(CFG.THOUGHT_CONFIG_CARD_KEY) || findStoryCardByTitle(CFG.THOUGHT_CONFIG_CARD_TITLE);
    if (existing) return existing;
    return createOrPatchStoryCard(
      CFG.THOUGHT_CONFIG_CARD_TITLE,
      CFG.THOUGHT_CONFIG_CARD_TYPE,
      CFG.THOUGHT_CONFIG_CARD_KEY,
      defaultThoughtConfigEntry(),
      "Thought Cards are a player-facing journal. They never enter the story context."
    );
  }

  function buildThoughtConfig() {
    ensureThoughtConfigCard();
    const card = findStoryCardByKeys(CFG.THOUGHT_CONFIG_CARD_KEY) || findStoryCardByTitle(CFG.THOUGHT_CONFIG_CARD_TITLE);
    const sections = parseConfigText(card && card.entry, THOUGHT_KEYS);
    const enabled = toBoolOr(configFirst(sections, "thoughts_enabled", null), CFG.THOUGHTS_ENABLED_DEFAULT);
    const characters = configList(sections, "thought_characters");
    const interval = Math.max(1, toIntOr(configFirst(sections, "thought_interval", CFG.THOUGHT_INTERVAL_DEFAULT), CFG.THOUGHT_INTERVAL_DEFAULT));
    const chance = Math.max(0, Math.min(100, toIntOr(configFirst(sections, "thought_formation_chance", CFG.THOUGHT_CHANCE_DEFAULT), CFG.THOUGHT_CHANCE_DEFAULT)));
    const maxThoughts = CFG.MAX_THOUGHTS_DEFAULT; // hardcoded 10; NOT a user config option
    // THOUGHT_SCENE_MODE: scene (default) | recent | roster. Validated; bad values ->
    // default "scene". Thought scene relevance is SEPARATE from Life Card scene relevance:
    // it only decides which character may get a journal entry, and must not affect Life
    // Card targeting, pressure selection, momentum, or story logic.
    let sceneMode = String(configFirst(sections, "thought_scene_mode", CFG.THOUGHT_SCENE_MODE_DEFAULT)).toLowerCase().replace(/\s+/g, "");
    if (["scene", "recent", "roster"].indexOf(sceneMode) === -1) sceneMode = CFG.THOUGHT_SCENE_MODE_DEFAULT;
    // THOUGHT_ORDER: ascending (default) | descending. Display only. "desc"/"newest"/"new"
    // are accepted as friendly aliases for descending; anything else -> ascending.
    let order = String(configFirst(sections, "thought_order", CFG.THOUGHT_ORDER_DEFAULT)).toLowerCase().replace(/\s+/g, "");
    order = (["descending", "desc", "newest", "new", "reverse"].indexOf(order) !== -1) ? "descending" : "ascending";
    return { enabled: enabled, characters: characters, interval: interval, chance: chance, maxThoughts: maxThoughts, sceneMode: sceneMode, order: order };
  }

  // Separate state slice -- never touches chaosGoblinV2 (Life Card state).
  function ensureThoughtState() {
    if (!globalThis.state || typeof state !== "object") globalThis.state = {};
    if (!state.livingThoughts || typeof state.livingThoughts !== "object") {
      state.livingThoughts = { version: CFG.VERSION, lastThoughtTurn: 0, pendingChar: "", byChar: {}, seq: {}, markedChar: "", markedTurn: 0 };
    }
    const ts = state.livingThoughts;
    if (!ts.byChar || typeof ts.byChar !== "object") ts.byChar = {};
    if (!ts.seq || typeof ts.seq !== "object") ts.seq = {};   // per-character PERMANENT thought counter
    if (typeof ts.lastThoughtTurn !== "number") ts.lastThoughtTurn = 0;
    if (typeof ts.markedChar !== "string") ts.markedChar = "";   // 💭 "recently updated" marker
    if (typeof ts.markedTurn !== "number") ts.markedTurn = 0;
    // Migrate legacy storage. Older saves stored byChar[name] as an array of plain
    // strings whose displayed number came from the array position -- so removing an old
    // thought renumbered the survivors. Convert each entry to a { n, text } object with a
    // PERMANENT number, and seed ts.seq[name] with the highest number seen so future
    // thoughts keep counting up (never reused, never reset).
    const names = Object.keys(ts.byChar);
    for (let i = 0; i < names.length; i++) {
      const nm = names[i];
      const arr = ts.byChar[nm];
      if (!Array.isArray(arr)) { ts.byChar[nm] = []; continue; }
      let maxN = (typeof ts.seq[nm] === "number") ? ts.seq[nm] : 0;
      for (let j = 0; j < arr.length; j++) {
        const e = arr[j];
        if (e && typeof e === "object" && typeof e.text === "string") {
          if (typeof e.n !== "number") e.n = ++maxN;
          else if (e.n > maxN) maxN = e.n;
        } else {
          arr[j] = { n: ++maxN, text: String(e == null ? "" : e) };
        }
      }
      if (maxN > (typeof ts.seq[nm] === "number" ? ts.seq[nm] : 0)) ts.seq[nm] = maxN;
    }
    return ts;
  }

  // Stable, non-matching key for lookup/recreation (UNCHANGED: lc-thoughts:name).
  function thoughtCardToken(name) {
    return CFG.THOUGHT_CARD_KEY_PREFIX + keyName(name);
  }

  // Character-first display title: "Name - Thoughts", with a TEMPORARY "💭 " marker
  // prepended while the card is recently updated. The KEY (above) identifies the card,
  // so adding/removing the marker just safely re-titles the same existing card.
  function thoughtCardTitle(name, marked) {
    return (marked ? CFG.THOUGHT_MARKER + " " : "") + name + CFG.THOUGHT_CARD_TITLE_SUFFIX;
  }

  // Thought Card: "Name - Thoughts" (temporary "💭 " prepended while recently updated)
  //   Purpose:     a numbered, player-readable thought journal for one character.
  //   Not purpose: not memory, not a brain, not context, not narrator guidance,
  //                not a logic source.
  // Render a character's Thought Card from the stored list. The keys are a
  // non-matching token (CFG.THOUGHT_CARD_KEY_PREFIX + name, no plain-name trigger),
  // so the AI Dungeon front-end never matches it against story text and the card is
  // NEVER injected into context. Type is "Custom" purely for player-side organization.
  // One stored thought renders as "N. text" where N is the PERMANENT number.
  function thoughtLine(item) {
    return item.n + ". " + item.text;
  }
  function renderThoughtLines(items) {
    const lines = [];
    for (let i = 0; i < items.length; i++) lines.push(thoughtLine(items[i]));
    return lines.join("\n");
  }

  // Greedily take the NEWEST items whose rendered lines fit within `max` characters
  // (joined by newlines). Returns { kept, dropped }, both in ascending (oldest -> newest)
  // order. At least one item is always kept, so the newest thought is never lost even if
  // it alone exceeds `max`.
  function fitNewest(items, max) {
    const kept = [];
    let len = 0;
    for (let i = items.length - 1; i >= 0; i--) {
      const line = thoughtLine(items[i]);
      const add = (kept.length ? 1 : 0) + line.length; // +1 for the joining newline
      if (kept.length > 0 && (len + add) > max) break;
      len += add;
      kept.unshift(items[i]);
    }
    const dropped = items.slice(0, items.length - kept.length);
    return { kept: kept, dropped: dropped };
  }

  // Render a character's Thought Card from the stored list, split by CHARACTER COUNT:
  //   - Entry  = the NEWEST thoughts, up to ~THOUGHT_ENTRY_MAX_CHARS.
  //   - Notes  = the OLDER overflow from Entry, up to ~THOUGHT_NOTES_MAX_CHARS.
  //   - Thoughts too old to fit even in Notes are dropped (oldest-first) from storage.
  // New thoughts always land in Entry; older ones flow Entry -> Notes; the oldest Notes
  // thoughts are trimmed only when Notes has no room. Numbers are permanent, so trimming
  // never renumbers anything that remains.
  function syncThoughtCard(name, TC, marked) {
    const ts = ensureThoughtState();
    let list = Array.isArray(ts.byChar[name]) ? ts.byChar[name] : [];

    const entryFit = fitNewest(list, CFG.THOUGHT_ENTRY_MAX_CHARS);
    const notesFit = fitNewest(entryFit.dropped, CFG.THOUGHT_NOTES_MAX_CHARS);

    // Drop the oldest thoughts that no longer fit anywhere so state stays bounded.
    const keepCount = entryFit.kept.length + notesFit.kept.length;
    if (keepCount < list.length) {
      list = list.slice(list.length - keepCount);
      ts.byChar[name] = list;
    }

    // DISPLAY ORDER (storage is always oldest->newest, so overflow/trim above is unaffected).
    // Descending reverses each field's kept items so the newest number sits at the top:
    //   ascending  Entry [3,4,5] Notes [1,2]  ->  3,4,5 / 1,2
    //   descending Entry [5,4,3] Notes [2,1]  ->  5,4,3 / 2,1
    const desc = TC && TC.order === "descending";
    const entryItems = desc ? entryFit.kept.slice().reverse() : entryFit.kept;
    const notesItems = desc ? notesFit.kept.slice().reverse() : notesFit.kept;
    const entry = entryItems.length ? renderThoughtLines(entryItems) : "(no thoughts yet)";
    const notes = notesItems.length ? renderThoughtLines(notesItems) : "";

    // On a NEW thought (marked), DELETE the existing card first so the create below makes
    // a BRAND-NEW card. AID's card menu is sorted by card CREATION order (newest first)
    // and ignores array position, so recreating is the only thing that actually lifts a
    // freshly-updated card to the top -- it becomes "newest" like a first-time thinker's
    // card. All thought text lives in state (ts.byChar), so the deleted card is fully
    // rebuilt from entry/notes below and nothing is lost. On non-marked syncs (e.g. the
    // 💭 marker expiring) we patch in place, so the card keeps its position.
    if (marked) removeStoryCardByKeys(thoughtCardToken(name));

    createOrPatchStoryCard(
      thoughtCardTitle(name, !!marked), // "Name - Thoughts" (+ 💭 while recently updated)
      CFG.THOUGHT_CARD_TYPE,
      thoughtCardToken(name),     // stable key: lc-thoughts:name (UNCHANGED) -- re-titling is safe
      entry,                      // newest thoughts
      notes                       // older overflow (Notes field)
    );
  }

  // Strip the 💭 marker from EVERY Thought Card title (optionally keeping it on one
  // character). Title-only: it re-titles the card in place and never touches Entry,
  // Notes, storage, numbering, or rollover. Scans the live story cards directly (not
  // just ts.markedChar), so it SELF-CORRECTS cases where a prior glitch left 💭 stuck
  // on more than one card. Ensures there is never more than one marked Thought Card.
  function clearAllThoughtMarkers(keepName) {
    if (!Array.isArray(globalThis.storyCards)) return;
    const prefix = CFG.THOUGHT_MARKER + " ";
    const keepTitle = keepName ? (cleanName(keepName) + CFG.THOUGHT_CARD_TITLE_SUFFIX) : "";
    for (let i = 0; i < storyCards.length; i++) {
      const card = storyCards[i];
      if (!card || typeof card.title !== "string") continue;
      // Only our Thought Cards (non-matching key prefix), and only if currently marked.
      if (String(card.keys || "").indexOf(CFG.THOUGHT_CARD_KEY_PREFIX) !== 0) continue;
      if (card.title.indexOf(prefix) !== 0) continue;
      const bare = card.title.slice(prefix.length);
      if (keepTitle && bare === keepTitle) continue; // leave the one we're (re)marking
      card.title = bare;
    }
  }

  // Normalize a thought for loose duplicate comparison: lowercase, strip punctuation,
  // collapse whitespace.
  function normThought(t) {
    return String(t || "").toLowerCase().replace(/[^a-z0-9\s]/g, "").replace(/\s+/g, " ").trim();
  }

  // True when two thoughts are near-duplicates -- exact-normalized match, or high token
  // overlap (Jaccard on unique words). Catches "I hope this works" vs "I really hope this
  // works out" so repetitive/samey thoughts do not pile up.
  function thoughtsTooSimilar(a, b) {
    a = normThought(a); b = normThought(b);
    if (!a || !b) return false;
    if (a === b) return true;
    const wa = a.split(" "), wb = b.split(" ");
    const setA = {}, setB = {};
    for (let i = 0; i < wa.length; i++) setA[wa[i]] = 1;
    for (let i = 0; i < wb.length; i++) setB[wb[i]] = 1;
    const keysA = Object.keys(setA), keysB = Object.keys(setB);
    let inter = 0;
    for (let i = 0; i < keysA.length; i++) if (setB[keysA[i]]) inter++;
    const uni = keysA.length + keysB.length - inter;
    return uni > 0 && (inter / uni) >= 0.6;
  }

  // Append a thought with a PERMANENT number. No overwrite, no keys. The stored list is
  // NOT capped by count -- syncThoughtCard bounds it by character count (Entry/Notes), so
  // removing old thoughts never renumbers the survivors.
  function appendThought(name, text, TC) {
    const ts = ensureThoughtState();
    name = cleanName(name);
    text = cleanText(text);
    if (!name || !text) return false;
    if (!Array.isArray(ts.byChar[name])) ts.byChar[name] = [];
    const arr = ts.byChar[name];
    // Skip near-duplicates against this character's recent thoughts (not just the immediately
    // previous exact string), so repetitive/samey thoughts do not accumulate.
    const recent = arr.slice(-5);
    for (let i = 0; i < recent.length; i++) {
      if (thoughtsTooSimilar(recent[i].text, text)) return false;
    }
    // Per-character counter that only ever increases: if the last thought was #10, the
    // next is #11 even after #1 is deleted. Never reset, never reused.
    if (typeof ts.seq[name] !== "number") ts.seq[name] = 0;
    ts.seq[name] += 1;
    arr.push({ n: ts.seq[name], text: text });
    // Visual activity marker: 💭 moves to the card that just updated. Strip 💭 off EVERY
    // Thought Card first (self-correcting any stuck markers), then mark only this one, so
    // exactly one card carries 💭. (Also clears on a timer; see expireThoughtMarker.)
    // Display-only -- does not touch Life Cards, storage, or Entry/Notes.
    clearAllThoughtMarkers(name);
    ts.markedChar = name;
    ts.markedTurn = (ensureState().turn) || 0;
    syncThoughtCard(name, TC, true);
    return true;
  }

  // Clear the 💭 marker after THOUGHT_MARKER_TURNS turns so it reads as "recent" activity
  // and comes back down on its own. Runs once per context turn; re-titles one card at most.
  function expireThoughtMarker() {
    const ts = ensureThoughtState();
    if (!ts.markedChar) return;
    const turn = (ensureState().turn) || 0;
    if ((turn - (ts.markedTurn || 0)) >= CFG.THOUGHT_MARKER_TURNS) {
      syncThoughtCard(ts.markedChar, buildThoughtConfig(), false); // drop the 💭
      ts.markedChar = "";
    }
  }


  // Candidate thought-characters for THIS turn, per THOUGHT_SCENE_MODE.
  // Thought Cards are journals of what characters are thinking in/around the CURRENT
  // story moment, so by default only scene-relevant characters are eligible. There is
  // NO silent fallback to the full roster -- if nobody relevant is present, the caller
  // generates nothing this turn (an off-screen thought feels disconnected/confusing).
  //   scene  -> ONLY characters detected in the current scene (tight trailing window).
  //   recent -> characters in the wider recent-scene window (current + just-mentioned).
  //   roster -> ANY configured thought character (opt-in; intentional off-screen/"chaos"
  //             thoughts). This is the ONLY mode that ignores scene relevance.
  // Detection mirrors the Life Card scene detector (whole-word name match on the recent
  // story text), but uses the THOUGHT roster and its OWN window -- fully separate from
  // Life Card scene relevance.
  function thoughtCandidates(TC, contextText) {
    // Thought Cards are player-facing journals, so they should only come from
    // scene-relevant characters. This prevents disconnected off-screen thoughts from
    // random roster characters (e.g. a thought appearing for someone not in the scene).
    const roster = unique(TC.characters || []);
    if (!roster.length) return [];
    // THOUGHT_SCENE_MODE handling:
    //   roster -> opt-in off-screen/"chaos" mode: ANY configured thought character.
    //             This is the ONLY mode that skips the scene-relevance gate.
    if (TC.sceneMode === "roster") return roster;
    const recent = recentSceneText(contextText);  // history tail (up to SCENE_SCAN_CHARS)
    //   scene  -> tight CURRENT-scene window (default).
    //   recent -> wider recent-scene window (current + just-mentioned).
    const tight = recent.length > CFG.THOUGHT_SCENE_TIGHT_CHARS ? recent.slice(-CFG.THOUGHT_SCENE_TIGHT_CHARS) : recent;
    const text = (TC.sceneMode === "recent") ? recent : tight;
    // The first-person POV lead (protagonist) is rarely written by name in the scene, but
    // is always present. When PROTAGONIST_ALWAYS_PRESENT is on, keep the protagonist
    // eligible even when unnamed -- mirroring the Life Card scene detector -- so an unnamed
    // lead like Sam can still be selected for a thought. Everyone else still needs a
    // whole-word name match in the current scene.
    // Fallback prevention: do NOT fall back to the full roster unless THOUGHT_SCENE_MODE
    // is explicitly "roster". If nobody is scene-relevant, return [] and the caller makes
    // no thought this turn -- an off-screen thought would feel disconnected.
    const pKey = LC.protagonist ? cleanName(LC.protagonist).toLowerCase() : "";
    return roster.filter(function(n) {
      if (pKey && LC.protagonistAlwaysPresent && cleanName(n).toLowerCase() === pKey) return true;
      return containsWholeWord(text, n);
    });
  }

  // Decide whether to fire a thought this turn; if so, select a character, stamp the
  // attempt, and return the ASK string (LC_PRIVATE-wrapped) to append to context.
  // Sets ts.pendingChar for the output hook. Independent of Life Cards.
  function buildThoughtAsk(contextText) {
    const ts = ensureThoughtState();
    ts.pendingChar = "";
    ts.candidates = []; // scene-relevant thinkers this turn (used by capture to reject mislabels)
    const TC = buildThoughtConfig();
    if (!TC.enabled || !TC.characters.length) return "";
    const turn = (ensureState().turn) || 0;
    if ((turn - (ts.lastThoughtTurn || 0)) < TC.interval) return ""; // interval gate
    if (randomInt(100) >= TC.chance) return "";                      // chance gate
    const cands = thoughtCandidates(TC, contextText);               // scene-relevant only
    if (!cands.length) return "";                                    // nobody relevant -> no thought
    const name = choose(cands);
    ts.pendingChar = name;
    ts.candidates = cands.slice(); // remember who was scene-relevant, for capture attribution
    ts.lastThoughtTurn = turn;
    // Names of the OTHER people currently in the scene (reuses the scene actors computed this
    // turn). Feeding the model the actual names is what makes it write "Jessica" instead of
    // "she" -- it can only use a name it is reminded of.
    const present = (ensureState().sceneActors || []).filter(function (n) { return n && n !== name; });
    const nameHint = present.length
      ? " Others present (name on first mention): " + present.join(", ") + "."
      : "";
    // ONE leading first-person parenthetical, name-LABELED so the thinker is unambiguous.
    // Naming rule: each other character is NAMED on first mention, then referred to with a
    // pronoun -- keeps who's-who clear without the clunky repeated-name reading.
    return "\n\n<LC_PRIVATE>\n" +
      "Begin your reply with ONE short parenthetical: " + name + "'s own private thought right now, in first person (I / me / my), ONE sentence, LABELED with their name. " +
      "Every other character must be named (first name) once before any pronoun or nickname; after that use normal flow of wording with pronouns." + nameHint + " " +
      "Make it a specific reaction to this exact moment, not a generic or repeated line. Then continue the story normally. The parenthetical is hidden from the player automatically. Format: (" + name + ": I ...)\n" +
      "</LC_PRIVATE>";
  }

  // Resolve a free-text name label (from a "Name:" thought prefix) to a configured Thought
  // character, case-insensitive and whitespace-normalized. Returns the canonical roster
  // name, or "" if the label is not a known Thought character. Used to ATTRIBUTE a labeled
  // thought to the correct card instead of blindly trusting the asked character.
  function resolveThoughtCharacter(label, TC) {
    const want = cleanName(label).toLowerCase();
    if (!want) return "";
    const roster = (TC && TC.characters) || [];
    for (let i = 0; i < roster.length; i++) {
      if (cleanName(roster[i]).toLowerCase() === want) return roster[i];
    }
    return "";
  }

  // Remove EVERY name-labeled thought parenthetical "(KnownThoughtChar: ...)" from text,
  // wherever it sits. Narrow on purpose: only parentheticals whose label resolves to a
  // CONFIGURED Thought character are touched, so ordinary narrative asides are left alone.
  // This is the BACKSTOP that stops a thought leaking into the visible story even when the
  // model places it mid- or end-of-reply instead of at the start.
  function stripLabeledThoughts(text, TC) {
    return String(text || "").replace(/\(\s*([A-Za-z][\w '\-]{0,30})\s*:\s*[^)]*\)/g, function (full, name) {
      return resolveThoughtCharacter(name, TC) ? "" : full;
    });
  }

  // Tidy the double-space / space-before-punctuation a removed mid-sentence parenthetical can
  // leave behind. Touches spaces/tabs only -- never newlines.
  function tidyThoughtSeam(text) {
    return String(text || "").replace(/[ \t]{2,}/g, " ").replace(/[ \t]+([.,!?;:])/g, "$1");
  }

  // Output-hook capture. Pulls the character's thought out of the reply and onto their card,
  // and -- crucially -- removes it from the visible story WHEREVER it appears so it can never
  // leak. Order:
  //   1) PRIMARY: a name-labeled thought "(KnownThoughtChar: ...)" ANYWHERE in the reply
  //      (start, middle, or end). Filed under the LABELED character (attribution fix), then
  //      removed from the text. This is the leak fix: a thought placed at the end is caught.
  //   2) FALLBACK (pending turn only): an unlabeled LEADING first-person parenthetical, filed
  //      under the asked character. Gated to pending turns so a narrative aside is not eaten.
  //   3) BACKSTOP (always): strip any remaining labeled thought parentheticals so none leak.
  function captureThought(original) {
    const ts = ensureThoughtState();
    const TC = buildThoughtConfig();
    let text = String(original || "");
    const asked = ts.pendingChar;                                    // who we asked (scene-relevant), or ""
    const allowed = Array.isArray(ts.candidates) ? ts.candidates : []; // scene-relevant thinkers this turn

    // A resolved label is TRUSTED only if it is who we asked OR another scene-relevant
    // candidate this turn. When two characters are near-identical (e.g. two attorneys),
    // the model often swaps the name in the parenthetical; an off-scene / not-asked label
    // is treated as a MISLABEL and its thought is filed under the character we ACTUALLY
    // asked -- never redirected to a character who was not in the scene.
    const isTrusted = function (who) {
      if (!who) return false;
      if (asked && who === asked) return true;
      return allowed.indexOf(who) !== -1;
    };

    // 1) PRIMARY: scan labeled parentheticals. Prefer one labeled with a TRUSTED character;
    //    otherwise keep the first resolvable one as the thought body to re-attribute.
    const labeled = /\(\s*([A-Za-z][\w '\-]{0,30})\s*:\s*([^)]*)\)/g;
    let mm, firstResolvable = null, trustedHit = null;
    while ((mm = labeled.exec(text)) !== null) {
      const who = resolveThoughtCharacter(mm[1], TC);
      if (!who) continue;
      const rec = { whole: mm[0], who: who, body: mm[2], index: mm.index };
      if (!firstResolvable) firstResolvable = rec;
      if (isTrusted(who)) { trustedHit = rec; break; }
    }
    const chosen = trustedHit || firstResolvable;
    if (chosen) {
      // Trusted label -> file under it. Untrusted label (mislabel) -> file under the asked
      // character; only when nothing was asked do we fall back to the label as best-effort.
      const target = trustedHit ? trustedHit.who : (asked || chosen.who);
      const body = cleanText(chosen.body);
      if (body) appendThought(target, body, TC);              // dedup happens inside appendThought
      text = text.slice(0, chosen.index) + text.slice(chosen.index + chosen.whole.length);
      text = stripLabeledThoughts(text, TC);                  // remove any extra labeled thoughts
      return tidyThoughtSeam(text);
    }

    // 2) FALLBACK: only when a thought was asked this turn, an unlabeled LEADING parenthetical.
    if (asked) {
      const m = /^\s*\(([^)]*)\)/.exec(text);
      if (m) {
        let inner = cleanText(m[1]);
        const lbl = /^([A-Za-z][\w '\-]{0,30}):\s*/.exec(inner);
        if (lbl) inner = inner.slice(lbl[0].length).trim();   // drop a stray label if present
        if (inner) {
          appendThought(asked, inner, TC);
          const rest = text.slice(m[0].length);
          return (rest && !/^\s/.test(rest)) ? " " + rest : rest;
        }
      }
    }

    // 3) BACKSTOP: even if nothing was captured, never let a labeled thought leak.
    return stripLabeledThoughts(text, TC);
  }

  function handleOutput(rawText) {
    const original = String(rawText || "");
    const parseSource = stripSelectedBlocks(original, ["LC_PRIVATE", "LC_SEED", "LC_CARDS", "CG_PRIVATE", "CG_SEED", "CG_CARDS"]);
    // Current tag is LC_MEMORY; also accept legacy CG_MEMORY from in-flight stories.
    let memoryBlocks = extractBlocks(parseSource, "LC_MEMORY");
    if (!memoryBlocks.length) memoryBlocks = extractBlocks(parseSource, "CG_MEMORY");
    const cg = ensureState();
    let parsedMemory = false;

    if (memoryBlocks.length) {
      const memory = parseMemoryBlock(memoryBlocks[0]);
      if (memory.owner && memory.event) {
        applyMemory(memory);
        parsedMemory = true;
        cg.lastRoll = "MEMORY parsed: " + memory.owner + " (" + memory.status + ")";
      } else {
        cg.lastRoll = "MEMORY block seen but unparseable (owner/event missing)";
      }
    }

    // Safety net: if the narrator used an on-screen seed but skipped <LC_MEMORY>,
    // mark the pressure card active (without inventing event text).
    if (!parsedMemory) {
      maybeAutoCompleteOnscreenSeed(original);
    }

    // THOUGHT CARD capture: ONLY when a thought was asked this turn (Thought system).
    // Captures the leading parenthetical and strips it from visible text. Independent of Life Cards.
    const visibleText = captureThought(original);

    syncCards();
    updateDebugCard();
    // Return the post-thought text WITHOUT trimming. The final output boundary
    // (finalizeResult) removes private tags, PRESERVES the model's leading separator so the
    // narration does not jam onto the prior story text, tidies the body, and applies the
    // empty/whitespace Unicode fallback -- so every return path (normal, early-exit, error)
    // is handled in one place.
    return visibleText;
  }

  function handleContext(rawText) {
    const cg = ensureState();
    cg.turn = (cg.turn || 0) + 1;
    cg.sceneActors = sceneActors(rawText); // who is in the current scene this turn
    syncCards();
    const directive = buildContextDirective(rawText);
    const active = buildActiveThreadsBlock();
    advanceThreadDormancy(); // dormancy progression only; injects nothing
    syncCards();
    // Thought system (independent of Life Cards): age the 💭 marker, then decide +
    // inject the thought ASK.
    expireThoughtMarker();
    const thoughtAsk = buildThoughtAsk(rawText);
    updateDebugCard();
    // Order matches the released build: the ACTIVE LIFE THREADS block is LAST so it is the
    // most salient thing the model reads (this is what makes Dynamic Large use the cards).
    // The thought ASK goes BEFORE it -- it only governs how the reply STARTS, so it does
    // not need to be last, and keeping it last was displacing the Life Cards.
    return String(rawText || "").trimEnd() + directive + thoughtAsk + active;
  }

  function handleInput(rawText) {
    ensureState();
    return stripBlocks(rawText);
  }

  // Load the editable config card into a runtime object the whole engine reads.
  // Safe defaults are used if the card is missing or unreadable.
  let LC = {
    protagonist: "",
    characters: [],
    rosterCharacters: [],
    rosterSource: "none",
    pressures: CFG.DEFAULT_PRESSURES.slice(),
    worldEvents: [],
    activityOff: false,
    activityTurns: CFG.DEFAULT_ACTIVITY_TURNS,
    legacyActivityUsed: false,
    targetCooldown: CFG.DEFAULT_TARGET_COOLDOWN,
    maxActive: CFG.MAX_ACTIVE_LIFE_CARDS,
    sceneRelevanceMode: CFG.SCENE_RELEVANCE_MODE,
    triggerOnTarget: CFG.TRIGGER_ON_TARGET,
    forceActiveCardTrigger: CFG.FORCE_ACTIVE_CARD_TRIGGER,
    protagonistAlwaysPresent: CFG.PROTAGONIST_ALWAYS_PRESENT,
    protagonistInvolvement: CFG.PROTAGONIST_INVOLVEMENT,
    relationships: {},
    relationshipNotes: [],
    relationshipCount: 0,
    relationshipsCardFound: false,
    mainConfigHadRelationships: false,
    rosterActorCount: 0,
    relActorCount: 0,
    checkpointEvery: CFG.CHECKPOINT_EVERY,
    threadReminderEvery: CFG.THREAD_REMINDER_EVERY
  };
  try { LC = buildRuntimeConfig(); } catch (e) {}

  const text = getGlobalText("");
  let result = text;
  try {
    if (hook === "context") result = handleContext(text);
    else if (hook === "output") result = handleOutput(text);
    else if (hook === "input") result = handleInput(text);
    else result = text;
  } catch (e) {
    // A thrown hook must not collapse to "" or a bare space (both read as an empty
    // response in AID). Start from the original text; finalizeResult below guarantees a
    // safe non-empty return and re-strips private tags for the output hook.
    debugFallback("catch:" + hook, (e && e.message) ? e.message : String(e));
    result = text;
  }
  // FINAL OUTPUT BOUNDARY: every hook returns through here, so no path can hand AID an
  // empty / whitespace-only / null / undefined value. Logs which branch fell back.
  result = finalizeResult(hook, result, text);
  setGlobalText(result);
  return result;
}

// Backward-compatibility alias. Existing adventures whose Context/Output/Input
// modifier tabs call ChaosGoblinV2(...) keep working without any edits.
function ChaosGoblinV2(hook, hookText) {
  return LivingCharacters(hook, hookText);
}