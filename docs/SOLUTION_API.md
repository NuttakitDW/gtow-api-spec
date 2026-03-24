# GTO Wizard Postflop API — Spot Solution

## Endpoint

```
GET https://api.{DOMAIN}.com/{version}/solutions/spot-solution/
```

## Authentication

| Header | Value | Description |
|--------|-------|-------------|
| `authorization` | `Bearer <JWT>` | Access token (short-lived, ~15 min) |
| `gwclientid` | UUID string | Client identifier |

---

## Query Parameters

| Parameter | Type | Required | Example | Description |
|-----------|------|----------|---------|-------------|
| `gametype` | string | Yes | `MTTGeneral` | Game format identifier |
| `depth` | string | Yes | `20.125` | Starting stack depth in big blinds |
| `stacks` | string | No | _(empty)_ | Custom stack sizes (omit for equal stacks) |
| `preflop_actions` | string | Yes | `F-F-F-F-F-F-R2-F-C` | Full preflop action sequence |
| `flop_actions` | string | Yes | _(empty)_ | Flop action sequence (empty string = root node) |
| `turn_actions` | string | Yes | _(empty)_ | Turn action sequence |
| `river_actions` | string | Yes | _(empty)_ | River action sequence |
| `board` | string | Yes | `AcTd6h` | Board cards (6 chars flop, 8 turn, 10 river) |

### Action Notation

Actions are separated by `-`:

| Code | Meaning | Example |
|------|---------|---------|
| `F` | Fold | `F` |
| `X` | Check | `X` |
| `C` | Call | `C` |
| `R{size}` | Raise (size in BB or multiplier) | `R2`, `R1.8` |
| `RAI` | All-in | `RAI` |
| `B{frac}` | Bet (fraction of pot) | `B0.5` |

### Preflop Action Sequence

The `preflop_actions` field encodes every seat's action in order. For example in a 9-handed MTT:

```
F-F-F-F-F-F-R2-F-C
│ │ │ │ │ │ │   │ └─ BB calls
│ │ │ │ │ │ │   └─── SB folds
│ │ │ │ │ │ └─────── BTN raises to 2 BB
│ │ │ │ │ └───────── CO folds
│ │ │ │ └─────────── HJ folds
│ │ │ └───────────── LJ folds
│ │ └─────────────── UTG+2 folds
│ └─────────────────  UTG+1 folds
└───────────────────  UTG folds
```

---

## Response

### Top-Level Structure

```json
{
  "action_solutions": [...],
  "players_info": [...],
  "hand_categories_range": [...],
  "draw_categories_range": [...],
  "blocker_rate": [...],
  "unblocker_rate": [...],
  "blockers_frequencies": [...],
  "game": {...},
  "warning": null,
  "hands_locked": null
}
```

| Field | Type | Description |
|-------|------|-------------|
| `action_solutions` | ActionSolution[] | Strategy data for each available action |
| `players_info` | PlayerInfo[] | Both active players with preflop ranges |
| `hand_categories_range` | int[1326] | Hand category per combo (`-1` = not in range) |
| `draw_categories_range` | int[1326] | Draw category per combo (`-1` = not in range) |
| `blocker_rate` | float[1326] | Blocker effect per combo (`-1` = not in range) |
| `unblocker_rate` | float[1326] | Unblocker effect per combo (`-1` = not in range) |
| `blockers_frequencies` | BlockerFrequency[] | Per-card blocker frequencies for each action |
| `game` | Game | Full game state |
| `warning` | string \| null | Warning message if applicable |
| `hands_locked` | any \| null | Paywall/access lock indicator |

---

### ActionSolution

Each entry represents one available action at the current decision node.

```json
{
  "action": {
    "code": "X",
    "position": "BB",
    "type": "CHECK",
    "betsize": "0",
    "allin": false,
    "is_hand_end": false,
    "is_showdown": false,
    "next_street": false,
    "display_name": "CHECK",
    "simple_group": "CHECK",
    "advanced_group": "CHECK",
    "betsize_by_pot": null,
    "next_position": "BTN"
  },
  "total_frequency": 0.9999958844696382,
  "strategy": [0.0, 0.0, 1.0, ...],
  "evs": [0.0, 0.0, 5.2, ...]
}
```

#### ActionDef Fields

