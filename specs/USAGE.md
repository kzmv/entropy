# Quick Start Guide

## Installation

```bash
npm install
```

## Usage

### Run the bot on a game state file:

```bash
npm run dev <path_to_json_file>
```

### Examples:

```bash
# Simple test case
npm run dev test_order_state.json

# Complex mid-game state
npm run dev test_order_complex.json

# Use specific game state from example file (first one must be Order's turn)
npm run dev example_game_states.json
```

## Input Format

The JSON file must contain a game state with:
- `state`: Array of 49 colors (7×7 board, row-major order)
- `nextToken`: The color token Chaos will place next
- `turn`: Must be `"order"` for this bot

**Colors:** `"R"`, `"G"`, `"Y"`, `"B"`, `"W"`, `"K"`, `"P"`, or `""` for empty

**Example:**
```json
{
  "state": [
    "", "", "", "", "", "", "",
    "", "", "", "", "", "", "",
    "", "", "", "", "", "", "",
    "", "", "", "R", "", "", "",
    "", "", "", "", "", "", "",
    "", "", "", "", "", "", "",
    "", "", "", "", "", "", ""
  ],
  "nextToken": "G",
  "turn": "order"
}
```

## Output Format

The bot outputs:
- Visual board representation
- Recommended move in format: `d4-d3` (from-to) or `pass`
- Calculation time

**Example Output:**
```
Current Board:
   a b c d e f g
1  · · · · · · ·
2  · · · · · · ·
3  · · · · · · ·
4  · · · R · · ·
5  · · · · · · ·
6  · · · · · · ·
7  · · · · · · ·

==================================================
NEXT MOVE: d4-d3
==================================================
Calculation time: 0.00s
```

## Build for Production

```bash
npm run build
npm start <path_to_json_file>
```
