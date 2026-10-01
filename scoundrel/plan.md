# Scoundrel Tracker – Plan

A single-file (`index.html`) companion app for playing **Scoundrel** (the solo
rogue-like card game by Zach Gage & Kurt Bieg) with a *physical* deck of cards.
You deal real cards; the app tracks health, weapon, room state, the remaining
dungeon, and calculates everything for you.

---

## 1. The game (rules the app enforces)

### Deck – "the Dungeon" (44 cards)
Standard 52-card deck **minus** jokers, red face cards (J/Q/K ♥♦) and red aces.

| Suit | Cards | Role | Value |
|------|-------|------|-------|
| ♣ Clubs  | 2–10, J, Q, K, A (13) | **Monster** | 2–10, J=11, Q=12, K=13, A=14 |
| ♠ Spades | 2–10, J, Q, K, A (13) | **Monster** | same as above |
| ♦ Diamonds | 2–10 (9) | **Weapon** | face value |
| ♥ Hearts | 2–10 (9) | **Health potion** | face value |

Total monster value in the dungeon = 2 × (2+…+14) = **208**.

### Setup
* Health starts at **20** (and can never exceed 20).
* No weapon equipped.

### A turn ("a Room")
1. Deal cards face up until the room holds **4 cards** (the card left over from
   the previous room stays and counts as one of the 4).
2. **Run away (avoid the room)** – optional: scoop all 4 cards and put them at
   the bottom of the dungeon. You may **not** run from two rooms in a row, and
   you can only run *before* interacting with any card of the room.
3. Otherwise, **face 3 of the 4 cards**, one at a time, in any order.
   The 4th card stays and becomes part of the next room.

### Resolving a card
* **Weapon (♦)** – must be equipped. The previous weapon and every monster
  stacked on it are discarded.
* **Health potion (♥)** – heal by its value (capped at 20). Only **one potion
  per room** has an effect; a second potion taken in the same room is simply
  discarded.
* **Monster (♣♠)** – choose how to fight:
  * **Barehanded** – take damage equal to the monster's value.
  * **With weapon** – take `max(0, monster − weapon)` damage. The monster is
    stacked on the weapon.
  * Weapon degradation: once a weapon has slain a monster, it can only be used
    on monsters with a value **strictly lower** than the *last* monster it
    slew. Otherwise you must fight barehanded.

### End of game & scoring
* **Death** (health ≤ 0): score = −(sum of values of all monsters not yet slain,
  i.e. still in the dungeon / room).
* **Survived** (whole dungeon cleared): score = remaining health. Special case:
  if health is 20 **and** the last card resolved was a potion, score = 20 + that
  potion's value.

### End of the dungeon
When the dungeon runs out, rooms simply have fewer cards. A room is complete
after 3 cards are resolved; if only 1 card remains and the dungeon is empty, it
forms a final 1-card room that must be resolved to win.

---

## 2. How the app works

### Core idea
The app never shuffles or deals – **you** do. For each physical card you deal,
you tap an empty slot and pick that card from a **deck picker**. The app knows
which 44 cards exist and which have already been seen, so only valid,
not-yet-used cards are selectable (fast and mistake-proof).

### The deck picker (suit, then card)
A bottom sheet in two steps, both on screen at once:

1. **Suit.** Four small decks, each drawn as three fanned cards marked with the
   suit's kanji. They sit in two groups: **Black** (clubs and spades, the
   monsters) and **Red** (hearts and diamonds, the potions and weapons). Each
   deck shows how many of its cards are left. Tapping a group head picks the
   color first and dims the other color; if only one suit of that color still
   has cards, it is picked for you.
2. **Card.** Picking a deck fans its cards out below, like a hand of cards:
   2–10, J, Q, K, A for clubs and spades, 2–10 for hearts and diamonds. Cards
   already seen are printed without ink and can't be tapped. Pointing at a card
   (or focusing it) names it under the fan, for example
   "Queen of Clubs · 妖狐 Yōko · Monster 12".

Tapping a card commits it and opens the next empty slot, so dealing a full room
is about 8 taps. To fix a mis-pick, the picker opens on that card's deck with
the card lifted and ringed in red. Until you face a card in the room, a
**Remove** button also clears the slot.

Keyboard shortcuts on desktop:
`c s h d` = suit, `b r` = color, `2`–`9`, `0`/`t` = 10, `j q k a`, `Esc` closes.

### Resolving cards
Tap a dealt card → an action sheet shows the card, its name and the
*pre-calculated* outcome of every option:

* Monster: **Fight with 7♦ (−2 HP)** / **Fight barehanded (−9 HP)**.
  The weapon option is disabled, with the reason, when the weapon is too worn.
* Weapon: **Equip 9♦** (discards 5♦ and the 2 monsters on it).
* Potion: **Drink 6♥ (+6 HP)**, or **Discard potion** if one already worked
  this room.
* Always: **Change card** (fix a mis-pick) while unresolved.

