# Entropy Bot - Technical Specification & Implementation Plans

## Interface Specification

### Data Types

```typescript
type Color = 'R' | 'G' | 'Y' | 'B' | 'W' | 'K' | 'P' | '';
type BoardState = Color[]; // 49 elements (7x7), row-major order
type Position = string; // e.g., "a1", "g7"
type Turn = 'chaos' | 'order';

interface GameState {
  state: BoardState;
  nextToken: Color; // The token Chaos will place (Order knows this)
  turn: Turn;
}

interface OrderMove {
  move: string; // "a1-b1" or "pass"
}

interface ChaosMove {
  move: Position; // "a1"
}
```

### API Functions

```typescript
function orderNextTurn(gameState: GameState): OrderMove;
function chaosNextTurn(gameState: GameState): ChaosMove;
```

### Board Indexing
- **Rows:** a-g (top to bottom)
- **Columns:** 1-7 (left to right)
- **Array index:** `row * 7 + col` where row/col are 0-indexed
- Example: "c4" → row=2, col=3 → index=17

---

## Order Bot Implementation Plans

### Plan 1: Minimax with Alpha-Beta Pruning

#### Overview
Use adversarial search assuming Chaos plays optimally to minimize Order's score. Depth-limited to meet 2-second constraint.

#### Algorithm Structure

```
function minimax(state, depth, alpha, beta, isMaximizing):
  if depth == 0 or gameOver:
    return evaluate(state)
  
  if isMaximizing (Order's turn):
    maxEval = -infinity
    for each legal Order move:
      newState = applyMove(state, move)
      eval = minimax(newState, depth-1, alpha, beta, false)
      maxEval = max(maxEval, eval)
      alpha = max(alpha, eval)
      if beta <= alpha:
        break
    return maxEval
  
  else (Chaos's turn):
    minEval = +infinity
    for each empty position:
      newState = placeToken(state, position, nextToken)
      eval = minimax(newState, depth-1, alpha, beta, true)
      minEval = min(minEval, eval)
      beta = min(beta, eval)
      if beta <= alpha:
        break
    return minEval
```

#### Implementation Details

**1. Depth Limitation**
- Start with depth = 2 (Order → Chaos → Order)
- Adjust dynamically based on branching factor
- Early game (many empty squares): depth = 1-2
- Late game (few pieces): depth = 3-4

**2. Move Ordering**
- Order moves by likely score gain (greedy heuristic first)
- Chaos placements: prioritize center, then edges
- Check high-value moves first for better pruning

**3. Evaluation Function**
```
evaluate(state) = 
  currentScore * 100
  + potentialPatterns * 50
  - chaosOpportunities * 30
  + boardControl * 20
```

**4. Optimizations**
- **Transposition Table:** Cache evaluated positions (limited size)
- **Iterative Deepening:** Start at depth 1, increase if time permits
- **Move Count Pruning:** If >500 moves, sample top 100 by heuristic
- **Killer Moves:** Remember moves that caused cutoffs

**5. Time Management**
- Reserve 1.8s for search, 0.2s for overhead
- Check elapsed time every 50 nodes
- Return best move found so far if time exceeded

#### Complexity Analysis
- **Best case:** O(b^(d/2)) with perfect pruning
- **Average case:** O(b^(3d/4))
- **Worst case:** O(b^d)
- b ≈ 200 (avg moves), d = 2 → ~40,000 nodes (feasible)

#### Pros & Cons
✅ **Pros:**
- Theoretically sound
- Handles opponent counter-play
- Good for tactical positions

❌ **Cons:**
- Chaos is random, not optimal (minimax assumption flawed)
- High computational cost
- May be too shallow at depth 2

---

### Plan 2: Heuristic-Based Evaluation

#### Overview
Fast position evaluation using domain-specific heuristics without deep search. Evaluate all legal moves, pick the best.

#### Core Algorithm

