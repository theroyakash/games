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
you tap an empty slot and pick the card with a **3-way picker**. The app knows
which 44 cards exist and which have already been seen, so only valid,
not-yet-used cards are selectable (fast and mistake-proof).

### The 3-way picker (Suit → Color → Rank)
A bottom sheet with three columns, all visible at once:

1. **Suit** – ♣ Clubs, ♠ Spades, ♥ Hearts, ♦ Diamonds (each labelled with its
   role: Monster / Potion / Weapon).
2. **Color** – Black / Red. Picking a suit auto-selects its color; picking a
   color first narrows the suit column to the matching suits (so you can go
   either Suit → Color or Color → Suit).
3. **Rank** – 2…10, J, Q, K, A. Ranks not in the dungeon (red J/Q/K/A) and
   cards already seen are disabled.

Tapping a rank commits the card and automatically jumps to the next empty slot,
so dealing a full room is ~8 taps. Keyboard shortcuts on desktop:
`c s h d` = suit, `b r` = color, `2`–`9`, `0`/`t` = 10, `j q k a`, `Esc` closes.

### Resolving cards
Tap a dealt card → a contextual action sheet shows the *pre-calculated*
outcome for every option:

* Monster: **⚔ Fight with weapon (−2 HP)** / **✊ Barehanded (−9 HP)**
  (weapon option disabled with the reason if the weapon is too worn).
* Weapon: **Equip (replaces 5♦ + 2 slain)**.
* Potion: **Drink (+6 HP)** or **Discard (potion already used this room)**.
* Always: **Change card** (fix a mis-pick) while unresolved.

Each card tile also shows a small preview badge (e.g. `−9` / `−2 ⚔` / `+6` /
`equip`) so you can plan the room at a glance.

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
┌─────────────────────────────────────────┐
│ SCOUNDREL            Room 3    ↶  ☰     │  header: undo, menu
├─────────────────────────────────────────┤
│ ❤ 14 / 20  ███████████████░░░░░░░       │  health bar
│                                         │
│ ⚔ WEAPON  [ 7♦ ]  slain: Q♠ 9♣          │  weapon + stack
│           usable on monsters < 9        │
│                                         │
│ 🂠 Dungeon 27 left · monsters 143 ·       │  dungeon summary
│   ♥ 5 left · ♦ 4 left                   │
├─────────────────────────────────────────┤
│  ROOM 3 · resolved 1/3 · potion unused  │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐    │
│  │  K♣  │ │  5♥  │ │  +   │ │  +   │    │  tap + → picker
│  │  13  │ │  +5  │ │ deal │ │ deal │    │  tap card → actions
│  │ −6 ⚔ │ │      │ │      │ │      │    │
│  └──────┘ └──────┘ └──────┘ └──────┘    │
│                                         │
│  [ 🏃 Run away ]        [ Next room → ] │
├─────────────────────────────────────────┤
│ ▸ Dungeon cards (44 grid, seen dimmed)  │
│ ▸ Log                                   │
└─────────────────────────────────────────┘
```

### 3-way picker (bottom sheet)

```
┌─────────────────────────────────────────┐
│ Deal card 3 of 4                     ✕  │
├──────────────┬──────────┬───────────────┤
│ 1 · SUIT     │ 2 · COLOR│ 3 · RANK      │
│ [♣ Clubs   ] │ [■ Black]│ [2][3][4][5]  │
│  Monster     │ [■ Red  ]│ [6][7][8][9]  │
│ [♠ Spades  ] │          │ [10][J][Q][K] │
│  Monster     │          │ [A]           │
│ [♥ Hearts  ] │          │ (used/invalid │
│  Potion      │          │  disabled)    │
│ [♦ Diamonds] │          │               │
│  Weapon      │          │               │
└──────────────┴──────────┴───────────────┘
```

### Card action sheet

```
┌─────────────────────────────────────────┐
│  Q♠  Monster · 12                    ✕  │
├─────────────────────────────────────────┤
│ [ ⚔ Fight with 7♦        take −5  ]     │
│ [ ✊ Fight barehanded     take −12 ]     │
│ [ ✎ Change card                   ]     │
└─────────────────────────────────────────┘
```

### Game over

```
┌─────────────────────────────────────────┐
│            ☠ You died  /  🏆 Escaped!    │
│                 Score: −87              │
│   Rooms: 9 · Monsters slain: 12         │
│        [ New game ]  [ Undo last ]      │
└─────────────────────────────────────────┘
```

---

## 5. Implementation notes
* One self-contained `index.html`: inline CSS + vanilla JS, no dependencies,
  works offline / from `file://`.
* Dark "dungeon" theme, large touch targets, red/black suit colors, responsive
  grid (2×2 room on narrow phones, 1×4 on wider screens).
* Pure functions for rules (`damage`, `canUseWeapon`, `score`) and a single
  `render()` that redraws from `state`.
