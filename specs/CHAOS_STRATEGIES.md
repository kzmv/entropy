# Chaos Bot Strategies Specification

Two complementary strategies for the Chaos player: Spoiler Strategy (reactive) and Strategic Gap Creation (proactive).

## Dependencies

- `COMMON_COMPONENTS.md`: All scoring, move generation, and utility functions

## Overview

Chaos player's goal: **Minimize Order's scoring opportunities** by disrupting pattern formation.

### Strategy Comparison

| Aspect | Spoiler Strategy | Gap Creation Strategy |
|--------|------------------|----------------------|
| Approach | Reactive | Proactive |
| Focus | Disrupt Order's plans | Create inherently difficult positions |
| Computation | Heavier (simulates Order) | Lighter (heuristics only) |
| Time | 0.5-1.2s | 0.1-0.3s |
| Best Phase | Mid-late game | Early-mid game |

### Recommended Hybrid

```typescript
function chaosNextTurn(gameState: GameState): ChaosMove {
  const phase = determineGamePhase(gameState.state);
  const emptyCount = getEmptyPositions(gameState.state).length;
  
  // Use hybrid approach: 60% Spoiler + 40% Gap Creation
  const spoilerScore = evaluateSpoilerStrategy(gameState);
  const gapScore = evaluateGapStrategy(gameState);
  
  // Weight by game phase
  let spoilerWeight = 0.6;
  let gapWeight = 0.4;
  
  if (phase === 'early') {
    spoilerWeight = 0.4;
    gapWeight = 0.6; // Gap creation more important early
  } else if (phase === 'late') {
    spoilerWeight = 0.8;
    gapWeight = 0.2; // Spoiler more important late
  }
  
  const bestPosition = findBestChaosPosition(
    gameState,
    spoilerWeight,
    gapWeight
  );
  
  return formatChaosMove(bestPosition);
}
```

---

# Strategy 1: Spoiler Strategy

## Core Philosophy

Actively identify Order's most promising opportunities and place tokens to minimize their potential. Simulate Order's best response to each placement.

## Algorithm

```typescript
function chaosNextTurn(gameState: GameState): ChaosMove {
  const color = gameState.nextToken;
  const emptyPositions = getEmptyPositions(gameState.state);
  
  let bestPosition = emptyPositions[0]; // Fallback
  let lowestOrderPotential = Infinity;
  
  for (const position of emptyPositions) {
    const score = evaluateSpoilerPlacement(gameState, position, color);
    
    if (score < lowestOrderPotential) {
      lowestOrderPotential = score;
      bestPosition = position;
    }
  }
  
  return formatChaosMove(bestPosition);
}

function evaluateSpoilerPlacement(
  gameState: GameState,
  position: number,
  color: Color
): number {
  // Lower score = better for Chaos (minimizes Order's opportunities)
  
  const newState = applyPlacement(gameState.state, position, color);
  
  // Evaluate multiple factors
  const patternBreaking = evaluatePatternBreaking(newState, position, color);
  const opportunityDenial = evaluateOpportunityDenial(newState, position);
  const bagAwareness = evaluateBagAwareness(newState, position, color);
  const orderBestScore = estimateOrderBestMove(newState);
  
  // Combine scores (negative = good for Chaos)
  return (
    -patternBreaking * 50 +
    -opportunityDenial * 30 +
    -bagAwareness * 20 +
    orderBestScore * 100  // Primary: minimize Order's potential score
  );
}
```

## Component 1: Pattern Breaking

