# Order Bot - Heuristic-Based Evaluation Specification

Fast position evaluation using domain-specific heuristics without deep search. Evaluates all legal moves and selects the best based on weighted scoring components.

## Dependencies

- `COMMON_COMPONENTS.md`: All scoring, move generation, and utility functions

## Overview

- **Strategy Type:** One-ply lookahead with sophisticated evaluation
- **Search:** Evaluate all legal moves (~300-1000 moves)
- **Time Budget:** 0.3-0.8s typical, 1.2s worst case
- **Target:** Well under 2-second limit, fast and predictable

## Core Algorithm

```typescript
function orderNextTurn(gameState: GameState): OrderMove {
  const currentScore = calculateFullBoardScore(gameState.state);
  
  let bestMove: Move | null = null;
  let bestEvaluation = evaluatePosition(gameState);
  
  // Consider passing as an option
  const passEvaluation = evaluatePosition(gameState);
  
  // Evaluate all legal moves
  for (const move of generateLegalMoves(gameState.state)) {
    const newState = applyMove(gameState.state, move);
    const newGameState = { ...gameState, state: newState };
    
    const evaluation = evaluatePosition(newGameState);
    
    if (evaluation > bestEvaluation) {
      bestEvaluation = evaluation;
      bestMove = move;
    }
    
    // Early termination: if we found a great move, take it
    if (evaluation > currentScore + 20) {
      break;
    }
  }
  
  return formatOrderMove(bestMove);
}
```

## Evaluation Function

### Complete Formula

```typescript
function evaluatePosition(gameState: GameState): number {
  const state = gameState.state;
  const nextToken = gameState.nextToken;
  
  const immediate = calculateFullBoardScore(state);
  const potential = calculatePatternPotential(state);
  const control = calculateBoardControl(state);
  const opportunities = calculateChaosOpportunities(state);
  const distribution = analyzeColorDistribution(state, nextToken);
  const fertility = calculateFertility(state);
  
  return (
    immediate * 100 +
    potential * 50 +
    control * 30 +
    opportunities * -40 +
    distribution * 20 +
    fertility * 25
  );
}
```

### Component Weights Rationale

| Component | Weight | Reasoning |
|-----------|--------|-----------|
| Immediate Score | 100 | Primary objective: maximize current score |
| Pattern Potential | 50 | Future scoring opportunities matter |
| Board Control | 30 | Strategic positioning advantage |
| Chaos Opportunities | -40 | Minimize opponent's options (negative = penalty) |
| Color Distribution | 20 | Prepare for known next token |
| Row/Column Fertility | 25 | Focus on productive areas |

---

## Component 1: Immediate Score (Weight: 100)

```typescript
function calculateImmediateScore(state: BoardState): number {
  return calculateFullBoardScore(state);
}
```

**Description:** The actual current score of the board. This is the ground truth and primary optimization target.

**Implementation:** Use the complete scoring algorithm from `COMMON_COMPONENTS.md`.

---

## Component 2: Pattern Potential (Weight: 50)

```typescript
interface PatternPotentialScore {
  almostComplete: number;  // 1 move away
  twoMove: number;         // 2 moves away
  threeMove: number;       // 3+ moves away
}

function calculatePatternPotential(state: BoardState): number {
  const analysis = analyzePatternPotential(state);
  
  return (
    analysis.almostComplete * 10 +
    analysis.twoMove * 5 +
    analysis.threeMove * 2
  );
}
```

**Examples:**
- `R_R` in a row: almostComplete = 1 (needs one piece to form RRR or RGR)
- `R__R` in a row: twoMove = 1 (needs two pieces)
- `RG_GR`: almostComplete = 1 (place any piece to complete RGXGR palindrome)

**Details:**
- Scan all rows and columns
- Identify patterns with empty spaces
- Weight closer-to-complete patterns higher
- Consider symmetry (if left and right pieces match, middle is key)