The best option carries a small **Suggested** tag. Each card in the room also
shows its role, value and a preview of the outcome underneath (`−9`, `−2` with
a sword icon, `+6`, `equip`), so you can plan the room at a glance.

### Room flow / main controls
* **Run away** – enabled only when the room is full, untouched and you didn't
  run last room. Cards go back to the dungeon (they become pickable again).
* **Next room** – appears once 3 cards are resolved; the leftover card is
  carried into slot 1 of the new room.
* **Undo** – every action is undoable (snapshot history).
* **New game** – with confirmation.

### Extra helpers
* **Dungeon tracker** – count of cards left, remaining monster total, potions
  and weapons left, and a mini grid of all 44 cards with seen cards dimmed
  (card counting!).
* **Log** – chronological list of what happened each room.
* **Auto-save** – state persisted to `localStorage`; reload-safe.
* **Game-over screen** – win/lose, score, rooms cleared, monsters slain.

---

## 3. State model

```js
state = {
  hp: 20,
  weapon: null | { card, slain: [card, ...] },
  room: [ { card, resolved: false } | null, x4 ],  // slots
  resolvedInRoom: 0,
  potionUsedInRoom: false,
  ranLastRoom: false,
  used: [cardId, ...],       // resolved / discarded cards (out of the dungeon)
  roomNo: 1,
  lastResolved: cardId|null,
  log: [ { room, text, kind } ],
  over: null | { won, score }
}
history = [ snapshot, ... ]  // for undo
```

Derived values:
* `inRoom` = cards currently in room slots.
* `dungeonLeft` = 44 − used − inRoom (cards still in the physical deck).
* `slotsNeeded` = min(4, inRoom + dungeonLeft) (room size to deal to).
* `weaponUsableOn(m)` = weapon && (no slain || m.value < lastSlain.value).

Card ids: `C2…C14`, `S2…S14`, `H2…H10`, `D2…D10`.

---

## 4. Wireframes

### Main screen (mobile-first, scales up to desktop)

```
┌───────────────────────────────────────────────┐
│ ROOM 03                    Undo  Rules  New   │  meta row: room, undo, rules, new game
│                                               │
│ Scoundrel [seal]                              │  wordmark with the 悪党 seal
│ A companion for playing Scoundrel with a      │
│ real deck. You deal, it keeps count.          │
├───────────────────────────────────────────────┤
│ HEALTH        WEAPON          DUNGEON         │  three stats, numbers in Martian Mono
│ 14 / 20       [7♦][Q♠][9♣]    27  143  5  4   │
│ ||||||||···   below 9         left pts ♥  ♦   │
├───────────────────────────────────────────────┤
│ Room 3       FACED 1 of 3  POTION Ready       │
│ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐       │
│ │K♣     │ │5♥     │ │       │ │       │       │  embossed cards, kanji name in the middle
│ │  ▓▓   │ │  ▓▓   │ │   +   │ │   +   │       │  tap + to deal, tap a card to face it
│ │     K♣│ │     5♥│ │ Deal  │ │ Deal  │       │
│ └───────┘ └───────┘ └───────┘ └───────┘       │
│ 13   −6 ⚔ 5      +5                           │  value and a preview of the outcome
│                                               │
│ Deal 2 more cards. Tap an empty slot.         │
│              [ Run away ] [ Next room → ]     │
├───────────────────────────────────────────────┤
│ The deck        (44 grid, seen cards dim)   + │  folds
│ Log                                           │
└───────────────────────────────────────────────┘
```

### Deck picker (bottom sheet)

```
┌───────────────────────────────────────────────┐
│ DEAL                                      ✕   │
│ Card 3 of 4                [K][5][ ][ ]       │  progress: cards dealt so far
├───────────────────────────────────────────────┤
│ 01 SUIT                                       │
│ ● BLACK  Monsters     ● RED  Potions, weapons │  tap a group head to pick the color first
│   ▞▮▚      ▞▮▚          ▞▮▚      ▞▮▚          │  each deck is three fanned cards
│  Clubs    Spades       Hearts  Diamonds       │
│  12 left  13 left      8 left   9 left        │
├───────────────────────────────────────────────┤
│ 02 CARD                               CLUBS   │
│         ▞ ▞ ▞ ▞ ▞ ▞ ▮ ▚ ▚  ▚ ▚ ▚ ▚            │  picking a deck fans its suit out
│         2 3 4 5 6 7 8 9 10 J Q K A            │  seen cards are printed without ink
│  Tap the card you dealt. Cards without ink    │  caption names the card under the pointer
│        are already out of play.               │
├───────────────────────────────────────────────┤
│ c s h d suit  b r color  2–9 0 j q k a card   │  keyboard hints, hidden on touch screens
└───────────────────────────────────────────────┘
```

### Card action sheet

