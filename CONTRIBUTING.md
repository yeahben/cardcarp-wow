# Contributing

Thank you for helping maintain this data!

Before jumping in, please review these community resources:

*   **[Join Discord](https://chat.yeahben.com):** Discuss structure and data accuracy.

## 🌭 Local Setup

Before submitting a pull-request, please test your changes locally to ensure they compile correctly.

1. Clone repository.
2. Install required Python dependencies: `pip install .`
3. Run the compile script to verify your YAML syntax and schema compliance: `cardcarp-compile`

## 🪲 Automated Validation

Every pull-request is automatically validated against the schema via GitHub Actions. If the pipeline fails, your pull-request cannot be merged. You can view the exact line causing the error by clicking "Details" on the failed GitHub-Action check.

The build also checks that every cross-reference resolves. A card pointing at an blueprint that does not exist, or a format banning a card by an ID nothing uses, fails the build and names the offender. Schema validation cannot catch this, because it only ever sees one file at a time.

<br>

## 🔑 How IDs Work

**Never write an ID into a YAML file.** Every ID is derived from the record's own properties at build time, and cross-references are written as human-readable names.

| Record | ID is derived from | Example |
| --- | --- | --- |
| Collection | `name` | `Azeroth` → `azeroth` |
| Set | `name` | `Heroes of Azeroth` → `heroes-of-azeroth` |
| Card | `set` + `index` | `Heroes of Azeroth` + `185` → `heroes-of-azeroth-185` |
| Blueprint | `name` + `origin` | `Galway Steamwhistle` + `heroes-of-azeroth-185` → `galway-steamwhistle-heroes-of-azeroth-185` |
| Deck | `collection` + `name` | `2013` + `Alliance Rogue` → `2013-alliance-rogue` |

So a card references its set as `set: Heroes of Azeroth`, not `set: heroes-of-azeroth`. The build kebab-cases it for you.

**`blueprint` is the one reference written in kebab-case**, because an blueprint has no single human name that identifies it — `blueprint: galway-steamwhistle-heroes-of-azeroth-185`.

Filenames exist purely so editors can find things. Renaming a file changes nothing about the data. If two records derive the same ID the build **fails and names both files**, rather than silently overwriting one.

### `origin` on blueprints

Names are not unique — six different cards are called "Deathwing the Destroyer". `origin` anchors an blueprint to the printing it first appeared in (`aspects-002`), so the ID stays unique. It is anchored to the *earliest* printing so that adding reprints never changes an existing blueprint's ID.

### Blueprint IDs vs card IDs

The two shapes answer different questions. An **blueprint** identifies a set of mechanics shared by every printing of a card — what it does. A **card** identifies one specific printing — its artist, rarity, set and collector number.

Card IDs are not merely internal plumbing: deck lists reference them, because a decklist has to name the printing that shipped in the box rather than just the mechanics.

<br>

## 📇 `index`

`index` is the printed collector number on a card, and the ordering label on a set or collection.

On **cards** it is required and zero-padded (`'001'`, `'185'`). On **sets and collections it is deliberately sparse.** Main-line releases are numbered `01`, `02`, `03`; their companion products take a letter suffix (`01a` for the tokens belonging to set `01`); and supplementary products — crafting sets, badge sets, holiday promos — are left unnumbered on purpose.

An unnumbered set is not missing data. Its folder simply carries no prefix:

```
data/card/standard/01-azeroth/
    01-heroes-of-azeroth/
    01a-heroes-of-azeroth-tokens/
    02-through-the-dark-portal/
    azeroth-badges-of-justice/     <- outside the numbered sequence
    azeroth-crafting/
```

<br>

## 🔍 Transcribing a Card

The printed card is the source of truth. Every set was read from scans, and a set file's `verified` date records when. If a record disagrees with the card in your hand, the card wins; if it disagrees with a later ruling or reworded reprint, it is not an error.

### Card or blueprint?

| On the card file (one printing) | On the blueprint (shared by reprints) |
| --- | --- |
| `index`, `rarity`, `artist`, `flavor`, `variation`, `set` | name, type line, cost, stats, class, faction and `rule` |

A reprint with different wording (added reminder text, "Talent" in place of "Hero Required") keeps pointing at the original blueprint, which holds the **first printing's** text. A same-name card that plays differently gets a blueprint of its own.

**Rarity** is read from the colour of the collector number: white Common, green Uncommon, blue Rare, purple Epic. Tokens have no rarity. **Artist** is the front face's "Art by" credit, with printed misspellings corrected to the artist's usual spelling.

### The type line

`Instant Ability — Fire Talent`, `Ally — Orc Warrior, Unique`, `Equipment — Armor — Plate, Chest (1)`:

| Field | Holds | Example |
| --- | --- | --- |
| `category` | The card type | `[Ally]`, `[Equipment]`, `[Ability, Ally]`, `[Master Hero]` |
| `type` | A printed prefix before the card type, and nothing else | `Arena`, `Scourge`, `Monster`, `Dungeon`, `Boss`, `Naxxramas` |
| `instant` | `true` when the line reads "Instant" or carries the gold Instant tab. The silver "Basic" tab is its opposite and is not recorded | |
| `race`, `class`, `talent`, `profession` | Those words, wherever they are printed. `class` also covers the lit class icons on abilities and equipment | `race: Orc`, `talent: [Fire]` |
| `subtype` | Every other word of the type line, in printed order. Multi-word creature types stay whole | `[Armor, Plate]`, `[Weapon, Two-Handed, Staff]`, `[Talent]`, `[Core Hound]` |
| `limit` | Tags at the right of the bar, as `{name, count}` | `Chest (1)` → `{name: Chest, count: 1}`; `Unique` → `{name: Unique}` |
| `stage` | The number in "Stage 2", "Boss 3" | |
| `sequence` | The number in "Event 21" on raid event cards | |
| `trait` | Anything else printed on the bar | `5 XP`, `Gehennas: 2`, `Traitor` affiliations |

`faction` is set only when a faction banner is printed. Cards showing both banners, and Monster cards, have none.

### Stats

| Field | Printed as |
| --- | --- |
| `cost` | The number in the top-left circle, kept as a string (`'3'`, `'X'`) |
| `damage` + `combat` | The ATK number and the damage type shown by its icon. Go by the icon, not the card's class: many hunters print the Melee blade |
| `health` | The red blood-drop |
| `defense` | The shield on armor |
| `strike` | The cost circle at the bottom right of a weapon |
| `capture` | The tower icon on a location |

### Rules text

`rule` has one entry per printed paragraph, top to bottom, each `{type: Ability, text: …}`. Italic reminder text stays in its parentheses inside its paragraph. A bold power name printed on its own line is its own entry. Flavor text is not a rule; it goes on the card file.

On double-sided cards each entry also carries `position: Front` or `position: Reverse`.

The printed arrow is written `→`. Icons printed inside the text are brace tokens:

| Token | Icon |
| --- | --- |
| `{1}` `{3}` `{x}` | A cost circle with that number or X |
| `{activate}` | The exhaust icon (box with a curved arrow) |
| `{alliance}` `{horde}` | Faction icons |
| `{mage}` `{death knight}` … | Class icons, lower-case class name |
| `{melee}` `{ranged}` `{arcane}` `{fire}` `{frost}` `{holy}` `{nature}` `{shadow}` | Damage-type icons |
| `{health}` `{def}` `{strike}` | Blood-drop, armor shield, and the blank circle that stands for a weapon's strike cost |
| `{basic}` `{instant}` | The silver Basic tab and gold Instant tab |
| `{winter veil}` | The Feast of Winter Veil set icon |

Quote the text in YAML when it starts with a brace.

<br>

## 🧬 Curation Decisions

Where a judgement call was made about the data, it is recorded here. **Please do not "correct" these back without raising it on Discord first.**

### Punctuation in names

381 blueprint names once carried an underscore where an apostrophe, quote, colon or question mark belonged — an artefact of an earlier import. These have been restored:

```
A_dal                      ->  A'dal
_A Plague upon Thee!_      ->  "A Plague upon Thee!"
Darkmoon Card_ Hurricane   ->  Darkmoon Card: Hurricane
Why Do We Fight_           ->  Why Do We Fight?
_Backstab_ Bindo Gearbomb  ->  "Backstab" Bindo Gearbomb
```

Names with a bare `-s` where a possessive apostrophe belonged (`A'dals Signet`) were corrected to the printed name during the scan check. Apostrophes are stripped when deriving IDs, so such a fix does **not** change any ID.

### Printed names win

A blueprint's `name` is the name printed on the card, including quote marks and capitals (`"I AM THE LAWGIVER!"`). Where a card prints no name (the blank Arena Grand Melee cards), a descriptive name is kept.

### Cards that are not game pieces

Art-only inserts (Tiny Snowman, Spirit of Competition) are `category: [Token]` with no rules. Raid cards with no type line, and the 2006 holiday greeting card, are `category: [Misc]`.

### Known gaps

- Reprint-only wording is not stored (see [Card or blueprint?](#card-or-blueprint)). Spire of Sunset costs 2 in Wrathgate and prints 3 in the 2011 Horde Mage starter; the blueprint keeps 2.

### Format legality

A card is legal in a format if **either** its set is allowed **or** its regulation is — not both. Regulation marks are a newer mechanic, so older cards qualify by set alone and newer ones by mark alone. An empty `set_allow` or `regulation_allow` means the format does not admit cards by that route at all.

Note that `regulation` is currently unpopulated on every blueprint, so any format gating purely on regulation matches nothing.

### Deck `collection`

A deck's `collection` field is a folder grouping (`2013`, `raid`, `champion`), **not** a reference to `data/collection`. The build deliberately does not check it.

<br>

## 🎴 What Counts as a Set

A set is a group of cards sharing a set-mark and its own numbering sequence — not a retail product. Tokens, crafting cards and badges ship alongside a main set but number independently, so each is its own set here, linked to the same collection.
