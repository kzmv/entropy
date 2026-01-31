# Entropy Bot - ORDER_MINIMAX Implementation

CLI application that uses minimax with alpha-beta pruning to determine the best move for the Order player in the Entropy board game.

## Features

- Minimax algorithm with alpha-beta pruning
- Iterative deepening (depths 1-4)
- Transposition table caching
- Move ordering for better pruning
- 2-second time limit with buffer
- Visual board display

## Installation

```bash
npm install
```

## Usage

### Run with ts-node (Development)

```bash
npm run dev <game_state_file.json>
```

### Build and Run (Production)

```bash
npm run build
npm start <game_state_file.json>
```

### Example

```bash
npm run dev example_game_states.json
```

## Input Format

The JSON file should contain a game state with the following structure:

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

Or an array of game states (the first one will be used):

```json
[
  {
    "id": 1,
    "description": "Example game state",
    "state": [...],
    "nextToken": "R",
    "turn": "order"
  }
]
```

### Board State Format
- 49 elements representing a 7x7 board (row-major order)
- Empty squares: `""`
- Colors: `"R"` (Red), `"G"` (Green), `"Y"` (Yellow), `"B"` (Blue), `"W"` (White), `"K"` (Black), `"P"` (Purple)

### Output Format

The bot outputs the next move in the format:
- `a1-a3` - Move piece from a1 to a3
- `pass` - Skip turn

## Algorithm Details

### Minimax with Alpha-Beta Pruning
- **Search Depth**: 1-4 ply (adaptive based on time)
- **Time Budget**: 1.8s search + 0.2s buffer = 2.0s total
- **Transposition Table**: 5000 entries (LRU cache)

### Evaluation Function
```
score = currentScore × 100 + potential × 50 - chaosOpportunities × 30
```

Where:
- `currentScore`: Actual board score from palindromic patterns
- `potential`: Pattern completion opportunities
- `chaosOpportunities`: Empty positions and disruptable patterns

### Move Ordering
- Order moves: Sorted by score delta (descending)
- Chaos placements: Sorted by score delta (ascending, most disruptive first)

## Project Structure

```
entropy-bot/
├── src/
│   ├── common/
│   │   ├── types.ts              # Type definitions
│   │   ├── constants.ts          # Game constants
│   │   ├── coordinates.ts        # Position conversions
│   │   ├── scoring.ts            # Palindrome scoring
│   │   ├── moveGeneration.ts    # Legal move generation
│   │   ├── boardAnalysis.ts     # Pattern analysis
│   │   └── utilities.ts         # TimeManager, LRUCache
│   ├── order/
│   │   ├── evaluation.ts         # Position evaluation
│   │   └── minimax.ts           # Minimax implementation
│   ├── cli.ts                    # CLI interface
│   └── index.ts                  # Main exports
├── example_game_states.json      # Test game states
├── package.json
├── tsconfig.json
└── README.md
```

## Performance

- **Depth 1**: ~0.1-0.2s
- **Depth 2**: ~0.8-1.8s
- **Depth 3**: Late game only
- **Depth 4**: Endgame only (<5 empty squares)

## Testing

Test with the provided example game states:

```bash
npm run dev example_game_states.json
```

The bot will analyze the first game state and output the recommended move.