```typescript
function evaluatePatternBreaking(
  state: BoardState,
  position: number,
  color: Color
): number {
  const { row, col } = indexToCoords(position);
  
  // Analyze row before placement
  const rowBefore = getRow(state, row);
  const rowPotentialBefore = analyzeSequencePotential(rowBefore);
  
  // Simulate placement
  const tempState = [...state];
  tempState[position] = color;
  
  // Analyze row after placement
  const rowAfter = getRow(tempState, row);
  const rowPotentialAfter = analyzeSequencePotential(rowAfter);
  
  // Same for column
  const colBefore = getColumn(state, col);
  const colPotentialBefore = analyzeSequencePotential(colBefore);
  
  const colAfter = getColumn(tempState, col);
  const colPotentialAfter = analyzeSequencePotential(colAfter);
  
  // Calculate disruption (reduction in potential)
  const rowDisruption = 
    (rowPotentialBefore.almostComplete - rowPotentialAfter.almostComplete) * 10 +
    (rowPotentialBefore.twoMove - rowPotentialAfter.twoMove) * 5;
  
  const colDisruption = 
    (colPotentialBefore.almostComplete - colPotentialAfter.almostComplete) * 10 +
    (colPotentialBefore.twoMove - colPotentialAfter.twoMove) * 5;
  
  return rowDisruption + colDisruption;
}
```

### Pattern Breaking Examples

**High-value disruption:**
- `R_R` → place `B` in middle → `RBR` (breaks potential RGR, RRR palindromes)
- `RGGR` → place `B` at end → `RGGRB` (prevents RGGXGGR extension)
- `G_G__` → place `R` at position 1 → `GRG__` (still allows patterns but limits them)

**Low-value disruption:**
- `___R_` → place anywhere (row has little potential anyway)
- `RGBYP` → place at end (already too diverse to form good patterns)

## Component 2: Opportunity Denial

```typescript
function evaluateOpportunityDenial(
  state: BoardState,
  position: number
): number {
  const { row, col } = indexToCoords(position);
  
  let denialScore = 0;
  
  // Count how many future palindromes this position could enable
  // Check all possible palindrome lengths centered or involving this position
  
  for (let len = 2; len <= 7; len++) {
    // Horizontal palindromes
    for (let start = Math.max(0, col - len + 1); start <= Math.min(6, col); start++) {
      if (start + len > 7) continue;
      
      if (couldFormPalindrome(state, row, start, len, 'horizontal', position)) {
        denialScore += len;
      }
    }
    
    // Vertical palindromes
    for (let start = Math.max(0, row - len + 1); start <= Math.min(6, row); start++) {
      if (start + len > 7) continue;
      
      if (couldFormPalindrome(state, start, col, len, 'vertical', position)) {
        denialScore += len;
      }
    }
  }
  
  return denialScore;
}

function couldFormPalindrome(
  state: BoardState,
  startRow: number,
  startCol: number,
  length: number,
  direction: 'horizontal' | 'vertical',
  position: number
): boolean {
  // Check if a palindrome of given length could form
  // Returns true if position is critical for this palindrome
  
  const sequence: Color[] = [];
  
  for (let i = 0; i < length; i++) {
    const idx = direction === 'horizontal' 
      ? coordsToIndex(startRow, startCol + i)
      : coordsToIndex(startRow + i, startCol);
    
    sequence.push(state[idx]);
  }
  
  // Check if position is in this sequence
  const posInSequence = direction === 'horizontal'
    ? (indexToCoords(position).col - startCol)
    : (indexToCoords(position).row - startRow);
  
  if (posInSequence < 0 || posInSequence >= length) {
    return false; // Position not in this sequence
  }
  
  // Check if remaining pieces could form palindrome
  const emptyCount = sequence.filter(c => c === '').length;
  const filledCount = length - emptyCount;
  
  if (filledCount < 2) return false; // Not enough pieces
  
  // Check mirror positions match
  let matches = 0;
  for (let i = 0; i < Math.floor(length / 2); i++) {
    const left = sequence[i];
    const right = sequence[length - 1 - i];
    
    if (left !== '' && right !== '' && left === right) {
      matches++;
    }
  }
  
  return matches >= 1; // At least one mirror pair exists
}
```

## Component 3: Bag Awareness