```typescript
function analyzeRowPotential(row: Color[]): PatternPotentialScore {
  const emptyCount = row.filter(c => c === '').length;
  const filledCount = 7 - emptyCount;
  
  if (filledCount < 2) {
    return { almostComplete: 0, twoMove: 0, threeMove: 0 };
  }
  
  let almostComplete = 0;
  let twoMove = 0;
  let threeMove = 0;
  
  // Check for mirror patterns (potential palindromes)
  // Example: R_R, G__G, etc.
  for (let i = 0; i < 7; i++) {
    if (row[i] === '') continue;
    
    const mirror = 6 - i;
    if (mirror > i && row[mirror] !== '' && row[mirror] === row[i]) {
      // Found matching ends
      const gapSize = mirror - i - 1;
      const emptyInGap = row.slice(i + 1, mirror).filter(c => c === '').length;
      
      if (emptyInGap === 1) almostComplete++;
      else if (emptyInGap === 2) twoMove++;
      else if (emptyInGap >= 3) threeMove++;
    }
  }
  
  // Check for adjacent pairs (AA can extend to AAA, AAAA, etc.)
  for (let i = 0; i < 6; i++) {
    if (row[i] !== '' && row[i] === row[i + 1]) {
      // Found pair, check if can extend
      if (i > 0 && row[i - 1] === '') almostComplete++;
      if (i < 5 && row[i + 2] === '') almostComplete++;
    }
  }
  
  return { almostComplete, twoMove, threeMove };
}
```

---

## Component 3: Board Control (Weight: 30)

```typescript
function calculateBoardControl(state: BoardState): number {
  const centerControl = calculateCenterControl(state);
  const edgeControl = calculateEdgeControl(state);
  const connectedGroups = calculateConnectedGroups(state);
  
  return centerControl * 2 + edgeControl * 1 + connectedGroups * 1.5;
}
```

### 3.1 Center Control

```typescript
function calculateCenterControl(state: BoardState): number {
  // Premium center squares: c4, d4, c5, d5 (indices 17, 24, 18, 25)
  const centerSquares = [
    coordsToIndex(2, 3), // c4
    coordsToIndex(3, 3), // d4
    coordsToIndex(2, 4), // c5
    coordsToIndex(3, 4)  // d5
  ];
  
  let control = 0;
  for (const idx of centerSquares) {
    if (state[idx] !== '') {
      control++;
    }
  }
  
  // Extended center (16 squares around center)
  const extendedCenter = [
    // Rows 2-4, cols 2-4
  ];
  
  for (const idx of extendedCenter) {
    if (state[idx] !== '') {
      control += 0.5;
    }
  }
  
  return control;
}
```

**Rationale:** Center squares participate in more rows/columns and are harder for Chaos to isolate.

### 3.2 Edge Control

```typescript
function calculateEdgeControl(state: BoardState): number {
  let control = 0;
  
  for (let i = 0; i < 49; i++) {
    const { row, col } = indexToCoords(i);
    const isEdge = row === 0 || row === 6 || col === 0 || col === 6;
    
    if (isEdge && state[i] !== '') {
      control++;
    }
  }
  
  return control;
}
```

**Rationale:** Edge pieces are in complete rows/columns and are more stable (fewer ways for Chaos to disrupt).

### 3.3 Connected Groups

```typescript
function calculateConnectedGroups(state: BoardState): number {
  let groups = 0;
  
  // Count same-color adjacencies in rows
  for (let row = 0; row < 7; row++) {
    const rowColors = getRow(state, row);
    groups += countColorGroups(rowColors);
  }
  
  // Count same-color adjacencies in columns
  for (let col = 0; col < 7; col++) {
    const colColors = getColumn(state, col);
    groups += countColorGroups(colColors);
  }
  
  return groups;
}

function countColorGroups(sequence: Color[]): number {
  const colorCounts = new Map<Color, number>();
  
  for (const color of sequence) {
    if (color !== '') {
      colorCounts.set(color, (colorCounts.get(color) || 0) + 1);
    }
  }
  
  // Count colors with 2+ pieces (can form patterns)
  let groups = 0;
  for (const count of colorCounts.values()) {
    if (count >= 2) {
      groups += count - 1; // AA=1, AAA=2, etc.
    }
  }
  
  return groups;
}
```

**Rationale:** Multiple pieces of the same color in a row/column can form high-value patterns (AA, AAA, AAAA).

---

## Component 4: Chaos Opportunities (Weight: -40)