```
function orderNextTurn(gameState):
  bestMove = "pass"
  bestScore = currentBoardScore(gameState)
  
  for each piece on board:
    for each legal destination:
      newState = simulateMove(piece, destination)
      score = evaluatePosition(newState)
      if score > bestScore:
        bestScore = score
        bestMove = formatMove(piece, destination)
  
  return bestMove
```

#### Evaluation Function Components

**1. Immediate Score (weight: 100)**
```typescript
immediateScore = calculateFullBoardScore(state);
```
- Scan all rows and columns
- Sum all palindrome scores
- This is the ground truth

**2. Pattern Potential (weight: 50)**
```typescript
patternPotential = sum of:
  - Almost-complete patterns (1 move away): +10 per pattern
  - Two-move patterns: +5 per pattern
  - Three-move patterns: +2 per pattern
```

Examples:
- `R_R` (needs one piece in middle) = almost-complete
- `R__R` (needs two pieces) = two-move pattern
- Row with `RR__GG` = potential for multiple patterns

**3. Board Control (weight: 30)**
```typescript
boardControl = 
  centerControl * 2 +
  edgeControl * 1 +
  connectedGroups * 1.5
```

- **Center control:** Pieces in c4, d4, c5, d5 (premium squares)
- **Edge control:** Pieces on borders (harder for Chaos to disrupt)
- **Connected groups:** Same-color pieces on same row/col (form patterns faster)

**4. Chaos Opportunities (weight: -40)**
```typescript
chaosOpportunities = 
  emptySquaresCount * 0.5 +
  disruptablePatterns * 2
```

- Fewer empty squares = less freedom for Chaos
- Disruptable patterns: patterns that can be broken by one Chaos placement

**5. Color Distribution (weight: 20)**
```typescript
colorDistribution = analyzeColorBalance(state, nextToken);
```

- If nextToken is rare on board: prepare positions to use it
- If nextToken is common: avoid creating easily-breakable patterns

**6. Row/Column Fertility (weight: 25)**
```typescript
fertility = sum over all rows/cols:
  if (emptyCount > 0 && emptyCount < 5):
    score += (7 - emptyCount) * colorDiversity
```

- Rows/cols with 2-4 pieces are "fertile" (can form patterns)
- Empty or full rows have low fertility

#### Complete Evaluation Formula

```typescript
function evaluatePosition(state: GameState): number {
  const immediate = calculateFullBoardScore(state);
  const potential = calculatePatternPotential(state);
  const control = calculateBoardControl(state);
  const opportunities = calculateChaosOpportunities(state);
  const distribution = analyzeColorDistribution(state, state.nextToken);
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

#### Implementation Details

**1. Move Generation**
```typescript
function* generateLegalMoves(state: BoardState) {
  for (let fromIdx = 0; fromIdx < 49; fromIdx++) {
    if (state[fromIdx] === '') continue;
    
    const [fromRow, fromCol] = indexToCoords(fromIdx);
    
    // Check all 4 directions (up, down, left, right)
    for (const direction of DIRECTIONS) {
      let [row, col] = [fromRow, fromCol];
      
      while (true) {
        row += direction.dRow;
        col += direction.dCol;
        
        if (!isValid(row, col)) break;
        
        const toIdx = coordsToIndex(row, col);
        if (state[toIdx] !== '') break; // Blocked
        
        yield { from: fromIdx, to: toIdx };
      }
    }
  }
}
```

**2. Incremental Scoring**
- Don't rescan entire board for each move
- Only check rows/cols affected by the move (2 rows, 2 cols max)
- Cache row/col scores, invalidate only changed ones

**3. Early Termination**
- If a move creates score > currentScore + 20: likely optimal, return early
- If 80% of moves evaluated and one is significantly better: stop

**4. Caching**
- Memoize pattern detection for identical row/col sequences
- LRU cache for board score calculations

#### Complexity Analysis
- **Move generation:** O(n × m) where n=pieces, m=avg 10 destinations = ~300 moves
- **Evaluation per move:** O(7 × 7) = O(49) for smart scoring
- **Total:** O(300 × 49) = ~15,000 operations (well within 2s budget)

#### Pros & Cons
✅ **Pros:**
- Fast and predictable timing
- Domain-specific intelligence
- Easily tunable weights
- Handles all game phases well

❌ **Cons:**
- No lookahead (tactical blind spots)
- Weight tuning requires testing
- May miss complex multi-move setups

---

## Chaos Bot Implementation Plans

### Plan 3: Spoiler Strategy

#### Overview
Actively identify and disrupt Order's best opportunities. Place tokens to minimize Order's potential score.

#### Core Algorithm

```
function chaosNextTurn(gameState):
  const color = gameState.nextToken
  bestPosition = null
  lowestOrderScore = Infinity
  
  for each empty position:
    newState = placeToken(gameState, position, color)
    
    // Simulate Order's best response
    orderBestScore = evaluateBestOrderMove(newState)
    
    if orderBestScore < lowestOrderScore:
      lowestOrderScore = orderBestScore
      bestPosition = position
  
  return bestPosition
