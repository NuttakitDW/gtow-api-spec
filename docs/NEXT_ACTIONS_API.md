# GTO Wizard Postflop API — Next Actions

Lightweight navigation endpoint. Returns available actions at the next decision node without strategy data.

## Endpoint

```
GET https://api.{DOMAIN}.com/{version}/poker/next-actions/
```

## Authentication

| Header | Value | Description |
|--------|-------|-------------|
| `authorization` | `Bearer <JWT>` | Access token (short-lived, ~15 min) |
| `gwclientid` | UUID string | Client identifier |

---

## Query Parameters

Same parameters as the spot-solution endpoint.

| Parameter | Type | Required | Example | Description |
|-----------|------|----------|---------|-------------|
| `gametype` | string | Yes | `MTTGeneral` | Game format identifier |
| `depth` | string | Yes | `20.125` | Starting stack depth in big blinds |
| `stacks` | string | No | _(empty)_ | Custom stack sizes (omit for equal stacks) |
| `preflop_actions` | string | Yes | `F-F-F-F-F-F-R2-F-C` | Full preflop action sequence |
| `flop_actions` | string | Yes | `X` | Flop action sequence |
| `turn_actions` | string | Yes | _(empty)_ | Turn action sequence |
| `river_actions` | string | Yes | _(empty)_ | River action sequence |
| `board` | string | Yes | `AcTd6h` | Board cards |

**Key difference from spot-solution:** The action parameters here represent the action just taken (e.g., `flop_actions=X` means "OOP just checked, what's next?"), whereas spot-solution uses them to identify the current decision node.

---

## Response

### Top-Level Structure