```
┌───────────────────────────────────────────────┐
│ ┌──────┐                                 ✕    │
│ │Q     │  MONSTER · 12                        │
│ │  ▓▓  │  Queen of Spades                     │
│ │     Q│  HANNYA                              │  kanji name 般若 with its romaji
│ └──────┘                                      │
├───────────────────────────────────────────────┤
│ Fight with 7♦  SUGGESTED              −5 HP   │
│ Monster is stacked on your weapon             │
├───────────────────────────────────────────────┤
│ Fight barehanded                     −12 HP   │
│ Keeps your weapon fresh                       │
├───────────────────────────────────────────────┤
│ Change card                                   │
│ Fix a mis-pick                                │
└───────────────────────────────────────────────┘
```

### Game over

```
┌───────────────────────────────────────────────┐
│                    [seal]                     │  生還 (returned alive) or 討死 (died fighting)
│            You died in the dungeon            │
│                     −87                       │
│                    SCORE                      │
├───────────────────────────────────────────────┤
│  ROOMS  SLAIN  DAMAGE  HEALED  RAN            │
│    9     12      48      21    1×             │
├───────────────────────────────────────────────┤
│      [ Undo last move ]  [ New game ]         │
└───────────────────────────────────────────────┘
```

---

## 5. Design

Quiet and typographic: a warm paper page, a lot of whitespace, ink-black text
and one vermilion accent, the red of a Japanese name seal.

* **Type.**
  * Headings: TRJN DaVinci Italic (`fonts/TRJNDaVinci-Italic.ttf`).
  * Interface and body text: Die Grotesk C, Regular and Medium (`fonts/`).
  * Stats and numbers: Martian Mono (Google Fonts). Big numbers use its light,
    narrow cut.
  * Kanji: Shippori Mincho B1 ExtraBold (Google Fonts). None of the fonts above
    has Japanese characters, so the page asks Google Fonts for only the 75 it
    uses (about 25 KB).
* **Cards** are embossed paper. The rank, suit and name are pressed into the
  card and the face carries a raised inner panel. Cards out of play keep the
  emboss but lose their ink.
* **Every card has a name** in kanji, shown with its romaji. Each suit has a
  theme, and higher cards get fiercer (monsters) or stronger (weapons and
  potions). The suit decks carry one kanji each: 獣 beasts, 霊 spirits,
  刀 blades, 薬 remedies.
* **Seals.** 悪党 (scoundrel) by the wordmark. The game-over card shows 生還
  (returned alive) or 討死 (died fighting).
* **Motion.** Dealt cards slide in and a chosen deck fans out card by card.
  The system's reduced-motion setting turns this off.
* **Fit.** The picker never needs scrolling. On short screens its fan flattens
  a little and the cards shrink until the sheet fits. On wide screens the
  section labels move into a left rail. On phones the progress cards move up
  next to the eyebrow, and the keyboard hints are hidden.

| Rank | ♣ Beasts | ♠ Spirits | ♦ Blades | ♥ Remedies |
|------|----------|-----------|----------|------------|
| 2 | 鬼火 Onibi | 餓鬼 Gaki | 竹槍 Takeyari | 朝露 Asatsuyu |
| 3 | 河童 Kappa | 幽霊 Yūrei | 木刀 Bokutō | 薬草 Yakusō |
| 4 | 鎌鼬 Kamaitachi | 骸骨 Gaikotsu | 短刀 Tantō | 清水 Shimizu |
| 5 | 化猫 Bakeneko | 山姥 Yamanba | 鎖鎌 Kusarigama | 甘露 Kanro |
| 6 | 狒々 Hihi | 雪女 Yuki-onna | 脇差 Wakizashi | 神酒 Miki |
| 7 | 百足 Mukade | 怨霊 Onryō | 薙刀 Naginata | 良薬 Ryōyaku |
| 8 | 牛鬼 Ushi-oni | 鬼婆 Onibaba | 大弓 Daikyū | 秘薬 Hiyaku |
| 9 | 火車 Kasha | 夜叉 Yasha | 太刀 Tachi | 霊薬 Reiyaku |
| 10 | 雷獣 Raijū | 羅刹 Rasetsu | 名刀 Meitō | 仙丹 Sentan |
| J | 天狗 Tengu | 修羅 Shura | | |
| Q | 妖狐 Yōko | 般若 Hannya | | |
| K | 大蛇 Orochi | 閻魔 Enma | | |
| A | 龍王 Ryūō | 死神 Shinigami | | |

---

## 6. Implementation notes
* One self-contained `index.html`: inline CSS + vanilla JS, no libraries or
  build step.
* TRJN DaVinci and Die Grotesk load from the `fonts/` folder next to
  `index.html`. Martian Mono and the kanji face come from Google Fonts, so they
  need a network connection. Offline (or from `file://` without a network) the
  page falls back to system fonts and works the same.
* Pure functions for rules (`damage`, `canUseWeapon`, `score`) and a single
  `render()` that redraws from `state`.
* The look lives apart from the rules: `NAMES` (card names), `cardHTML` (the
  embossed card), `renderPicker` and `fanGeometry` / `layoutFan` (the deck
  picker). Restyling never touches the rule code.