```

#### Strategy Components

**1. Pattern Breaking (Priority 1)**
```typescript
function evaluatePatternBreaking(state: BoardState, pos: number, color: Color): number {
  const score = 0;
  const [row, col] = indexToCoords(pos);
  
  // Check row
  const rowBefore = analyzeRow(state, row);
  state[pos] = color;
  const rowAfter = analyzeRow(state, row);
  score += (rowBefore.potential - rowAfter.potential) * 10;
  
  // Check column (same logic)
  score += (colBefore.potential - colAfter.potential) * 10;
  
  return score; // Higher = more disruption
}
```

Examples of high-value disruption:
- `R_R` + place `B` in middle → breaks forming palindrome
- `RGGR` + place `B` at end → prevents `RGGGR`
- Long sequences of same color → insert different color

**2. Opportunity Denial (Priority 2)**
```typescript
function evaluateOpportunityDenial(state: BoardState, pos: number): number {
  // Count how many future palindromes this position could enable
  const [row, col] = indexToCoords(pos);
  
  let opportunities = 0;
  
  // Check if pos is in middle of potential palindromes
  for (let len = 2; len <= 7; len++) {
    if (canFormPalindrome(row, col, len, 'horizontal')) opportunities++;
    if (canFormPalindrome(row, col, len, 'vertical')) opportunities++;
  }
  
  return opportunities; // Higher = more denial value
}
```

**3. Bag Awareness (Priority 3)**
```typescript
function inferRemainingColors(state: BoardState): Map<Color, number> {
  const remaining = new Map([
    ['R', 7], ['G', 7], ['Y', 7], ['B', 7],
    ['W', 7], ['K', 7], ['P', 7]
  ]);
  
  for (const color of state) {
    if (color !== '') {
      remaining.set(color, remaining.get(color)! - 1);
    }
  }
  
  return remaining;
}

function evaluateBagAwareness(state: BoardState, pos: number, color: Color): number {
  const remaining = inferRemainingColors(state);
  const [row, col] = indexToCoords(pos);
  
  // If placing a rare color, prioritize positions where it can't form patterns
  // If placing a common color, use it to block
  
  const colorCount = remaining.get(color) || 0;
  
  if (colorCount <= 2) {
    // Rare color: place where it's isolated
    return calculateIsolationScore(state, pos);
  } else {
    // Common color: place to block patterns
    return calculateBlockingScore(state, pos, color);
  }
}
```

**4. Positional Strategy (Priority 4)**
```typescript
const POSITION_VALUES = [
  // Premium: Center and near-center
  // Low: Corners (less pattern potential)
  
  // Prefer positions that are in multiple rows/cols of interest
  // Avoid positions in dead zones (already broken rows/cols)
];

