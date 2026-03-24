# GTO Wizard API Spec

Reverse-engineered API specification for the GTO Wizard postflop solver.

## Endpoints

GTO Wizard uses two endpoints to power the postflop solution UI:

### 1. Spot Solution — full strategy data

```
GET /{version}/solutions/spot-solution/
```

Returns the **complete GTO solution** at a decision node: strategy frequencies, expected values, ranges, blockers, and hand categories for all 1326 hand combinations. This is the heavy payload (~960KB) that powers the strategy grid and EV display.

### 2. Next Actions — lightweight navigation

```
GET /{version}/poker/next-actions/
```

Returns only the **available actions** at the next node in the game tree. No strategy arrays, no EVs — just the action list and game state metadata. Called when the user clicks an action to navigate forward.

### How they work together

```
User loads a spot
  └─► spot-solution  → full strategy data (render grid)

User clicks "Check"
  └─► next-actions   → what actions exist at the next node (render buttons)
  └─► spot-solution  → full strategy for the new node (render grid)
```

### Comparison

| | spot-solution | next-actions |
|---|---|---|
| **Purpose** | Full GTO strategy | Navigation / action list |
| **Response size** | ~960 KB | ~5 KB |
| **strategy[1326]** | Yes | No |
| **evs[1326]** | Yes | No |
| **ranges[1326]** | Yes | No |
| **blocker data** | Yes | No |
| **available actions** | Yes (inside `action_solutions`) | Yes (inside `available_actions`) |
| **game state** | Yes | Yes |
| **future_actions** | No | Yes (lookahead on street transitions) |
| **is_solution_end** | No | Yes (terminal node detection) |
| **preset_action_code** | No | Yes (UI hint for default selection) |

## Project Structure

```
docs/
  SOLUTION_API.md                      # Spot solution endpoint spec
  NEXT_ACTIONS_API.md                  # Next actions endpoint spec

examples/
  spot-solution/                       # Captured spot-solution API data
    1_flop_root_oop/                   # Flop root, BB to act
    2_flop_check_ip/                   # Flop after BB checks, BTN to act
    3_turn_root_oop/                   # Turn root, BB to act (after X-X)
    4_turn_check_ip/                   # Turn after BB checks, BTN to act
    5_river_root_oop/                  # River root, BB to act (after X-X, X-X)
    6_river_check_ip/                  # River after BB checks, BTN to act

  next-actions/                        # Captured next-actions API data
    1_flop_oop_check/                  # After OOP checks on flop
    2_flop_ip_check/                   # After IP checks on flop (street transition)
    3_turn_oop_check/                  # After OOP checks on turn
    4_turn_ip_check/                   # After IP checks on turn (street transition)
    5_river_oop_check/                 # After OOP checks on river
    6_river_ip_check/                  # After IP checks on river (showdown)
```

Each example folder contains:
- `curl.txt` — full cURL command with headers
- `payload.txt` — query parameters
- `response.json` — full JSON response

## Scenario

All examples use the same hand:

- **Format:** 9-handed MTT, 20bb effective
- **Preflop:** BTN opens to 2bb, BB calls
- **Board:** Ac Td 6h (flop) / Th (turn) / Ah (river)
- **Line:** Check-check on every street to showdown

## Current Coverage

| Endpoint | Method | Status |
|----------|--------|--------|
| `/{version}/solutions/spot-solution/` | GET | Documented |
| `/{version}/poker/next-actions/` | GET | Documented |

## Development

**Swagger UI** (interactive docs):
```bash
npx swagger-ui-watcher openapi.yaml
```

**Mock API server** (queryable endpoints):
```bash
npx @stoplight/prism-cli mock openapi.yaml --port 3000
```

Example queries (auth headers required):
```bash
# Next Actions
curl "http://localhost:3000/v1/poker/next-actions?gametype=MTTGeneral&depth=20.125&preflop_actions=F-F-F-F-F-F-R2-F-C&flop_actions=X&board=AcTd6h" \
  -H "Authorization: Bearer test" \
  -H "gwclientid: test"

# Spot Solution
curl "http://localhost:3000/v1/solutions/spot-solution?gametype=MTTGeneral&depth=20.125&preflop_actions=F-F-F-F-F-F-R2-F-C&board=AcTd6h" \
  -H "Authorization: Bearer test" \
  -H "gwclientid: test"
```

## Patch: 9-Handed Table (vs Original 8-Handed)

This spec extends the original GTO Wizard API from **8-handed** to **9-handed** tables. Below are the key differences:

### Seat Order

| | 8-Handed (original) | 9-Handed (this spec) |
|---|---|---|
| **Seats** | UTG, UTG+1, LJ, HJ, CO, BTN, SB, BB | UTG, UTG+1, **UTG+2**, LJ, HJ, CO, BTN, SB, BB |
| **Total positions** | 8 | 9 |

### Preflop Action Sequence

The `preflop_actions` parameter gains one additional action for the UTG+2 seat:

```diff
- F-F-F-F-F-R2-F-C          (8 actions)
+ F-F-F-F-F-F-R2-F-C        (9 actions)
  │ │ │ │ │ │ │   │ └─ BB
  │ │ │ │ │ │ │   └─── SB
  │ │ │ │ │ │ └─────── BTN
  │ │ │ │ │ └───────── CO
  │ │ │ │ └─────────── HJ
  │ │ │ └───────────── LJ
+ │ │ └─────────────── UTG+2 (new)
  │ └─────────────────  UTG+1
  └───────────────────  UTG
```

### Game State — `game.players` Array

Responses include **9 player objects** instead of 8. The additional player (UTG+2) appears after UTG+1:

```json
{ "position": "UTG",   "is_folded": true, ... },
{ "position": "UTG+1", "is_folded": true, ... },
{ "position": "UTG+2", "is_folded": true, ... },  // added
{ "position": "LJ",    "is_folded": true, ... },
...
```

### Position Enum

The `position` field now includes `UTG+2`:

```
[BB, SB, BTN, CO, HJ, LJ, UTG+2, UTG+1, UTG]
```

### What Stays the Same

- All postflop logic, strategy arrays, EV calculations, and hand indexing (1326 combos) are unchanged
- Board encoding, action codes (X, F, C, R{size}, RAI), and response schemas remain identical
- The pot size (`5.500` for BTN open + BB call) stays the same since only UTG+2 folds preflop
- Stack sizes and bounty fields are unaffected