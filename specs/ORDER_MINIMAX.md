# Order Bot - Minimax with Alpha-Beta Pruning Specification

Strategy using adversarial search to find optimal moves by assuming Chaos plays to minimize Order's score.

## Dependencies

- `COMMON_COMPONENTS.md`: All scoring, move generation, and utility functions

## Overview

- **Search Type:** Minimax with alpha-beta pruning
- **Depth:** 1-4 ply (dynamic based on game phase and branching factor)
- **Time Budget:** 1.8s for search, 0.2s for overhead
- **Target:** Stay under 2-second hard limit

## Core Algorithm

```typescript
interface MinimaxResult {
  score: number;
  move: Move | null;
}

function orderNextTurn(gameState: GameState): OrderMove {
  const timeManager = new TimeManager(2.0);
  const transpositionTable = new LRUCache<string, number>(5000);
  
  // Use iterative deepening
  let bestMove: Move | null = null;
  let depth = 1;
  
  while (timeManager.remaining() > 500 && depth <= 4) { // Keep 500ms buffer
    const result = iterativeDeepening(
      gameState,
      depth,
      timeManager,
      transpositionTable
    );
    
    if (result.move !== null) {
      bestMove = result.move;
    }
    
    depth++;
  }
  
  return formatOrderMove(bestMove);
}

function iterativeDeepening(
  gameState: GameState,
  maxDepth: number,
  timeManager: TimeManager,
  transpositionTable: LRUCache<string, number>
): MinimaxResult {
  const allMoves = getAllLegalMoves(gameState.state);
  
  // Order moves by heuristic score (best first for better pruning)
  const orderedMoves = orderMovesByHeuristic(gameState.state, allMoves);
  
  let bestMove: Move | null = null;
  let bestScore = -Infinity;
  const alpha = -Infinity;
  const beta = Infinity;
  
  for (const move of orderedMoves) {
    if (timeManager.shouldStop()) break;
    
    const newState = applyMove(gameState.state, move);
    
    const score = minimax(
      { ...gameState, state: newState },
      maxDepth - 1,
      alpha,
      beta,
      false, // Next is Chaos (minimizing)
      timeManager,
      transpositionTable
    );
    
    if (score > bestScore) {
      bestScore = score;
      bestMove = move;
    }
  }
  
  return { score: bestScore, move: bestMove };
}
```

## Minimax Implementation

```typescript
function minimax(
  gameState: GameState,
  depth: number,
  alpha: number,
  beta: number,
  isMaximizing: boolean,
  timeManager: TimeManager,
  transpositionTable: LRUCache<string, number>
): number {
  // Check time limit
  if (timeManager.shouldStop()) {
    return evaluatePosition(gameState);
  }
  
  // Check transposition table
  const stateKey = getBoardStateKey(gameState.state);
  const cached = transpositionTable.get(stateKey);
  if (cached !== undefined) {
    return cached;
  }
  
  // Base case: depth limit or game over
  if (depth === 0 || isGameOver(gameState.state)) {
    const eval = evaluatePosition(gameState);
    transpositionTable.set(stateKey, eval);
    return eval;
  }
  
  if (isMaximizing) {
    // Order's turn: maximize score
    return maximizingPlayer(gameState, depth, alpha, beta, timeManager, transpositionTable);
  } else {
    // Chaos's turn: minimize score
    return minimizingPlayer(gameState, depth, alpha, beta, timeManager, transpositionTable);
  }
}

function maximizingPlayer(
  gameState: GameState,
  depth: number,
  alpha: number,
  beta: number,
  timeManager: TimeManager,
  transpositionTable: LRUCache<string, number>
): number {
  let maxEval = -Infinity;
  
  const moves = getAllLegalMoves(gameState.state);
  const orderedMoves = orderMovesByHeuristic(gameState.state, moves);
  
  // Always consider passing as an option
  const passEval = minimax(
    gameState,
    depth - 1,
    alpha,
    beta,
    false,
    timeManager,
    transpositionTable
  );
  
  maxEval = Math.max(maxEval, passEval);
  alpha = Math.max(alpha, maxEval);
  
  for (const move of orderedMoves) {
    if (timeManager.shouldStop() || beta <= alpha) break;
    
    const newState = applyMove(gameState.state, move);
    
    const eval = minimax(
      { ...gameState, state: newState },
      depth - 1,
      alpha,
      beta,
      false,
      timeManager,
      transpositionTable
    );
    
    maxEval = Math.max(maxEval, eval);
    alpha = Math.max(alpha, eval);
    
    if (beta <= alpha) {
      break; // Beta cutoff
    }
  }
  
  return maxEval;
}

function minimizingPlayer(
  gameState: GameState,
  depth: number,
  alpha: number,
  beta: number,
  timeManager: TimeManager,
  transpositionTable: LRUCache<string, number>
): number {
  let minEval = Infinity;
  
  const emptyPositions = getEmptyPositions(gameState.state);
  const orderedPositions = orderChaosPlacementsByHeuristic(
    gameState.state,
    emptyPositions,
    gameState.nextToken
  );
  
  for (const position of orderedPositions) {
    if (timeManager.shouldStop() || beta <= alpha) break;
    
    const newState = applyPlacement(gameState.state, position, gameState.nextToken);
    
    // After Chaos places, Order moves (maximize)
    const eval = minimax(
      { ...gameState, state: newState },
      depth - 1,
      alpha,
      beta,
      true,
      timeManager,
      transpositionTable
    );
    
    minEval = Math.min(minEval, eval);
    beta = Math.min(beta, eval);
    
    if (beta <= alpha) {
      break; // Alpha cutoff
    }
  }
  
  return minEval;
}
```