function getPositionalValue(pos: number): number {
  const [row, col] = indexToCoords(pos);
  
  // Center positions have more impact
  const centerDist = Math.abs(row - 3) + Math.abs(col - 3);
  return (6 - centerDist) * 2;
}
```

#### Complete Evaluation Formula

```typescript
function evaluateChaosPlacement(
  state: BoardState, 
  pos: number, 
  color: Color
): number {
  const breaking = evaluatePatternBreaking(state, pos, color);
  const denial = evaluateOpportunityDenial(state, pos);
  const bagAware = evaluateBagAwareness(state, pos, color);
  const positional = getPositionalValue(pos);
  
  // Simulate Order's best response
  const tempState = [...state];
  tempState[pos] = color;
  const orderBestScore = evaluateBestOrderMove(tempState);
  
  return (
    breaking * 50 +
    denial * 30 +
    bagAware * 20 +
    positional * 10 +
    (-orderBestScore) * 100  // Main objective: minimize Order's score
  );
}
```

#### Implementation Details

**1. Order Response Simulation**
```typescript
function evaluateBestOrderMove(state: GameState): number {
  // Use fast heuristic to estimate Order's best move
  // Don't do full search (too expensive)
  
  let bestScore = calculateFullBoardScore(state);
  
  // Sample ~50 most promising Order moves
  const moves = generateTopOrderMoves(state, 50);
  
  for (const move of moves) {
    const newState = applyMove(state, move);
    const score = calculateFullBoardScore(newState);
    if (score > bestScore) {
      bestScore = score;
    }
  }
  
  return bestScore;
}
```

**2. Caching**
- Cache row/col analysis
- Cache pattern detection results

#### Complexity Analysis
- **Positions to evaluate:** ~49 early game → ~10 late game
- **Per position:** Pattern analysis O(7) + Order simulation O(50 moves)
- **Total:** O(49 × 7 × 50) = ~17,000 operations (feasible)

#### Pros & Cons
✅ **Pros:**
- Directly opposes Order's strategy
- Considers Order's response
- Good against strong Order players

❌ **Cons:**
- Computationally expensive
- May create accidental patterns
- Reactive rather than proactive

---

### Plan 4: Strategic Gap Creation

#### Overview
Proactive strategy to create board positions that inherently resist pattern formation, regardless of Order's response.

#### Core Algorithm

```
function chaosNextTurn(gameState):
  const color = gameState.nextToken
  bestPosition = null
  highestEntropy = -Infinity
  
  for each empty position:
    entropyScore = evaluateEntropy(gameState, position, color)
    
    if entropyScore > highestEntropy:
      highestEntropy = entropyScore
      bestPosition = position
  
  return bestPosition
