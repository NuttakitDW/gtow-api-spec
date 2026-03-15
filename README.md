# GTO Wizard API Spec

Reverse-engineered API specification for the GTO Wizard postflop solver.

## Project Structure

```
docs/
  API_SPEC.md          # Full API spec for the spot-solution endpoint

examples/
  spot1/               # Captured API data: Flop root, BB vs BTN, AcTd6h
    curl.txt           # cURL command with headers
    flop_root_payload.txt   # Query parameters
    flop_root_response.json # Full JSON response (~960KB)
```

## Current Coverage

| Endpoint | Method | Status |
|----------|--------|--------|
| `/v4/solutions/spot-solution/` | GET | Documented |

## Quick Reference

**Base URL:** `https://api.gtowizard.com`

**Example query:**
```
GET /v4/solutions/spot-solution/?gametype=MTTGeneral&depth=20.125&preflop_actions=F-F-F-F-F-R2-F-C&flop_actions=&turn_actions=&river_actions=&board=AcTd6h
```

Returns GTO strategy for all 1326 hand combinations at a given decision node, including:
- Strategy frequencies and expected values per action
- Both players' preflop ranges
- Hand/draw categories, blocker data
- Full game state (pot, stacks, positions)