```typescript
function evaluateBagAwareness(
  state: BoardState,
  position: number,
  color: Color
): number {
  const remaining = inferRemainingColors(state);
  const colorCount = remaining.get(color) || 0;
  
  const { row, col } = indexToCoords(position);
  
  let score = 0;
  
  if (colorCount <= 2) {
    // Rare color: place where it's isolated (can't easily form patterns)
    score += calculateIsolationScore(state, position, color);
  } else {
    // Common color: use to block high-value positions
    score += calculateBlockingScore(state, position, color);
  }
  
  return score;
}

function calculateIsolationScore(
  state: BoardState,
  position: number,
  color: Color
): number {
  const { row, col } = indexToCoords(position);
  
  // Check how many same-color pieces are in this row/column
  const rowColors = getRow(state, row);
  const colColors = getColumn(state, col);
  
  const sameColorInRow = rowColors.filter(c => c === color).length;
  const sameColorInCol = colColors.filter(c => c === color).length;
  
  // Lower count = more isolated = better
  const isolation = (7 - sameColorInRow) + (7 - sameColorInCol);
  
  return isolation;
}

function calculateBlockingScore(
  state: BoardState,
  position: number,
  color: Color
): number {
  const { row, col } = indexToCoords(position);
  
  // Check if this position is in a high-potential row/column
  const rowPotential = analyzeSequencePotential(getRow(state, row));
  const colPotential = analyzeSequencePotential(getColumn(state, col));
  
  const totalPotential = 
    rowPotential.almostComplete * 10 +
    rowPotential.twoMove * 5 +
    colPotential.almostComplete * 10 +
    colPotential.twoMove * 5;
  
  // Higher potential = better blocking position
  return totalPotential;
}
```

## Component 4: Order Best Move Simulation

```typescript
function estimateOrderBestMove(state: BoardState): number {
  // Estimate Order's best possible move from this position
  // Use fast heuristic, not full search (too expensive)
  
  const currentScore = calculateFullBoardScore(state);
  let bestScore = currentScore;
  
  // Sample top N most promising Order moves
  const allMoves = getAllLegalMoves(state);
  const topMoves = selectTopMovesByQuickHeuristic(state, allMoves, 50);
  
  for (const move of topMoves) {
    const newState = applyMove(state, move);
    const score = calculateFullBoardScore(newState);
    
    if (score > bestScore) {
      bestScore = score;
    }
  }
  
  return bestScore;
}

function selectTopMovesByQuickHeuristic(
  state: BoardState,
  moves: Move[],
  count: number
): Move[] {
  if (moves.length <= count) {
    return moves;
  }
  
  // Score by immediate point delta
  const scoredMoves = moves.map(move => {
    const newState = applyMove(state, move);
    const delta = calculateScoreDelta(state, newState, move);
    return { move, score: delta };
  });
  
  scoredMoves.sort((a, b) => b.score - a.score);
  return scoredMoves.slice(0, count).map(sm => sm.move);
}
```

## Positional Heuristics

```typescript
function getPositionalValue(position: number, phase: string): number {
  const { row, col } = indexToCoords(position);
  
  // Distance from center
  const centerDist = Math.abs(row - 3) + Math.abs(col - 3);
  
  if (phase === 'early') {
    // Early game: prefer center
    return (6 - centerDist) * 2;
  } else if (phase === 'mid') {
    // Mid game: balanced
    return 5;
  } else {
    // Late game: any position matters
    return 3;
  }
}
```

## Complete Spoiler Evaluation

```typescript
function evaluateCompleteSpoilerPlacement(
  gameState: GameState,
  position: number
): number {
  const color = gameState.nextToken;
  const state = gameState.state;
  const phase = determineGamePhase(state);
  
  // Place token
  const newState = applyPlacement(state, position, color);
  
  // Evaluate components
  const patternBreaking = evaluatePatternBreaking(state, position, color);
  const opportunityDenial = evaluateOpportunityDenial(newState, position);
  const bagAwareness = evaluateBagAwareness(state, position, color);
  const orderBestScore = estimateOrderBestMove(newState);
  const positional = getPositionalValue(position, phase);
  
  // Combine (lower = better for Chaos)
  return (
    -patternBreaking * 50 +
    -opportunityDenial * 30 +
    -bagAwareness * 20 +
    orderBestScore * 100 +
    -positional * 10
  );
}
```

---

# Strategy 4: Strategic Gap Creation

## Core Philosophy

Create board positions that inherently resist pattern formation through color fragmentation, symmetry breaking, and awkward gap spacing.

## Algorithm