```

#### Strategy Components

**1. Color Fragmentation (Priority 1)**
```typescript
function evaluateColorFragmentation(state: BoardState, pos: number, color: Color): number {
  const [row, col] = indexToCoords(pos);
  
  let fragmentation = 0;
  
  // Check row: prefer positions that break color runs
  const rowColors = getRowColors(state, row);
  const colColors = getColColors(state, col);
  
  // Count color runs before and after placement
  const rowRunsBefore = countColorRuns(rowColors);
  rowColors[col] = color;
  const rowRunsAfter = countColorRuns(rowColors);
  
  fragmentation += (rowRunsAfter - rowRunsBefore) * 5;
  
  // Same for column
  const colRunsBefore = countColorRuns(colColors);
  colColors[row] = color;
  const colRunsAfter = countColorRuns(colColors);
  
  fragmentation += (colRunsAfter - colRunsBefore) * 5;
  
  return fragmentation; // Higher = more fragmented (good for Chaos)
}
```

Example:
- Row: `RRG___` → placing `B` at position 3 → `RRGB__` (breaks potential RRGGR)
- Better than placing at position 5 → `RRGB_`

**2. Symmetry Breaking (Priority 2)**
```typescript
function evaluateSymmetryBreaking(state: BoardState, pos: number, color: Color): number {
  const [row, col] = indexToCoords(pos);
  
  let symmetryBreak = 0;
  
  // Check row for mirror positions
  const mirrorCol = 6 - col;
  const mirrorColor = state[row * 7 + mirrorCol];
  
  if (mirrorColor !== '' && mirrorColor !== color) {
    symmetryBreak += 10; // Good: creates asymmetry
  }
  
  // Check column for mirror positions
  const mirrorRow = 6 - row;
  const mirrorColorV = state[mirrorRow * 7 + col];
  
  if (mirrorColorV !== '' && mirrorColorV !== color) {
    symmetryBreak += 10;
  }
  
  // Check for existing palindromes and place to break them
  for (let len = 2; len <= 7; len++) {
    if (wouldBreakPalindrome(state, pos, color, len)) {
      symmetryBreak += len * 3;
    }
  }
  
  return symmetryBreak;
}
```

**3. Gap Spacing (Priority 3)**
```typescript
function evaluateGapSpacing(state: BoardState, pos: number): number {
  const [row, col] = indexToCoords(pos);
  
  // Prefer positions that create awkward gaps
  // Gaps of size 1, 3, 5 are harder to fill usefully than 2, 4, 6
  
  let gapScore = 0;
  
  // Check horizontal gaps
  const leftGap = countEmptyToLeft(state, row, col);
  const rightGap = countEmptyToRight(state, row, col);
  
  if (leftGap % 2 === 1) gapScore += 5; // Odd gaps are awkward
  if (rightGap % 2 === 1) gapScore += 5;
  
  // Check vertical gaps
  const upGap = countEmptyUp(state, row, col);
  const downGap = countEmptyDown(state, row, col);
  
  if (upGap % 2 === 1) gapScore += 5;
  if (downGap % 2 === 1) gapScore += 5;
  
  return gapScore;
}
```

**4. Zone Control (Priority 4)**
```typescript
function evaluateZoneControl(state: BoardState, pos: number, color: Color): number {
  // Divide board into zones (3x3 regions)
  // Prefer positions that maximize color diversity in each zone
  
  const zone = getZone(pos); // 9 zones total
  const zoneColors = getColorsInZone(state, zone);
  
  if (!zoneColors.has(color)) {
    return 8; // New color in zone = good diversity
  } else {
    return 2; // Already present = less valuable
  }
}
```

**5. Edge vs Center Strategy**
```typescript
function evaluateEdgeStrategy(state: BoardState, pos: number, gamePhase: string): number {
  const [row, col] = indexToCoords(pos);
  const isEdge = (row === 0 || row === 6 || col === 0 || col === 6);
  const isCenter = (row >= 2 && row <= 4 && col >= 2 && col <= 4);
  
  if (gamePhase === 'early') {
    // Early game: control center to limit Order's options
    return isCenter ? 10 : 5;
  } else if (gamePhase === 'mid') {
    // Mid game: balance
    return isCenter ? 8 : 6;
  } else {
    // Late game: edges matter more (fewer positions left)
    return isEdge ? 10 : 8;
  }
}
```

#### Complete Evaluation Formula

```typescript
function evaluateEntropy(state: BoardState, pos: number, color: Color): number {
  const gamePhase = determineGamePhase(state); // early/mid/late
  
  const fragmentation = evaluateColorFragmentation(state, pos, color);
  const symmetry = evaluateSymmetryBreaking(state, pos, color);
  const gapSpacing = evaluateGapSpacing(state, pos);
  const zoneControl = evaluateZoneControl(state, pos, color);
  const edgeStrategy = evaluateEdgeStrategy(state, pos, gamePhase);
  
  return (
    fragmentation * 40 +
    symmetry * 35 +
    gapSpacing * 15 +
    zoneControl * 20 +
    edgeStrategy * 10
  );
}
```

#### Implementation Details

**1. Game Phase Detection**
```typescript
function determineGamePhase(state: BoardState): string {
  const filled = state.filter(c => c !== '').length;
  
  if (filled < 16) return 'early';   // 0-15 pieces
  if (filled < 35) return 'mid';     // 16-34 pieces
  return 'late';                      // 35-49 pieces
}
```

**2. Pattern Optimization**
- Precompute zone boundaries
- Cache color distributions
- Incremental gap calculations

#### Complexity Analysis
- **Positions to evaluate:** ~49 early → ~10 late
- **Per position:** Fragmentation O(7) + Symmetry O(7) + Gaps O(7) + Zones O(9)
- **Total:** O(49 × 30) = ~1,500 operations (very fast)

#### Pros & Cons
✅ **Pros:**
- Very fast (no deep search)
- Proactive disruption
- Works well in all game phases
- Creates long-term problems for Order

❌ **Cons:**
- Doesn't directly consider Order's threats
- May miss immediate critical blocks
- Requires fine-tuning for balance

---

## Scoring Algorithm Implementation

### Requirements

The scoring system must:
1. Detect all palindromes in complete rows and columns (no diagonals)
2. Count nested palindromes separately
3. Handle all 30 pattern types from game rules

### Algorithm Design

```typescript
function calculateFullBoardScore(state: BoardState): number {
  let totalScore = 0;
  
  // Check all 7 rows
  for (let row = 0; row < 7; row++) {
    const rowColors = getRow(state, row);
    totalScore += scoreSequence(rowColors);
  }
  
  // Check all 7 columns
  for (let col = 0; col < 7; col++) {
    const colColors = getCol(state, col);
    totalScore += scoreSequence(colColors);
  }
  
  return totalScore;
}