## Depth Strategy

```typescript
function determineSearchDepth(gameState: GameState): number {
  const phase = determineGamePhase(gameState.state);
  const emptyCount = getEmptyPositions(gameState.state).length;
  const moveCount = getAllLegalMoves(gameState.state).length;
  
  // Branching factor estimate
  const branchingFactor = moveCount * emptyCount;
  
  if (phase === 'early') {
    // High branching factor, stay shallow
    return branchingFactor > 5000 ? 1 : 2;
  } else if (phase === 'mid') {
    // Moderate branching
    return branchingFactor > 2000 ? 2 : 3;
  } else {
    // Late game, can go deeper
    return branchingFactor > 500 ? 3 : 4;
  }
}
```

## Move Ordering for Pruning

```typescript
function orderMovesByHeuristic(state: BoardState, moves: Move[]): Move[] {
  // Score each move with fast heuristic
  const scoredMoves = moves.map(move => {
    const newState = applyMove(state, move);
    const scoreDelta = calculateScoreDelta(state, newState, move);
    return { move, score: scoreDelta };
  });
  
  // Sort descending (best first)
  scoredMoves.sort((a, b) => b.score - a.score);
  
  return scoredMoves.map(sm => sm.move);
}

function orderChaosPlacementsByHeuristic(
  state: BoardState,
  positions: number[],
  color: Color
): number[] {
  // Prioritize positions that disrupt patterns
  const scoredPositions = positions.map(pos => {
    const disruptionScore = evaluatePatternDisruption(state, pos, color);
    return { pos, score: disruptionScore };
  });
  
  // Sort descending (most disruptive first)
  scoredPositions.sort((a, b) => b.score - a.score);
  
  return scoredPositions.map(sp => sp.pos);
}

function evaluatePatternDisruption(
  state: BoardState,
  position: number,
  color: Color
): number {
  const { row, col } = indexToCoords(position);
  
  const rowBefore = getRow(state, row);
  const colBefore = getColumn(state, col);
  
  const potentialBefore = 
    analyzeSequencePotential(rowBefore).almostComplete +
    analyzeSequencePotential(colBefore).almostComplete;
  
  // Simulate placement
  const newState = applyPlacement(state, position, color);
  const rowAfter = getRow(newState, row);
  const colAfter = getColumn(newState, col);
  
  const potentialAfter = 
    analyzeSequencePotential(rowAfter).almostComplete +
    analyzeSequencePotential(colAfter).almostComplete;
  
  // Higher score = more disruption (good for Chaos)
  return potentialBefore - potentialAfter;
}
```

## Evaluation Function

```typescript
function evaluatePosition(gameState: GameState): number {
  const state = gameState.state;
  
  // Primary component: actual score
  const currentScore = calculateFullBoardScore(state);
  
  // Secondary components: potential and threats
  const potential = calculatePatternPotential(state);
  const chaosOpportunities = calculateChaosOpportunities(state);
  const boardControl = calculateBoardControl(state);
  
  return (
    currentScore * 100 +
    potential * 50 +
    boardControl * 20 +
    chaosOpportunities * -30
  );
}

function calculatePatternPotential(state: BoardState): number {
  const analysis = analyzePatternPotential(state);
  
  return (
    analysis.almostComplete * 10 +
    analysis.twoMove * 5 +
    analysis.threeMove * 2
  );
}

function calculateChaosOpportunities(state: BoardState): number {
  const emptyCount = getEmptyPositions(state).length;
  
  // Count patterns that are vulnerable to disruption
  let disruptablePatterns = 0;
  
  for (let row = 0; row < 7; row++) {
    const rowColors = getRow(state, row);
    if (isDisruptable(rowColors)) disruptablePatterns++;
  }
  
  for (let col = 0; col < 7; col++) {
    const colColors = getColumn(state, col);
    if (isDisruptable(colColors)) disruptablePatterns++;
  }
  
  return emptyCount * 0.5 + disruptablePatterns * 2;
}

function isDisruptable(sequence: Color[]): boolean {
  // A sequence is disruptable if it has empty spaces within a potential palindrome
  const emptyCount = sequence.filter(c => c === '').length;
  const filledCount = sequence.filter(c => c !== '').length;
  
  return emptyCount > 0 && filledCount >= 2;
}
```