```typescript
function chaosNextTurn(gameState: GameState): ChaosMove {
  const color = gameState.nextToken;
  const emptyPositions = getEmptyPositions(gameState.state);
  
  let bestPosition = emptyPositions[0]; // Fallback
  let highestEntropy = -Infinity;
  
  for (const position of emptyPositions) {
    const entropyScore = evaluateEntropy(gameState, position, color);
    
    if (entropyScore > highestEntropy) {
      highestEntropy = entropyScore;
      bestPosition = position;
    }
  }
  
  return formatChaosMove(bestPosition);
}

function evaluateEntropy(
  gameState: GameState,
  position: number,
  color: Color
): number {
  const state = gameState.state;
  const phase = determineGamePhase(state);
  
  const fragmentation = evaluateColorFragmentation(state, position, color);
  const symmetry = evaluateSymmetryBreaking(state, position, color);
  const gapSpacing = evaluateGapSpacing(state, position);
  const zoneControl = evaluateZoneControl(state, position, color);
  const edgeStrategy = evaluateEdgeStrategy(state, position, phase);
  
  return (
    fragmentation * 40 +
    symmetry * 35 +
    gapSpacing * 15 +
    zoneControl * 20 +
    edgeStrategy * 10
  );
}
```

## Component 1: Color Fragmentation

```typescript
function evaluateColorFragmentation(
  state: BoardState,
  position: number,
  color: Color
): number {
  const { row, col } = indexToCoords(position);
  
  let fragmentation = 0;
  
  // Analyze row
  const rowColors = getRow(state, row);
  const rowRunsBefore = countColorRuns(rowColors);
  
  rowColors[col] = color;
  const rowRunsAfter = countColorRuns(rowColors);
  
  fragmentation += (rowRunsAfter - rowRunsBefore) * 5;
  
  // Analyze column
  const colColors = getColumn(state, col);
  const colRunsBefore = countColorRuns(colColors);
  
  colColors[row] = color;
  const colRunsAfter = countColorRuns(colColors);
  
  fragmentation += (colRunsAfter - colRunsBefore) * 5;
  
  return fragmentation;
}

function countColorRuns(sequence: Color[]): number {
  // Count the number of color "runs" (contiguous same-color segments)
  let runs = 0;
  let currentRun: Color | null = null;
  
  for (const color of sequence) {
    if (color === '') continue;
    
    if (color !== currentRun) {
      runs++;
      currentRun = color;
    }
  }
  
  return runs;
}
```

**Examples:**
- `RRG___` → place `B` at 3 → `RRGB__` (increases runs from 2 to 3) ✓
- `RRG___` → place `G` at 3 → `RRGG__` (keeps runs at 2) ✗
- More runs = more fragmentation = harder to form palindromes

## Component 2: Symmetry Breaking

```typescript
function evaluateSymmetryBreaking(
  state: BoardState,
  position: number,
  color: Color
): number {
  const { row, col } = indexToCoords(position);
  
  let symmetryBreak = 0;
  
  // Check horizontal mirror
  const mirrorCol = 6 - col;
  const mirrorColor = state[row * 7 + mirrorCol];
  
  if (mirrorColor !== '' && mirrorColor !== color) {
    symmetryBreak += 10; // Creates asymmetry
  } else if (mirrorColor === color) {
    symmetryBreak -= 5; // Creates symmetry (bad for Chaos)
  }
  
  // Check vertical mirror
  const mirrorRow = 6 - row;
  const mirrorColorV = state[mirrorRow * 7 + col];
  
  if (mirrorColorV !== '' && mirrorColorV !== color) {
    symmetryBreak += 10;
  } else if (mirrorColorV === color) {
    symmetryBreak -= 5;
  }
  
  // Check if placement breaks existing patterns
  symmetryBreak += checkPalindromeBreaking(state, position, color);
  
  return symmetryBreak;
}

function checkPalindromeBreaking(
  state: BoardState,
  position: number,
  color: Color
): number {
  const { row, col } = indexToCoords(position);
  
  let breaking = 0;
  
  // Check row for potential palindromes
  const rowColors = getRow(state, row);
  
  for (let len = 2; len <= 7; len++) {
    for (let start = 0; start <= 7 - len; start++) {
      if (col < start || col >= start + len) continue;
      
      const segment = rowColors.slice(start, start + len);
      const posInSegment = col - start;
      
      // Check if this would break a potential palindrome
      if (wouldBreakPalindrome(segment, posInSegment, color)) {
        breaking += len;
      }
    }
  }
  
  // Same for column
  const colColors = getColumn(state, col);
  
  for (let len = 2; len <= 7; len++) {
    for (let start = 0; start <= 7 - len; start++) {
      if (row < start || row >= start + len) continue;
      
      const segment = colColors.slice(start, start + len);
      const posInSegment = row - start;
      
      if (wouldBreakPalindrome(segment, posInSegment, color)) {
        breaking += len;
      }
    }
  }
  
  return breaking;
}

function wouldBreakPalindrome(
  segment: Color[],
  position: number,
  color: Color
): boolean {
  const len = segment.length;
  const mirror = len - 1 - position;
  
  // If position is empty and mirror has a color
  if (segment[position] === '' && segment[mirror] !== '') {
    // Placing different color breaks potential palindrome
    return color !== segment[mirror];
  }
  
  return false;
}
```