function scoreSequence(sequence: Color[]): number {
  // Remove empty spaces, work with contiguous filled sequences
  const filled = sequence.filter(c => c !== '');
  
  if (filled.length < 2) return 0;
  
  let score = 0;
  
  // Check all possible palindrome lengths and positions
  for (let len = 2; len <= filled.length; len++) {
    for (let start = 0; start <= filled.length - len; start++) {
      const substring = filled.slice(start, start + len);
      
      if (isPalindrome(substring)) {
        score += scorePalindrome(substring);
      }
    }
  }
  
  return score;
}

function isPalindrome(seq: Color[]): boolean {
  const len = seq.length;
  for (let i = 0; i < len / 2; i++) {
    if (seq[i] !== seq[len - 1 - i]) {
      return false;
    }
  }
  return true;
}

function scorePalindrome(seq: Color[]): number {
  // Use pattern matching from game rules
  // This requires implementing the 30 patterns
  
  const signature = getPalindromeSignature(seq);
  return PALINDROME_SCORES[signature] || seq.length;
}
```

### Pattern Recognition System

```typescript
// Pattern signature: abstract pattern like "AABAA" from concrete "RRBRBR"
function getPalindromeSignature(seq: Color[]): string {
  const mapping = new Map<Color, string>();
  let nextLetter = 'A'.charCodeAt(0);
  let signature = '';
  
  for (const color of seq) {
    if (!mapping.has(color)) {
      mapping.set(color, String.fromCharCode(nextLetter++));
    }
    signature += mapping.get(color);
  }
  
  return signature;
}

// Scoring table based on game rules
const PALINDROME_SCORES: Record<string, number> = {
  // Length 2
  'AA': 2,
  
  // Length 3
  'ABA': 3,
  'AAA': 7,
  
  // Length 4
  'ABBA': 6,
  'AAAA': 16,
  
  // Length 5
  'ABCBA': 8,
  'AABAA': 12,
  'ABABA': 14,
  'ABBBA': 12,
  'AAAAA': 30,
  
  // Length 6
  'AABBAA': 16,
  'ABAABA': 18,
  'ABBBBA': 22,
  'ABCCBA': 12,
  'AAAAAA': 50,
  
  // Length 7
  'ABCDCBA': 15,
  'AAABAAA': 29,
  'AABABAA': 25,
  'AABBBAA': 23,
  'AABCBAA': 19,
  'ABAAABA': 25,
  'ABABABA': 27,
  'ABACABA': 21,
  'ABBABBA': 27,
  'ABBBBBA': 37,
  'ABBCBBA': 19,
  'ABCACBA': 15,
  'ABCBCBA': 21,
  'ABCCCBA': 19,
  'AAAAAAA': 77,
};
```

### Optimization Considerations

**1. Caching**
```typescript
const scoreCache = new Map<string, number>();