| Field | Type | Description |
|-------|------|-------------|
| `code` | string | Action code: `X`, `R1.8`, `RAI`, `B0.5`, `C`, `F` |
| `position` | string | Position of the acting player (`BB`, `BTN`, `SB`, etc.) |
| `type` | string | Action type enum (see below) |
| `betsize` | string | Bet/raise size in big blinds (`"0"` for check/fold) |
| `allin` | boolean | Whether this action is all-in |
| `is_hand_end` | boolean | `true` if this action ends the hand (e.g., fold) |
| `is_showdown` | boolean | `true` if this action leads to showdown |
| `next_street` | boolean | `true` if this action advances to the next street |
| `display_name` | string | Human-readable label (`CHECK`, `BET`, `ALLIN`, `CALL`, `FOLD`) |
| `simple_group` | string | Simple grouping for UI (`CHECK`, `RAISE`, `CALL`, `FOLD`) |
| `advanced_group` | string | Detailed grouping (`CHECK`, `BET_SMALL`, `BET_OVERBET`, etc.) |
| `betsize_by_pot` | float \| null | Bet size as fraction of pot (`null` for check/fold/call) |
| `next_position` | string | Position of the next player to act |

#### Action Type Enum

| Value | Description |
|-------|-------------|
| `CHECK` | Check |
| `FOLD` | Fold |
| `CALL` | Call |
| `BET` | Bet (first aggression on a street) |
| `RAISE` | Raise (note: BB's first bet on flop is typed as `RAISE`) |
| `ALLIN` | All-in |

#### ActionSolution Fields

| Field | Type | Description |
|-------|------|-------------|
| `action` | ActionDef | Action metadata |
| `total_frequency` | float | Weighted average frequency across all in-range hands [0.0, 1.0] |
| `strategy` | float[1326] | Probability of taking this action for each hand combo [0.0, 1.0] |
| `evs` | float[1326] | Expected value in chips for each hand combo |

---

### PlayerInfo

One entry per active (non-folded) player.

```json
{
  "player": {
    "relative_postflop_position": "IP",
    "hand": null,
    "is_dealer": true,
    "is_folded": false,
    "is_hero": false,
    "is_active": false,
    "stack": "20.125",
    "current_stack": "18.000",
    "chips_on_table": "0",
    "bounty": null,
    "profile": null,
    "position": "BTN",
    "bounty_in_bb": null,
    "name": "B",
    "seat": 5
  },
  "range": [0.0, 0.0, ..., 0.42, ...]
}
```

#### Player Fields

| Field | Type | Description |
|-------|------|-------------|
| `relative_postflop_position` | string | `"OOP"` or `"IP"` |
| `hand` | string \| null | Specific hand (null in solution mode) |
| `is_dealer` | boolean | Whether this player is the dealer |
| `is_folded` | boolean | Whether this player folded preflop |
| `is_hero` | boolean | Whether this is the hero (active decision maker) |
| `is_active` | boolean | Whether it's this player's turn to act |
| `stack` | string | Starting stack in BB |
| `current_stack` | string | Remaining stack (after preflop betting) |
| `chips_on_table` | string | Chips currently committed on this street |
| `bounty` | string \| null | Bounty amount (MTT bounty formats) |
| `profile` | string \| null | Player profile identifier |
| `position` | string | Table position (`BB`, `BTN`, `SB`, `UTG`, etc.) |
| `bounty_in_bb` | string \| null | Bounty expressed in big blinds |
| `name` | string | Short display name (`B`, `B2`, etc.) |
| `seat` | int | Seat number |

#### Range Array

| Field | Type | Description |
|-------|------|-------------|
| `range` | float[1326] | Preflop range frequency for each hand combo [0.0, 1.0]. `0.0` = hand not in range |

---

### Game

Full game state at the current decision point.

```json
{
  "players": [...],
  "current_street": {
    "type": "FLOP",
    "start_pot": "5.500",
    "end_pot": "5.500"
  },
  "pot": "5.500",
  "pot_odds": null,
  "active_position": "BB",
  "board": "AcTd6h",
  "bet_display_name": "BET"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `players` | Player[] | All seats at the table (including folded players) |
| `current_street` | StreetInfo | Current street metadata |
| `pot` | string | Total pot size in BB |
| `pot_odds` | string \| null | Pot odds for the active player (when facing a bet) |
| `active_position` | string | Position of the player to act |
| `board` | string | Board cards |
| `bet_display_name` | string | How the first aggression is labeled (`"BET"`) |

#### StreetInfo

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Street enum: `FLOP`, `TURN`, `RIVER` |
| `start_pot` | string | Pot size at the start of this street |
| `end_pot` | string | Pot size at the end of this street (after current actions) |

#### Game Players vs PlayerInfo Players

- `game.players` — All 9 seats including folded players. No `range` array. No `name` or `seat` fields.
- `players_info` — Only the 2 active (non-folded) postflop players. Includes `range[1326]`, `name`, and `seat`.

---

### BlockerFrequency

Per-card blocker data for the 49 remaining cards (52 minus 3 board cards).

```json
{
  "card": "2s",
  "actions": [
    {
      "action": "X",
      "frequency": "-1.1621343032341969E-7"
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `card` | string | Card name (e.g., `"2s"`, `"Ah"`) |
| `actions` | BlockerAction[] | Frequency shift per action when holding this card |

#### BlockerAction

| Field | Type | Description |
|-------|------|-------------|
| `action` | string | Action code (matches `action_solutions[].action.code`) |
| `frequency` | string | Frequency delta (how much holding this card shifts the action frequency) |

---

## Hand Combo Indexing

All 1326-element arrays use the same indexing scheme for the 1326 unique 2-card hand combinations.

### Card Encoding (0–51)

Cards are numbered: `card_index = rank * 4 + suit`

| Rank | Clubs (0) | Diamonds (1) | Hearts (2) | Spades (3) |
|------|-----------|--------------|------------|------------|
| 2    | 0         | 1            | 2          | 3          |
| 3    | 4         | 5            | 6          | 7          |
| 4    | 8         | 9            | 10         | 11         |
| 5    | 12        | 13           | 14         | 15         |
| 6    | 16        | 17           | 18         | 19         |
| 7    | 20        | 21           | 22         | 23         |
| 8    | 24        | 25           | 26         | 27         |
| 9    | 28        | 29           | 30         | 31         |
| T    | 32        | 33           | 34         | 35         |
| J    | 36        | 37           | 38         | 39         |
| Q    | 40        | 41           | 42         | 43         |
| K    | 44        | 45           | 46         | 47         |
| A    | 48        | 49           | 50         | 51         |

### Combo Enumeration

```
index = 0
for c1 in 0..52:
    for c2 in (c1+1)..52:
        combos[index] = (c1, c2)
        index += 1
```

### Index Formula

```
index = c1 * (103 - c1) / 2 + (c2 - c1 - 1)
```

where `c1 < c2`

### Examples

| Hand | c1 | c2 | Index |
|------|----|----|-------|
| 2c2d | 0 | 1 | 0 |
| 2c2h | 0 | 2 | 1 |
| AhAs | 50 | 51 | 1325 |

### Board Card Handling

Hands containing board cards have `0.0` in all arrays (strategy, EVs, range). For board `AcTd6h`, any combo containing card index 48 (Ac), 33 (Td), or 18 (6h) will be zeroed out.

---

## Response Size

| Component | Approximate Size |
|-----------|-----------------|
| Per action (strategy + evs) | ~21 KB |
| 3 actions typical | ~63 KB |
| 2 player ranges | ~21 KB |
| Categories + blockers | ~50 KB |
| Game metadata | ~3 KB |
| **Total (uncompressed)** | **~960 KB** |

---

## Example

### Request

```
GET /{version}/solutions/spot-solution/?gametype=MTTGeneral&depth=20.125&stacks=&preflop_actions=F-F-F-F-F-F-R2-F-C&flop_actions=&turn_actions=&river_actions=&board=AcTd6h
```

**Scenario:** 9-handed MTT, 20bb effective. BTN opens to 2bb, BB calls. Flop is Ac Td 6h. BB (OOP) is first to act.

### Response Summary

3 available actions for BB:

| Action | Code | Type | Bet Size (BB) | Pot Fraction | Total Frequency |
|--------|------|------|---------------|--------------|-----------------|
| Check | `X` | CHECK | 0 | — | 99.99% |
| Bet 33% pot | `R1.8` | RAISE | 1.8 | 0.327 | 0.0004% |
| All-in | `RAI` | RAISE | 18.0 | 3.273 | 0.00% |

**Note:** On this board at this stack depth, the GTO solution is almost pure check from BB.