## Component 3: Gap Spacing

```typescript
function evaluateGapSpacing(state: BoardState, position: number): number {
  const { row, col } = indexToCoords(position);
  
  let gapScore = 0;
  
  // Count empty spaces in each direction
  const leftGap = countEmptyToLeft(state, row, col);
  const rightGap = countEmptyToRight(state, row, col);
  const upGap = countEmptyUp(state, row, col);
  const downGap = countEmptyDown(state, row, col);
  
  // Odd-sized gaps are awkward for Order
  if (leftGap % 2 === 1) gapScore += 5;
  if (rightGap % 2 === 1) gapScore += 5;
  if (upGap % 2 === 1) gapScore += 5;
  if (downGap % 2 === 1) gapScore += 5;
  
  return gapScore;
}

function countEmptyToLeft(state: BoardState, row: number, col: number): number {
  let count = 0;
  for (let c = col - 1; c >= 0; c--) {
    if (state[row * 7 + c] !== '') break;
    count++;
  }
  return count;
}

function countEmptyToRight(state: BoardState, row: number, col: number): number {
  let count = 0;
  for (let c = col + 1; c < 7; c++) {
    if (state[row * 7 + c] !== '') break;
    count++;
  }
  return count;
}

function countEmptyUp(state: BoardState, row: number, col: number): number {
  let count = 0;
  for (let r = row - 1; r >= 0; r--) {
    if (state[r * 7 + col] !== '') break;
    count++;
  }
  return count;
}

function countEmptyDown(state: BoardState, row: number, col: number): number {
  let count = 0;
  for (let r = row + 1; r < 7; r++) {
    if (state[r * 7 + col] !== '') break;
    count++;
  }
  return count;
}
```

**Rationale:** Odd gaps (1, 3, 5) are harder for Order to fill productively:
- Gap of 1: Can place one piece, but limited pattern options
- Gap of 2: Can create AB or AA patterns easily
- Gap of 3: Can place pieces but odd symmetry is awkward
- Gap of 4: Good for ABBA type patterns

## Component 4: Zone Control

```typescript
function evaluateZoneControl(
  state: BoardState,
  position: number,
  color: Color
): number {
  // Divide board into 9 zones (3x3 regions)
  const zone = getZone(position);
  const zoneColors = getColorsInZone(state, zone);
  
  // Prefer positions that maximize color diversity in zone
  if (!zoneColors.has(color)) {
    return 8; // New color in zone = good diversity
  } else {
    const colorCount = Array.from(zoneColors.values()).filter(c => c === color).length;
    return Math.max(0, 5 - colorCount); // Diminishing returns
  }
}

function getZone(position: number): number {
  // Zones are numbered 0-8:
  // 0 1 2
  // 3 4 5
  // 6 7 8
  
  const { row, col } = indexToCoords(position);
  const zoneRow = Math.floor(row / 2.33); // 0-6 -> 0-2
  const zoneCol = Math.floor(col / 2.33);
  
  return Math.min(zoneRow, 2) * 3 + Math.min(zoneCol, 2);
}

function getColorsInZone(state: BoardState, zone: number): Set<Color> {
  const colors = new Set<Color>();
  
  const zoneRow = Math.floor(zone / 3);
  const zoneCol = zone % 3;
  
  const rowStart = zoneRow * 2;
  const rowEnd = Math.min(rowStart + 3, 7);
  const colStart = zoneCol * 2;
  const colEnd = Math.min(colStart + 3, 7);
  
  for (let r = rowStart; r < rowEnd; r++) {
    for (let c = colStart; c < colEnd; c++) {
      const color = state[r * 7 + c];
      if (color !== '') {
        colors.add(color);
      }
    }
  }
  
  return colors;
}
```