function scoreSequenceCached(sequence: Color[]): number {
  const key = sequence.join('');
  
  if (scoreCache.has(key)) {
    return scoreCache.get(key)!;
  }
  
  const score = scoreSequence(sequence);
  scoreCache.set(key, score);
  
  return score;
}
```

**2. Incremental Scoring**
```typescript
// When making a move, only rescore affected rows/cols
function scoreDelta(oldState: BoardState, newState: BoardState, move: Move): number {
  const [fromRow, fromCol] = indexToCoords(move.from);
  const [toRow, toCol] = indexToCoords(move.to);
  
  const oldScore = 
    scoreRow(oldState, fromRow) + scoreRow(oldState, toRow) +
    scoreCol(oldState, fromCol) + scoreCol(oldState, toCol);
  
  const newScore = 
    scoreRow(newState, fromRow) + scoreRow(newState, toRow) +
    scoreCol(newState, fromCol) + scoreCol(newState, toCol);
  
  return newScore - oldScore;
}
```

### Edge Cases

1. **Empty sequences:** Score = 0
2. **Single piece:** Score = 0 (no palindrome)
3. **Overlapping palindromes:** Count each separately (e.g., ABCBA = 5 for ABCBA + 3 for BCB)
4. **All same color:** Maximum points (77 for AAAAAAA)

---

## Performance Targets

| Bot Strategy | Expected Time | Worst Case | Moves Evaluated |
|-------------|---------------|------------|-----------------|
| Order Minimax (d=2) | 1.2-1.8s | 2.0s | ~40,000 nodes |
| Order Heuristic | 0.3-0.8s | 1.2s | ~300 moves |
| Chaos Spoiler | 0.5-1.2s | 1.8s | ~50 positions × 50 simulations |
| Chaos Gap Creation | 0.1-0.3s | 0.5s | ~50 positions |

---

## Testing Strategy

1. **Unit Tests**
   - Pattern detection
   - Score calculation for known patterns
   - Move generation correctness
   - Coordinate conversion

2. **Integration Tests**
   - Full game simulations
   - Bot vs Bot matches
   - Performance benchmarks

3. **Validation Tests**
   - Compare against human expert moves
   - Verify 2-second constraint
   - Check score accuracy vs manual calculation

---

## Recommendations

### Best Combination for Production

**Order Bot:** Heuristic-Based Evaluation
- Faster, more predictable
- Better user experience (consistent timing)
- Easier to tune and improve
- Can add shallow 1-ply lookahead later if needed

**Chaos Bot:** Hybrid (Spoiler + Gap Creation)
```typescript
function chaosNextTurn(gameState: GameState): ChaosMove {
  const spoilerScore = evaluateSpoiler(gameState);
  const gapScore = evaluateGapCreation(gameState);
  
  // Combine strategies: 60% spoiler, 40% gap creation
  const combinedScore = spoilerScore * 0.6 + gapScore * 0.4;
  
  return bestPositionByCombinedScore;
}
```

### Development Phases

1. **Phase 1:** Implement scoring algorithm + basic move generation
2. **Phase 2:** Order Heuristic-Based bot (simpler, test scoring)
3. **Phase 3:** Chaos Gap Creation bot (simpler, fast)
4. **Phase 4:** Chaos Spoiler bot (more complex)
5. **Phase 5:** Order Minimax bot (most complex)
6. **Phase 6:** Tune weights and optimize