```json
{
  "next_actions": {...},
  "future_actions": [...]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `next_actions` | NextActionsNode | The immediate next decision point |
| `future_actions` | NextActionsNode[] | Lookahead nodes for street transitions (see below) |

---

### NextActionsNode

Represents a decision point in the game tree.

```json
{
  "game": {...},
  "available_actions": [...],
  "custom_solution_id": null,
  "is_node_locked": false,
  "is_edited": false,
  "forced_fold": false,
  "available_node_edits": null,
  "merged_actions": [],
  "preset_action_code": "X"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `game` | Game | Game state at this node (same structure as spot-solution) |
| `available_actions` | AvailableAction[] | Actions available at this node |
| `custom_solution_id` | string \| null | Custom solution override ID |
| `is_node_locked` | boolean | Whether this node is locked behind paywall |
| `is_edited` | boolean | Whether this node has been manually edited |
| `forced_fold` | boolean | Whether a fold is forced at this node |
| `available_node_edits` | any \| null | Available node edit options |
| `merged_actions` | string[] | List of action codes that have been merged |
| `preset_action_code` | string \| null | Default action hint for UI selection (`null` at terminal nodes) |

---

### AvailableAction

Each available action at the node. Similar to spot-solution's `ActionSolution` but without strategy/EV arrays.

```json
{
  "action": {
    "code": "X",
    "position": "BTN",
    "type": "CHECK",
    "betsize": "0.000",
    "allin": false,
    "is_hand_end": false,
    "is_showdown": false,
    "next_street": true,
    "display_name": "CHECK",
    "simple_group": "CHECK",
    "advanced_group": "CHECK",
    "betsize_by_pot": null,
    "next_position": "BB"
  },
  "frequency": null,
  "is_solution_end": false,
  "can_be_solved_by_ai": false,
  "next_position": "BB",
  "selected": false
}
```

#### ActionDef Fields

Same schema as spot-solution. See [API_SPEC.md](API_SPEC.md#actiondef-fields) for full reference.

#### Wrapper Fields

| Field | Type | Description |
|-------|------|-------------|
| `action` | ActionDef | Action metadata (same as spot-solution) |
| `frequency` | float \| null | Overall frequency (`null` — only spot-solution provides this) |
| `is_solution_end` | boolean | Whether taking this action reaches a terminal node |
| `can_be_solved_by_ai` | boolean | Whether AI solver is available for the resulting node |
| `next_position` | string | Position of the next player to act |
| `selected` | boolean | Whether this action is currently selected in the UI |

---

## Game Object

Same structure as spot-solution's `game` field. Contains all seats, current street info, pot, and active position.

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
  "active_position": "BTN",
  "board": "AcTd6h",
  "bet_display_name": "BET"
}
```

See [API_SPEC.md](API_SPEC.md#game) for full field reference.

---

## future_actions

The `future_actions` array provides **lookahead** for street transitions. It is populated when both players check and the game advances to the next street, allowing the UI to pre-render the next node's action buttons.

### When it appears

| Scenario | future_actions |
|----------|---------------|
| OOP checks (IP still to act on same street) | `[]` empty |
| IP checks (street transition: flop→turn, turn→river) | `[NextActionsNode]` with 1 item |
| River check-check (showdown) | `[]` empty |
| Any non-check action | `[]` empty |

### Example: Flop X-X triggers turn lookahead

**Request:** `flop_actions=X-X` (both checked on flop)

**Response:**
```json
{
  "next_actions": {
    "game": { "active_position": "BB", "current_street": { "type": "TURN" } },
    "available_actions": [/* BB's turn actions */],
    "preset_action_code": "X"
  },
  "future_actions": [
    {
      "game": { "active_position": "BTN", "current_street": { "type": "TURN" } },
      "available_actions": [/* BTN's turn actions (lookahead) */],
      "preset_action_code": "X"
    }
  ]
}
```

The `future_actions[0]` here shows BTN's available actions on the turn — pre-fetched so the UI can display them without an additional API call if BB checks again.

---

## Terminal Node (Showdown)

When the hand is over (e.g., river check-check), the response indicates a terminal state:

```json
{
  "next_actions": {
    "game": { "active_position": "BB", "board": "AcTd6hThAh" },
    "available_actions": [],
    "preset_action_code": null
  },
  "future_actions": []
}
```

**Terminal indicators:**
- `available_actions` is an empty array
- `preset_action_code` is `null`
- `future_actions` is empty

---

## Complete Check-Check Flow

Full sequence of next-actions calls for a check-check line from flop to showdown:

| Step | Query (flop/turn/river actions) | Board | Active | Actions Available | future_actions | preset |
|------|------|-------|--------|---|---|---|
| 1 | `flop_actions=X` | AcTd6h | BTN | 6 (X, R1.8, R3, R4.55, R6.9, RAI) | none | `X` |
| 2 | `flop_actions=X-X` | AcTd6h | BB | 7 (turn actions for BB) | 1 (turn actions for BTN) | `X` |
| 3 | `flop_actions=X-X, turn_actions=X` | AcTd6hTh | BTN | 7 (X, R1.8, R3, R4.55, R6.9, R11, RAI) | none | `X` |
| 4 | `flop_actions=X-X, turn_actions=X-X` | AcTd6hTh | BB | 7 (river actions for BB) | 1 (river actions for BTN) | `X` |
| 5 | `flop_actions=X-X, turn_actions=X-X, river_actions=X` | AcTd6hThAh | BTN | 7 (X, bets, RAI) | none | `X` |
| 6 | `flop_actions=X-X, turn_actions=X-X, river_actions=X-X` | AcTd6hThAh | BB | **0** (showdown) | none | `null` |

### Observations

- **Board grows** as streets advance: 6 chars (flop) → 8 (turn) → 10 (river)
- **future_actions appears only on street transitions** (steps 2, 4) — when IP checks to close a street
- **Action count varies**: BTN (IP) gets more bet sizing options than BB (OOP) on some streets
- **Turn introduces R11**: a new bet size not available on the flop
- **Step 6 is terminal**: no actions, null preset, hand goes to showdown