## Component 5: Edge vs Center Strategy

```typescript
function evaluateEdgeStrategy(
  state: BoardState,
  position: number,
  phase: string
): number {
  const { row, col } = indexToCoords(position);
  const isEdge = row === 0 || row === 6 || col === 0 || col === 6;
  const isCorner = (row === 0 || row === 6) && (col === 0 || col === 6);
  const isCenter = row >= 2 && row <= 4 && col >= 2 && col <= 4;
  
  if (phase === 'early') {
    // Early game: control center to limit Order's flexibility
    if (isCenter) return 10;
    if (isEdge) return 5;
    if (isCorner) return 3;
    return 7;
  } else if (phase === 'mid') {
    // Mid game: balanced approach
    if (isCenter) return 8;
    if (isEdge) return 6;
    if (isCorner) return 4;
    return 7;
  } else {
    // Late game: edges become more important
    if (isEdge) return 10;
    if (isCenter) return 8;
    if (isCorner) return 7;
    return 6;
  }
}
```

---

# Hybrid Strategy Implementation

## Combining Both Approaches

```typescript
function chaosNextTurnHybrid(gameState: GameState): ChaosMove {
  const color = gameState.nextToken;
  const phase = determineGamePhase(gameState.state);
  const emptyPositions = getEmptyPositions(gameState.state);
  
  // Determine weights based on game phase
  let spoilerWeight: number;
  let gapWeight: number;
  
  switch (phase) {
    case 'early':
      spoilerWeight = 0.4;
      gapWeight = 0.6;
      break;
    case 'mid':
      spoilerWeight = 0.6;
      gapWeight = 0.4;
      break;
    case 'late':
      spoilerWeight = 0.8;
      gapWeight = 0.2;
      break;
  }
  
  let bestPosition = emptyPositions[0];
  let bestScore = -Infinity;
  
  for (const position of emptyPositions) {
    // Evaluate with Spoiler strategy (minimize Order's potential)
    const spoilerScore = -evaluateCompleteSpoilerPlacement(gameState, position);
    
    // Evaluate with Gap Creation strategy (maximize entropy)
    const gapScore = evaluateEntropy(gameState, position, color);
    
    // Normalize scores (both should be roughly 0-100 range)
    const normalizedSpoiler = spoilerScore / 100;
    const normalizedGap = gapScore / 100;
    
    // Weighted combination
    const combinedScore = 
      normalizedSpoiler * spoilerWeight +
      normalizedGap * gapWeight;
    
    if (combinedScore > bestScore) {
      bestScore = combinedScore;
      bestPosition = position;
    }
  }
  
  return formatChaosMove(bestPosition);
}
```

## Strategy Selection Logic

```typescript
function selectChaosStrategy(gameState: GameState): 'spoiler' | 'gap' | 'hybrid' {
  const phase = determineGamePhase(gameState.state);
  const emptyCount = getEmptyPositions(gameState.state).length;
  const moveCount = getAllLegalMoves(gameState.state).length;
  
  // If Order has many strong moves, use Spoiler
  if (moveCount > 500 && phase !== 'early') {
    return 'spoiler';
  }
  
  // Early game with few patterns: use Gap Creation
  if (phase === 'early' && emptyCount > 35) {
    return 'gap';
  }
  
  // Default: hybrid
  return 'hybrid';
}
```

---

# Performance Analysis

## Complexity Comparison