## Transposition Table

```typescript
function getBoardStateKey(state: BoardState): string {
  // Create unique key for board position
  return state.join('');
}

function isGameOver(state: BoardState): boolean {
  // Game is over when board is full
  return getEmptyPositions(state).length === 0;
}
```

## Optimizations

### 1. Killer Move Heuristic

```typescript
class KillerMoves {
  private killers: Map<number, Move[]>; // depth -> moves
  private maxPerDepth = 2;
  
  constructor() {
    this.killers = new Map();
  }
  
  addKiller(depth: number, move: Move): void {
    if (!this.killers.has(depth)) {
      this.killers.set(depth, []);
    }
    
    const moves = this.killers.get(depth)!;
    
    // Add if not present
    if (!moves.some(m => m.from === move.from && m.to === move.to)) {
      moves.unshift(move);
      
      // Keep only top N
      if (moves.length > this.maxPerDepth) {
        moves.pop();
      }
    }
  }
  
  getKillers(depth: number): Move[] {
    return this.killers.get(depth) || [];
  }
  
  clear(): void {
    this.killers.clear();
  }
}
```

### 2. Move Count Pruning

```typescript
function pruneMovesIfNeeded(moves: Move[], maxMoves: number = 100): Move[] {
  if (moves.length <= maxMoves) {
    return moves;
  }
  
  // Keep only top N moves by heuristic score
  return moves.slice(0, maxMoves);
}
```

### 3. Aspiration Windows

```typescript
function aspirationSearch(
  gameState: GameState,
  depth: number,
  previousScore: number,
  window: number,
  timeManager: TimeManager,
  transpositionTable: LRUCache<string, number>
): MinimaxResult {
  // Search with narrow window around previous best score
  let alpha = previousScore - window;
  let beta = previousScore + window;
  
  const result = minimax(gameState, depth, alpha, beta, true, timeManager, transpositionTable);
  
  // If outside window, re-search with full window
  if (result <= alpha || result >= beta) {
    return minimax(gameState, depth, -Infinity, Infinity, true, timeManager, transpositionTable);
  }
  
  return result;
}
```

## Complexity Analysis

### Best Case (Perfect Pruning)
- **Complexity:** O(b^(d/2))
- **Example:** b=200, d=2 → ~14,000 nodes

### Average Case
- **Complexity:** O(b^(3d/4))
- **Example:** b=200, d=2 → ~28,000 nodes

### Worst Case (No Pruning)
- **Complexity:** O(b^d)
- **Example:** b=200, d=2 → ~40,000 nodes

### Expected Performance
- **Depth 1:** ~200 nodes, 0.1-0.2s
- **Depth 2:** ~15,000-40,000 nodes, 0.8-1.8s
- **Depth 3:** Only late game with low branching factor
- **Depth 4:** Only endgame with <5 empty squares

## Pros & Cons

### ✅ Advantages
1. **Theoretically sound:** Finds provably best move within search depth
2. **Handles tactics well:** Good at seeing forcing sequences
3. **Considers opponent:** Accounts for Chaos's best responses
4. **Scalable:** Can increase depth with more time/compute

### ❌ Disadvantages
1. **Chaos is random:** Minimax assumes optimal play, but Chaos draws randomly
2. **Computational cost:** May struggle to reach depth 2 in complex positions
3. **Shallow horizon:** Depth 2 may miss strategic setups
4. **Time variance:** Performance varies significantly with board complexity

## Testing Strategy

### Unit Tests
```typescript
test('minimax finds winning move', () => {
  const gameState = createTestState([
    'R', 'R', '', '', '', '', '',
    // ... test position where RRR is achievable
  ]);
  
  const move = orderNextTurn(gameState);
  
  expect(move.move).not.toBe('pass');
  // Verify move creates RRR pattern
});

test('minimax respects time limit', () => {
  const complexState = createComplexState();
  const start = Date.now();
  
  orderNextTurn(complexState);
  
  const elapsed = Date.now() - start;
  expect(elapsed).toBeLessThan(2100); // 2s + 100ms tolerance
});
```

### Performance Benchmarks
- Track nodes evaluated per second
- Measure pruning effectiveness (alpha-beta cutoffs)
- Monitor cache hit rates

## Recommendations

### When to Use
- **Competitive play** where optimal moves matter
- **Endgame positions** with low branching factor
- **Critical situations** where tactics dominate

### When Not to Use
- **Early game** with high branching factor (>5000)
- **Casual play** where speed matters more than optimality
- **Time-constrained environments** (e.g., mobile devices)

### Hybrid Approach
Consider combining with heuristic bot:
```typescript
function orderNextTurn(gameState: GameState): OrderMove {
  const phase = determineGamePhase(gameState.state);
  const branchingFactor = estimateBranchingFactor(gameState.state);
  
  if (phase === 'late' || branchingFactor < 1000) {
    return orderMinimaxBot(gameState);
  } else {
    return orderHeuristicBot(gameState);
  }
}
```