```typescript
function calculateChaosOpportunities(state: BoardState): number {
  const emptyCount = getEmptyPositions(state).length;
  const disruptablePatterns = countDisruptablePatterns(state);
  
  return emptyCount * 0.5 + disruptablePatterns * 2;
}

function countDisruptablePatterns(state: BoardState): number {
  let count = 0;
  
  // Check rows
  for (let row = 0; row < 7; row++) {
    const rowColors = getRow(state, row);
    if (isVulnerableToDisruption(rowColors)) {
      count++;
    }
  }
  
  // Check columns
  for (let col = 0; col < 7; col++) {
    const colColors = getColumn(state, col);
    if (isVulnerableToDisruption(colColors)) {
      count++;
    }
  }
  
  return count;
}

function isVulnerableToDisruption(sequence: Color[]): boolean {
  const emptyCount = sequence.filter(c => c === '').length;
  const filledCount = sequence.filter(c => c !== '').length;
  
  // Vulnerable if:
  // 1. Has empty spaces (Chaos can place there)
  // 2. Has 2+ filled pieces (forming or can form patterns)
  if (emptyCount === 0 || filledCount < 2) {
    return false;
  }
  
  // Check if there's a near-complete palindrome
  const filled = sequence.filter(c => c !== '');
  
  // Simple heuristic: if first and last filled pieces match
  if (filled.length >= 2 && filled[0] === filled[filled.length - 1]) {
    return true;
  }
  
  // Or if there are pairs/runs of same color
  for (let i = 0; i < filled.length - 1; i++) {
    if (filled[i] === filled[i + 1]) {
      return true;
    }
  }
  
  return false;
}
```

**Rationale:** Minimize spaces where Chaos can disrupt forming patterns. This is a negative component (penalty).

---

## Component 5: Color Distribution (Weight: 20)

```typescript
function analyzeColorDistribution(state: BoardState, nextToken: Color): number {
  const remaining = inferRemainingColors(state);
  const onBoard = getColorDistribution(state);
  
  const nextTokenRemaining = remaining.get(nextToken) || 0;
  const nextTokenOnBoard = onBoard.get(nextToken) || 0;
  
  let score = 0;
  
  // If nextToken is rare on board, prepare to use it
  if (nextTokenOnBoard <= 1) {
    score += 10; // Opportunity to start new patterns
  }
  
  // If nextToken is common on board, risk of easy disruption
  if (nextTokenOnBoard >= 4) {
    score -= 5; // Chaos can easily break patterns
  }
  
  // Check if board has good color diversity (harder to disrupt)
  const uniqueColors = onBoard.size;
  score += uniqueColors * 2;
  
  // Strategic preparation: check if positions exist for nextToken
  score += evaluateNextTokenPositions(state, nextToken);
  
  return score;
}

function evaluateNextTokenPositions(state: BoardState, nextToken: Color): number {
  let score = 0;
  
  // Find positions where placing nextToken could complete patterns
  const emptyPositions = getEmptyPositions(state);
  
  for (const pos of emptyPositions) {
    const { row, col } = indexToCoords(pos);
    
    const rowColors = getRow(state, row);
    const colColors = getColumn(state, col);
    
    // Check if nextToken would complete a pattern
    const rowHasNextToken = rowColors.some(c => c === nextToken);
    const colHasNextToken = colColors.some(c => c === nextToken);
    
    if (rowHasNextToken || colHasNextToken) {
      score += 2; // Good position exists for next Chaos token
    }
  }
  
  return score;
}
```

**Rationale:** Order knows the next token Chaos will place. Prepare positions to maximize or minimize its impact.

---

## Component 6: Row/Column Fertility (Weight: 25)