| Strategy | Positions | Per Position | Total Operations | Expected Time |
|----------|-----------|--------------|------------------|---------------|
| Spoiler | ~40 | ~500 | ~20,000 | 0.5-1.2s |
| Gap Creation | ~40 | ~50 | ~2,000 | 0.1-0.3s |
| Hybrid | ~40 | ~300 | ~12,000 | 0.3-0.8s |

## Pros & Cons

### Spoiler Strategy

✅ **Advantages:**
- Directly opposes Order's plans
- Considers Order's best moves
- Effective against strong Order players
- Good in mid-late game

❌ **Disadvantages:**
- Computationally expensive
- May create accidental patterns
- Reactive (not proactive)
- Slower response time

### Gap Creation Strategy

✅ **Advantages:**
- Very fast (0.1-0.3s)
- Proactive disruption
- Creates long-term problems
- Good in early game

❌ **Disadvantages:**
- Doesn't directly consider Order's threats
- May miss critical blocks
- Less effective late game
- Requires tuning

### Hybrid Strategy

✅ **Advantages:**
- Balanced approach
- Adapts to game phase
- Good all-around performance
- Reasonable speed (0.3-0.8s)

❌ **Disadvantages:**
- More complex to tune
- Neither strategy fully optimized
- Potential for conflicting signals

---

# Testing Strategy

## Unit Tests

```typescript
test('spoiler disrupts forming palindrome', () => {
  const gameState = createTestState([
    'R', '', 'R', '', '', '', '',
    // Order could complete RGR or RRR
  ]);
  gameState.nextToken = 'B';
  
  const move = chaosNextTurn(gameState);
  
  // Should place B in position to break potential patterns
  expect(move.move).toBe('a2'); // Breaks the R_R pattern
});

test('gap creation maximizes fragmentation', () => {
  const gameState = createTestState([
    'R', 'R', '', '', '', '', '',
    // Has RR run
  ]);
  gameState.nextToken = 'G';
  
  const move = chaosGapCreationBot(gameState);
  
  // Should place to break the run
  expect(move.move).toBe('a3'); // Creates R-R-G fragmentation
});

test('hybrid strategy completes within time', () => {
  const complexState = createComplexState();
  const start = Date.now();
  
  chaosNextTurnHybrid(complexState);
  
  const elapsed = Date.now() - start;
  expect(elapsed).toBeLessThan(1200);
});
```

## Integration Tests

```typescript
test('chaos bot limits Order score', () => {
  const finalScore = playFullGame(
    orderHeuristicBot,
    chaosHybridBot
  );
  
  // Chaos should keep Order below average
  expect(finalScore).toBeLessThan(80);
});

test('chaos bot vs greedy Order', () => {
  const finalScore = playFullGame(
    orderGreedyBot, // Simple greedy strategy
    chaosHybridBot
  );
  
  expect(finalScore).toBeLessThan(60); // Should strongly limit greedy play
});
```

---

# Recommendations

## Production Configuration

```typescript
const CHAOS_CONFIG = {
  strategy: 'hybrid',
  spoilerWeight: 0.6,
  gapWeight: 0.4,
  adaptToPhase: true,
  maxSimulations: 50, // For Spoiler component
  timeLimit: 1.5
};
```

## Strategy Selection Guide

| Situation | Recommended Strategy | Rationale |
|-----------|---------------------|-----------|
| Early game (<16 pieces) | Gap Creation or Hybrid (60% Gap) | Build difficult structure |
| Mid game (16-34 pieces) | Hybrid (60% Spoiler) | Balance disruption + structure |
| Late game (35+ pieces) | Spoiler or Hybrid (80% Spoiler) | Critical to block Order |
| Fast play required | Gap Creation only | Fastest response |
| vs Strong Order bot | Spoiler or Hybrid (70% Spoiler) | Need active defense |
| vs Weak Order bot | Gap Creation | Structural problems sufficient |

## Future Enhancements

1. **Machine Learning:** Train weights from game data
2. **Pattern Library:** Pre-compute high-risk patterns to block
3. **Opening Book:** Pre-defined good Chaos placements for first 5-10 moves
4. **Adaptive Weighting:** Adjust weights based on Order's playing style
5. **Monte Carlo:** Sample random Order responses instead of simulating best move