```typescript
function calculateFertility(state: BoardState): number {
  let totalFertility = 0;
  
  // Analyze each row
  for (let row = 0; row < 7; row++) {
    const rowColors = getRow(state, row);
    totalFertility += calculateSequenceFertility(rowColors);
  }
  
  // Analyze each column
  for (let col = 0; col < 7; col++) {
    const colColors = getColumn(state, col);
    totalFertility += calculateSequenceFertility(colColors);
  }
  
  return totalFertility;
}

function calculateSequenceFertility(sequence: Color[]): number {
  const emptyCount = sequence.filter(c => c === '').length;
  const filledCount = 7 - emptyCount;
  
  // Fertility curve: rows/cols with 2-4 pieces are most fertile
  // 0-1 pieces: low fertility (nothing to build on)
  // 2-4 pieces: high fertility (can form patterns)
  // 5-6 pieces: medium fertility (less flexibility)
  // 7 pieces: no fertility (complete, but good!)
  
  if (filledCount === 0 || filledCount === 1) {
    return 0;
  } else if (filledCount >= 2 && filledCount <= 4) {
    // Sweet spot
    const colorDiversity = calculateColorDiversity(sequence);
    return (5 - emptyCount) * colorDiversity * 2;
  } else if (filledCount >= 5 && filledCount <= 6) {
    return filledCount * 1.5;
  } else {
    // Full row/column
    return 10; // Bonus for complete
  }
}

function calculateColorDiversity(sequence: Color[]): number {
  const uniqueColors = new Set(sequence.filter(c => c !== '')).size;
  
  // Diversity of 2-3 is good (can form interesting palindromes)
  // Diversity of 1 is excellent (uniform, high-value patterns)
  // Diversity of 4+ is poor (hard to form palindromes)
  
  if (uniqueColors === 1) {
    return 3; // Best: uniform color (AAA, AAAA, etc.)
  } else if (uniqueColors === 2) {
    return 2.5; // Good: ABBA, ABABA type patterns
  } else if (uniqueColors === 3) {
    return 2; // OK: ABCBA type patterns
  } else {
    return 1; // Poor: too diverse
  }
}
```

**Rationale:** Focus on rows/columns with potential. Empty or nearly-full rows offer less strategic value.

---

## Optimization Techniques

### 1. Incremental Evaluation

```typescript
function evaluateMove(
  baseState: BoardState,
  move: Move,
  baseEvaluation: number
): number {
  // Only recalculate affected components
  const newState = applyMove(baseState, move);
  
  const { row: fromRow, col: fromCol } = indexToCoords(move.from);
  const { row: toRow, col: toCol } = indexToCoords(move.to);
  
  // Incremental score calculation
  const scoreDelta = calculateScoreDelta(baseState, newState, move);
  
  // Recalculate only affected rows/columns for other components
  const affectedRows = new Set([fromRow, toRow]);
  const affectedCols = new Set([fromCol, toCol]);
  
  // This is much faster than full board evaluation
  return baseEvaluation + scoreDelta + calculatePartialComponents(
    newState,
    affectedRows,
    affectedCols
  );
}
```

### 2. Move Pruning

```typescript
function getTopMoves(state: BoardState, maxMoves: number = 100): Move[] {
  const allMoves = getAllLegalMoves(state);
  
  if (allMoves.length <= maxMoves) {
    return allMoves;
  }
  
  // Quick score each move by immediate score delta
  const scoredMoves = allMoves.map(move => {
    const newState = applyMove(state, move);
    const scoreDelta = calculateScoreDelta(state, newState, move);
    return { move, score: scoreDelta };
  });
  
  // Sort and take top N
  scoredMoves.sort((a, b) => b.score - a.score);
  return scoredMoves.slice(0, maxMoves).map(sm => sm.move);
}
```

### 3. Early Termination

```typescript
function findBestMove(gameState: GameState): Move | null {
  const currentScore = calculateFullBoardScore(gameState.state);
  
  let bestMove: Move | null = null;
  let bestEval = -Infinity;
  
  for (const move of generateLegalMoves(gameState.state)) {
    const newState = applyMove(gameState.state, move);
    const eval = evaluatePosition({ ...gameState, state: newState });
    
    if (eval > bestEval) {
      bestEval = eval;
      bestMove = move;
    }
    
    // Early termination: if we found a move that scores 20+ points, take it
    if (eval > currentScore + 20) {
      return bestMove;
    }
  }
  
  return bestMove;
}
```

### 4. Caching

```typescript
const evaluationCache = new LRUCache<string, number>(500);

function evaluatePositionCached(gameState: GameState): number {
  const key = getBoardStateKey(gameState.state);
  
  const cached = evaluationCache.get(key);
  if (cached !== undefined) {
    return cached;
  }
  
  const eval = evaluatePosition(gameState);
  evaluationCache.set(key, eval);
  
  return eval;
}
```

---

## Complexity Analysis

### Time Complexity
- **Move generation:** O(n × m) where n ≈ 20-40 pieces, m ≈ 10 destinations → ~300-400 moves
- **Evaluation per move:** O(49) for smart incremental scoring
- **Total:** O(300 × 49) ≈ 15,000 operations

### Space Complexity
- **State storage:** O(49) per position
- **Cache:** O(500) cached evaluations
- **Total:** O(1) effective (constant space)

### Expected Performance
- **Early game** (few pieces): 0.2-0.4s
- **Mid game** (many pieces): 0.4-0.8s
- **Late game** (few empty): 0.3-0.5s
- **Worst case:** 1.2s (never exceeds 2s)

---

## Pros & Cons

### ✅ Advantages
1. **Fast and predictable:** Consistent timing across all positions
2. **Strategic depth:** Multi-component evaluation captures strategy
3. **Tunable:** Easy to adjust weights based on testing
4. **Explainable:** Clear why moves are chosen
5. **Robust:** Handles all game phases well
6. **No search overhead:** No tree traversal complexity

### ❌ Disadvantages
1. **No lookahead:** Can't see 2-3 move combinations
2. **Tactical blind spots:** May miss forcing sequences
3. **Weight sensitivity:** Requires tuning for optimal play
4. **Heuristic errors:** Domain assumptions may be wrong
5. **Greedy:** Locally optimal moves may not be globally optimal

---

## Weight Tuning Strategy

### Testing Framework
```typescript
interface TuningConfig {
  immediateWeight: number;
  potentialWeight: number;
  controlWeight: number;
  opportunitiesWeight: number;
  distributionWeight: number;
  fertilityWeight: number;
}

function testConfiguration(
  config: TuningConfig,
  testGames: GameState[]
): number {
  let totalScore = 0;
  
  for (const game of testGames) {
    const finalScore = playGameWithWeights(game, config);
    totalScore += finalScore;
  }
  
  return totalScore / testGames.length;
}
```

### Recommended Starting Weights
```typescript
const DEFAULT_WEIGHTS: TuningConfig = {
  immediateWeight: 100,
  potentialWeight: 50,
  controlWeight: 30,
  opportunitiesWeight: -40,
  distributionWeight: 20,
  fertilityWeight: 25
};
```

### Tuning Process
1. Play 100 test games with default weights
2. Record average score
3. Adjust one weight at a time (±20%)
4. Keep changes that improve average score
5. Repeat until convergence

---

## Testing Strategy

### Unit Tests
```typescript
test('evaluates winning position highly', () => {
  const gameState = createTestState([
    'R', 'R', 'R', '', '', '', '',  // RRR = 7 points
    // ...
  ]);
  
  const eval = evaluatePosition(gameState);
  expect(eval).toBeGreaterThan(700); // 7 * 100 + components
});

test('prefers scoring moves over neutral moves', () => {
  const gameState = createTestState(/* ... */);
  
  const move1 = { from: 0, to: 1 }; // Scores 5 points
  const move2 = { from: 2, to: 3 }; // Scores 0 points
  
  const eval1 = evaluateMove(gameState.state, move1);
  const eval2 = evaluateMove(gameState.state, move2);
  
  expect(eval1).toBeGreaterThan(eval2);
});

test('completes moves within time limit', () => {
  const complexState = createComplexState();
  const start = Date.now();
  
  orderNextTurn(complexState);
  
  const elapsed = Date.now() - start;
  expect(elapsed).toBeLessThan(1500); // Well under 2s
});
```

### Integration Tests
```typescript
test('bot achieves good score in full game', () => {
  const finalScore = playFullGame(orderHeuristicBot, randomChaosBot);
  
  expect(finalScore).toBeGreaterThan(75); // Average score
});
```

---

## Recommendations

### Best Use Cases
- **Default strategy** for production
- **Fast response** requirements
- **All game phases** (early, mid, late)
- **Casual to intermediate** play

### Enhancement Opportunities
1. **Add 1-ply lookahead:** Check immediate Chaos response
2. **Pattern library:** Pre-compute high-value pattern setups
3. **Machine learning:** Learn weight values from game data
4. **Opening book:** Pre-computed strong early moves

### Production Configuration
```typescript
const PRODUCTION_CONFIG = {
  weights: DEFAULT_WEIGHTS,
  maxMovesToEvaluate: 200, // Limit for very complex positions
  enableEarlyTermination: true,
  cacheSize: 500,
  timeLimit: 1.8 // Reserve 0.2s buffer
};
```
