# Entropy Bot - Master Implementation Plan

This document provides the overall roadmap for implementing the Entropy bot, showing how all specification files connect and the recommended development phases.

## Specification Files Overview

| File | Purpose | Dependencies |
|------|---------|--------------|
| `COMMON_COMPONENTS.md` | Shared utilities, scoring, move generation | None (foundation) |
| `ORDER_MINIMAX.md` | Order bot using minimax with alpha-beta pruning | COMMON_COMPONENTS |
| `ORDER_HEURISTIC.md` | Order bot using heuristic evaluation | COMMON_COMPONENTS |
| `CHAOS_STRATEGIES.md` | Chaos bots (Spoiler + Gap Creation) | COMMON_COMPONENTS |
| `IMPLEMENTATION_PLAN.md` | This file - master roadmap | All above |

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Bot Entry Points                        │
│  orderNextTurn(gameState) / chaosNextTurn(gameState)       │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┴──────────────┐
        │                             │
┌───────▼────────┐           ┌────────▼────────┐
│  Order Bots    │           │  Chaos Bots     │
│                │           │                 │
│ • Minimax      │           │ • Spoiler       │
│ • Heuristic    │           │ • Gap Creation  │
│ • Hybrid       │           │ • Hybrid        │
└───────┬────────┘           └────────┬────────┘
        │                             │
        └──────────────┬──────────────┘
                       │
        ┌──────────────▼──────────────────┐
        │    Common Components            │
        │                                 │
        │ • Scoring Algorithm             │
        │ • Move Generation               │
        │ • Board Analysis Utilities      │
        │ • Pattern Detection             │
        │ • Coordinate System             │
        │ • Performance Utilities         │
        └─────────────────────────────────┘
```

## Recommended File Structure

```
entropy-bot/
├── src/
│   ├── common/
│   │   ├── types.ts              # GameState, Color, Move interfaces
│   │   ├── coordinates.ts        # Position ↔ index conversions
│   │   ├── scoring.ts            # Palindrome detection & scoring
│   │   ├── moveGeneration.ts    # Legal move generation
│   │   ├── boardAnalysis.ts     # Pattern potential, bag inference
│   │   ├── utilities.ts         # TimeManager, LRUCache
│   │   └── constants.ts         # COLORS, BOARD_SIZE, scores
│   │
│   ├── order/
│   │   ├── minimax.ts           # Minimax with alpha-beta
│   │   ├── heuristic.ts         # Heuristic evaluation
│   │   ├── evaluation.ts        # Shared evaluation functions
│   │   └── index.ts             # orderNextTurn() export
│   │
│   ├── chaos/
│   │   ├── spoiler.ts           # Spoiler strategy
│   │   ├── gapCreation.ts       # Gap creation strategy
│   │   ├── hybrid.ts            # Hybrid chaos strategy
│   │   └── index.ts             # chaosNextTurn() export
│   │
│   └── index.ts                 # Main exports
│
├── tests/
│   ├── common/
│   │   ├── scoring.test.ts
│   │   ├── moveGeneration.test.ts
│   │   └── coordinates.test.ts
│   │
│   ├── order/
│   │   ├── minimax.test.ts
│   │   └── heuristic.test.ts
│   │
│   ├── chaos/
│   │   ├── spoiler.test.ts
│   │   └── gapCreation.test.ts
│   │
│   └── integration/
│       ├── fullGame.test.ts
│       └── performance.test.ts
│
├── example_game_states.json     # Test scenarios
├── specs/                       # Specification documents
├── package.json
├── tsconfig.json
└── README.md
```

---

## Phase 1: Foundation (Common Components)

**Goal:** Implement and test all shared utilities that both Order and Chaos bots will use.

**Estimated Time:** 3-4 days

### Tasks

#### 1.1 Type Definitions (`types.ts`)
- [ ] Define `Color`, `BoardState`, `Position` types
- [ ] Define `GameState` interface
- [ ] Define `Move`, `OrderMove`, `ChaosMove` interfaces
- [ ] Define `Coordinates` interface

**Test:** Type checking compiles without errors

#### 1.2 Coordinate System (`coordinates.ts`)
- [ ] Implement `positionToIndex("c4" → 17)`
- [ ] Implement `indexToPosition(17 → "c4")`
- [ ] Implement `indexToCoords(17 → {row: 2, col: 3})`
- [ ] Implement `coordsToIndex({row: 2, col: 3} → 17)`
- [ ] Implement `isValidCoord(row, col)`

**Tests:**
```typescript
test('positionToIndex converts correctly', () => {
  expect(positionToIndex('a1')).toBe(0);
  expect(positionToIndex('c4')).toBe(17);
  expect(positionToIndex('g7')).toBe(48);
});

test('round-trip conversion', () => {
  for (let i = 0; i < 49; i++) {
    expect(positionToIndex(indexToPosition(i))).toBe(i);
  }
});
```

#### 1.3 Scoring Algorithm (`scoring.ts`)

**Priority:** CRITICAL - Everything depends on this

- [ ] Implement `calculateFullBoardScore(state)`
- [ ] Implement `getRow(state, row)`
- [ ] Implement `getColumn(state, col)`
- [ ] Implement `scoreSequence(colors[])`
- [ ] Implement `isPalindrome(colors[])`
- [ ] Implement `getPalindromeSignature(colors[])`
- [ ] Implement `scorePalindrome(colors[])`
- [ ] Define `PALINDROME_SCORES` lookup table (30 patterns)
- [ ] Implement `calculateScoreDelta(oldState, newState, move)` (optimization)

**Tests:**
```typescript
test('scores AA correctly', () => {
  const row = ['R', 'R', '', '', '', '', ''];
  expect(scoreSequence(row)).toBe(2);
});

test('scores AAA correctly', () => {
  const row = ['R', 'R', 'R', '', '', '', ''];
  expect(scoreSequence(row)).toBe(7 + 2 + 2); // AAA + AA + AA
});

test('scores ABCBA correctly', () => {
  const row = ['R', 'G', 'Y', 'G', 'R', '', ''];
  expect(scoreSequence(row)).toBe(8 + 3); // ABCBA + BCB
});

test('scores full board', () => {
  const state = [...]; // Known test board
  expect(calculateFullBoardScore(state)).toBe(expectedScore);
});
```

**Validation:** Test against all 30 pattern types from game rules

#### 1.4 Move Generation (`moveGeneration.ts`)
- [ ] Implement `generateLegalMoves(state)` generator
- [ ] Implement `getAllLegalMoves(state)`
- [ ] Implement `getLegalMovesForPiece(state, fromIdx)`
- [ ] Implement `applyMove(state, move)`
- [ ] Implement `applyPlacement(state, position, color)`
- [ ] Implement `formatOrderMove(move)`
- [ ] Implement `formatChaosMove(position)`
- [ ] Define `DIRECTIONS` constant

**Tests:**
```typescript
test('generates rook-style moves', () => {
  const state = Array(49).fill('');
  state[24] = 'R'; // Center piece
  
  const moves = getAllLegalMoves(state);
  expect(moves.length).toBe(24); // 6 up + 6 down + 6 left + 6 right
});

test('moves blocked by other pieces', () => {
  const state = Array(49).fill('');
  state[24] = 'R'; // Center
  state[25] = 'G'; // Right neighbor
  
  const moves = getLegalMovesForPiece(state, 24);
  const rightMoves = moves.filter(m => m.to > 24 && m.to < 28);
  expect(rightMoves.length).toBe(0); // Blocked
});

test('applies move correctly', () => {
  const state = Array(49).fill('');
  state[0] = 'R';
  
  const newState = applyMove(state, { from: 0, to: 1 });
  expect(newState[0]).toBe('');
  expect(newState[1]).toBe('R');
});
```

#### 1.5 Board Analysis (`boardAnalysis.ts`)
- [ ] Implement `inferRemainingColors(state)`
- [ ] Implement `getColorDistribution(state)`
- [ ] Implement `determineGamePhase(state)`
- [ ] Implement `getEmptyPositions(state)`
- [ ] Implement `getFilledPositions(state)`
- [ ] Implement `analyzePatternPotential(state)`
- [ ] Implement `analyzeSequencePotential(sequence)`
- [ ] Implement `calculateBoardControl(state)`
- [ ] Implement `calculateCenterControl(state)`
- [ ] Implement `calculateEdgeControl(state)`
- [ ] Implement `calculateConnectedGroups(state)`

**Tests:**
```typescript
test('infers remaining colors correctly', () => {
  const state = Array(49).fill('');
  state[0] = 'R';
  state[1] = 'R';
  
  const remaining = inferRemainingColors(state);
  expect(remaining.get('R')).toBe(5);
  expect(remaining.get('G')).toBe(7);
});

test('detects game phase', () => {
  const earlyState = Array(49).fill('');
  for (let i = 0; i < 10; i++) earlyState[i] = 'R';
  expect(determineGamePhase(earlyState)).toBe('early');
  
  const lateState = Array(49).fill('R');
  expect(determineGamePhase(lateState)).toBe('late');
});
```

#### 1.6 Performance Utilities (`utilities.ts`)
- [ ] Implement `TimeManager` class
- [ ] Implement `LRUCache<K, V>` class
- [ ] Add performance monitoring helpers

**Tests:**
```typescript
test('TimeManager tracks time', () => {
  const timer = new TimeManager(1.0);
  expect(timer.hasTimeRemaining()).toBe(true);
  
  // Wait 1 second
  setTimeout(() => {
    expect(timer.hasTimeRemaining()).toBe(false);
  }, 1100);
});

test('LRUCache evicts oldest', () => {
  const cache = new LRUCache<string, number>(2);
  cache.set('a', 1);
  cache.set('b', 2);
  cache.set('c', 3); // Should evict 'a'
  
  expect(cache.get('a')).toBeUndefined();
  expect(cache.get('b')).toBe(2);
  expect(cache.get('c')).toBe(3);
});
```

#### 1.7 Constants (`constants.ts`)
- [ ] Define `COLORS` array
- [ ] Define `BOARD_SIZE`, `TOTAL_SQUARES`, etc.
- [ ] Export all shared constants

### Phase 1 Acceptance Criteria
- ✅ All unit tests pass (>95% coverage)
- ✅ Scoring matches manual calculations for all 30 patterns
- ✅ Move generation produces valid moves only
- ✅ Coordinate conversions are bidirectional
- ✅ Performance: Scoring a full board < 5ms

---

## Phase 2: Order Bot - Heuristic (Simpler First)

**Goal:** Implement the faster, simpler Order bot to validate the foundation.

**Estimated Time:** 2-3 days

**Rationale:** Start with Heuristic bot because:
1. Simpler to implement (no tree search)
2. Validates all common components
3. Provides baseline for comparison
4. Faster for testing

### Tasks

#### 2.1 Core Evaluation (`order/evaluation.ts`)
- [ ] Implement `evaluatePosition(gameState)`
- [ ] Implement `calculatePatternPotential(state)`
- [ ] Implement `calculateChaosOpportunities(state)`
- [ ] Implement `analyzeColorDistribution(state, nextToken)`
- [ ] Implement `calculateFertility(state)`
- [ ] Implement evaluation component weights as constants

**Tests:**
```typescript
test('evaluates scoring position highly', () => {
  const gameState = createScoringState();
  const eval = evaluatePosition(gameState);
  expect(eval).toBeGreaterThan(500); // Has actual score
});

test('penalizes empty board', () => {
  const emptyState = { state: Array(49).fill(''), nextToken: 'R', turn: 'order' };
  const eval = evaluatePosition(emptyState);
  expect(eval).toBeLessThan(100); // Low score, no patterns
});
```

#### 2.2 Heuristic Bot (`order/heuristic.ts`)
- [ ] Implement `orderNextTurn(gameState)`
- [ ] Implement move evaluation loop
- [ ] Implement early termination optimization
- [ ] Implement incremental evaluation
- [ ] Add performance monitoring

**Tests:**
```typescript
test('finds scoring move', () => {
  const gameState = createTestState([
    'R', 'R', '', '', '', '', '',  // Can complete RRR
    // ...
  ]);
  
  const move = orderNextTurn(gameState);
  expect(move.move).toBe('a3-a1'); // Or whatever completes RRR
});

test('chooses pass when no good moves', () => {
  const badState = createBadState();
  const move = orderNextTurn(badState);
  expect(move.move).toBe('pass');
});

test('completes under 2 seconds', () => {
  const complexState = loadGameState(5); // Mid-game from examples
  const start = Date.now();
  
  orderNextTurn(complexState);
  
  const elapsed = Date.now() - start;
  expect(elapsed).toBeLessThan(2000);
});
```

#### 2.3 Integration Testing
- [ ] Test full game simulation
- [ ] Measure average game score
- [ ] Profile performance bottlenecks

### Phase 2 Acceptance Criteria
- ✅ Bot responds in < 1.2s on average
- ✅ Achieves average score > 70 points against random Chaos
- ✅ Never crashes or times out
- ✅ Passes all move legality checks

---

## Phase 3: Chaos Bot - Gap Creation (Simpler First)

**Goal:** Implement faster Chaos strategy to complete basic bot-vs-bot capability.

**Estimated Time:** 2 days

**Rationale:** Gap Creation is faster and doesn't require Order simulation.

### Tasks

#### 3.1 Entropy Evaluators (`chaos/gapCreation.ts`)
- [ ] Implement `evaluateColorFragmentation(state, pos, color)`
- [ ] Implement `evaluateSymmetryBreaking(state, pos, color)`
- [ ] Implement `evaluateGapSpacing(state, pos)`
- [ ] Implement `evaluateZoneControl(state, pos, color)`
- [ ] Implement `evaluateEdgeStrategy(state, pos, phase)`
- [ ] Implement `countColorRuns(sequence)`

**Tests:**
```typescript
test('prefers positions that fragment colors', () => {
  const state = ['R', 'R', 'G', '', '', '', ''];
  const fragmentation1 = evaluateColorFragmentation(state, 3, 'B'); // R-R-G-B
  const fragmentation2 = evaluateColorFragmentation(state, 3, 'R'); // R-R-G-R
  
  expect(fragmentation1).toBeGreaterThan(fragmentation2);
});
```

#### 3.2 Gap Creation Bot (`chaos/gapCreation.ts`)
- [ ] Implement `chaosNextTurn(gameState)`
- [ ] Implement `evaluateEntropy(gameState, pos, color)`
- [ ] Integrate all entropy components
- [ ] Add weight configuration

**Tests:**
```typescript
test('chooses disruptive positions', () => {
  const gameState = createTestState([
    'R', 'R', '', 'R', 'R', '', '',  // Has potential RRBR pattern
  ]);
  gameState.nextToken = 'G';
  
  const move = chaosNextTurn(gameState);
  // Should place to disrupt, not between matching pieces
  expect(move.move).not.toBe('a3'); // Not in good symmetry position
});

test('completes under 0.5 seconds', () => {
  const state = loadGameState(5);
  const start = Date.now();
  
  chaosNextTurn(state);
  
  expect(Date.now() - start).toBeLessThan(500);
});
```

### Phase 3 Acceptance Criteria
- ✅ Bot responds in < 0.5s
- ✅ Order scores < 75 points (disrupts effectively)
- ✅ Creates visibly fragmented boards

---

## Phase 4: Full Game Capability

**Goal:** Create playable bot-vs-bot system with basic strategies.

**Estimated Time:** 1 day

### Tasks

#### 4.1 Main Entry Point (`src/index.ts`)
```typescript
export { orderNextTurn } from './order';
export { chaosNextTurn } from './chaos';
export type { GameState, OrderMove, ChaosMove } from './common/types';
```

#### 4.2 Integration Tests (`tests/integration/`)
- [ ] Implement full game simulator
- [ ] Test Order Heuristic vs Chaos Gap Creation
- [ ] Test 100 games, measure statistics
- [ ] Profile performance

**Tests:**
```typescript
test('completes full game', () => {
  const finalScore = playFullGame(orderNextTurn, chaosNextTurn);
  expect(finalScore).toBeGreaterThan(0);
  expect(finalScore).toBeLessThan(200); // Sanity check
});

test('average score in range', () => {
  const scores = [];
  for (let i = 0; i < 100; i++) {
    scores.push(playFullGame(orderNextTurn, chaosNextTurn));
  }
  
  const avg = scores.reduce((a, b) => a + b) / scores.length;
  expect(avg).toBeGreaterThan(60); // Above poor
  expect(avg).toBeLessThan(100); // Below good (Chaos is effective)
});
```

#### 4.3 Example Runner
- [ ] Create CLI to run example game states
- [ ] Add visualization of board state
- [ ] Add move explanation output

### Phase 4 Acceptance Criteria
- ✅ Can play complete games
- ✅ Average score 60-85 points
- ✅ No crashes in 100-game test
- ✅ All moves are legal

---

## Phase 5: Order Bot - Minimax (Advanced)

**Goal:** Implement stronger Order bot with tree search.

**Estimated Time:** 3-4 days

**Rationale:** Implement after heuristic bot to have baseline for comparison.

### Tasks

#### 5.1 Minimax Core (`order/minimax.ts`)
- [ ] Implement `minimax(state, depth, alpha, beta, isMax)`
- [ ] Implement `maximizingPlayer(...)`
- [ ] Implement `minimizingPlayer(...)`
- [ ] Implement iterative deepening
- [ ] Implement transposition table
- [ ] Implement time management

**Tests:**
```typescript
test('finds forced win', () => {
  const gameState = createForcedWinState(); // RR_ in row, Order's turn
  const move = orderMinimaxBot(gameState);
  
  // Should complete RRR
  expect(move.move).toContain('a3');
});

test('respects time limit', () => {
  const complexState = loadGameState(4);
  const start = Date.now();
  
  orderMinimaxBot(complexState);
  
  expect(Date.now() - start).toBeLessThan(2100); // 2s + 100ms tolerance
});

test('prunes effectively', () => {
  const state = loadGameState(4);
  let nodesEvaluated = 0;
  
  // Add counter to minimax
  orderMinimaxBot(state, { onNodeEval: () => nodesEvaluated++ });
  
  expect(nodesEvaluated).toBeLessThan(50000); // Should prune well
});
```

#### 5.2 Move Ordering (`order/minimax.ts`)
- [ ] Implement `orderMovesByHeuristic(state, moves)`
- [ ] Implement `orderChaosPlacementsByHeuristic(...)`
- [ ] Implement killer move heuristic (optional)

#### 5.3 Depth Strategy
- [ ] Implement `determineSearchDepth(gameState)`
- [ ] Add adaptive depth based on branching factor
- [ ] Add aspiration windows (optional)

### Phase 5 Acceptance Criteria
- ✅ Completes under 2s
- ✅ Finds better moves than heuristic bot (A/B test)
- ✅ Average score > 80 points
- ✅ Effective pruning (< 50k nodes evaluated)

---

## Phase 6: Chaos Bot - Spoiler (Advanced)

**Goal:** Implement reactive Chaos strategy.

**Estimated Time:** 2-3 days

### Tasks

#### 6.1 Spoiler Components (`chaos/spoiler.ts`)
- [ ] Implement `evaluatePatternBreaking(state, pos, color)`
- [ ] Implement `evaluateOpportunityDenial(state, pos)`
- [ ] Implement `evaluateBagAwareness(state, pos, color)`
- [ ] Implement `estimateOrderBestMove(state)`

**Tests:**
```typescript
test('breaks forming palindrome', () => {
  const state = ['R', '', 'R', '', '', '', ''];
  const breaking = evaluatePatternBreaking(state, 1, 'G');
  
  expect(breaking).toBeGreaterThan(5); // High disruption value
});

test('simulates Order response', () => {
  const state = createMidGameState();
  const orderScore = estimateOrderBestMove(state);
  
  expect(orderScore).toBeGreaterThan(0); // Should find scoring moves
});
```

#### 6.2 Spoiler Bot (`chaos/spoiler.ts`)
- [ ] Implement `chaosNextTurn(gameState)`
- [ ] Implement complete spoiler evaluation
- [ ] Optimize Order simulation (limit to top 50 moves)

**Tests:**
```typescript
test('blocks Order best move', () => {
  const gameState = createCriticalState(); // Order can score big
  gameState.nextToken = 'B';
  
  const move = chaosSpoilerBot(gameState);
  
  // Should place to minimize Order's score
  const newState = applyPlacement(gameState.state, positionToIndex(move.move), 'B');
  const orderBestAfter = estimateOrderBestMove(newState);
  
  expect(orderBestAfter).toBeLessThan(60); // Effectively blocked
});

test('completes under 1.5 seconds', () => {
  const state = loadGameState(6);
  const start = Date.now();
  
  chaosSpoilerBot(state);
  
  expect(Date.now() - start).toBeLessThan(1500);
});
```

### Phase 6 Acceptance Criteria
- ✅ Completes under 1.5s
- ✅ Order scores < 70 points (strong disruption)
- ✅ Outperforms Gap Creation in mid-late game

---

## Phase 7: Hybrid Strategies & Tuning

**Goal:** Combine strategies and optimize weights.

**Estimated Time:** 2-3 days

### Tasks

#### 7.1 Chaos Hybrid (`chaos/hybrid.ts`)
- [ ] Implement `chaosNextTurnHybrid(gameState)`
- [ ] Implement dynamic weight adjustment by phase
- [ ] Implement strategy selection logic

**Tests:**
```typescript
test('adapts to game phase', () => {
  const earlyState = createEarlyGameState();
  const lateState = createLateGameState();
  
  // Should use different weights
  const earlyWeights = getWeightsForPhase('early');
  const lateWeights = getWeightsForPhase('late');
  
  expect(earlyWeights.gapWeight).toBeGreaterThan(lateWeights.gapWeight);
});
```

#### 7.2 Order Hybrid (Optional)
- [ ] Implement adaptive strategy (minimax late, heuristic early)

#### 7.3 Weight Tuning
- [ ] Run parameter sweep on evaluation weights
- [ ] Test 100+ games per configuration
- [ ] Find optimal weights for each strategy

**Tuning Process:**
```typescript
const weightConfigs = generateWeightVariations();

for (const config of weightConfigs) {
  const scores = runGames(100, config);
  const avgScore = average(scores);
  
  if (avgScore > bestScore) {
    bestScore = avgScore;
    bestConfig = config;
  }
}
```

### Phase 7 Acceptance Criteria
- ✅ Hybrid bots outperform single strategies
- ✅ Optimal weights identified through testing
- ✅ Documentation of tuning results

---

## Phase 8: Polish & Production Ready

**Goal:** Prepare for production deployment.

**Estimated Time:** 2-3 days

### Tasks

#### 8.1 Documentation
- [ ] Write comprehensive README
- [ ] Add JSDoc comments to all public functions
- [ ] Create API documentation
- [ ] Document configuration options

#### 8.2 Error Handling
- [ ] Add input validation
- [ ] Handle edge cases (empty board, full board)
- [ ] Add graceful degradation for time limits
- [ ] Add logging and debugging utilities

#### 8.3 Performance Optimization
- [ ] Profile hot paths
- [ ] Optimize scoring algorithm
- [ ] Optimize move generation
- [ ] Add caching where beneficial

#### 8.4 Configuration System
```typescript
interface BotConfig {
  orderStrategy: 'minimax' | 'heuristic' | 'hybrid';
  chaosStrategy: 'spoiler' | 'gap' | 'hybrid';
  timeLimit: number;
  weights: WeightConfig;
  debug: boolean;
}
```

#### 8.5 CLI/API Interface
- [ ] Create simple CLI for testing
- [ ] Add game replay capability
- [ ] Add move explanation mode
- [ ] Add performance profiling mode

### Phase 8 Acceptance Criteria
- ✅ All code documented
- ✅ All error cases handled
- ✅ Performance targets met consistently
- ✅ Ready for integration

---

## Testing Strategy

### Test Pyramid

```
         /\
        /  \  E2E Tests (10%)
       /----\  - Full games
      /      \ - 100-game suites
     /--------\ 
    / Integration \ (20%)
   /--------------\  - Bot vs Bot
  /                \ - Multi-move sequences
 /------------------\
/   Unit Tests (70%) \
----------------------
 - Individual functions
 - Edge cases
 - Performance tests
```

### Test Data

Use `example_game_states.json` for:
- Unit test scenarios
- Integration test starting positions
- Performance benchmarks
- Regression testing

### Performance Benchmarks

| Metric | Target | Measured |
|--------|--------|----------|
| Order Heuristic | < 1.2s | TBD |
| Order Minimax | < 2.0s | TBD |
| Chaos Gap Creation | < 0.5s | TBD |
| Chaos Spoiler | < 1.5s | TBD |
| Chaos Hybrid | < 1.0s | TBD |
| Full Board Score | < 5ms | TBD |
| Move Generation | < 10ms | TBD |

### Quality Gates

Before each phase completion:
- ✅ All tests pass
- ✅ Code coverage > 90%
- ✅ Performance targets met
- ✅ No linting errors
- ✅ Documentation complete

---

## Risk Management

### Known Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Scoring algorithm bugs | Medium | Critical | Extensive testing, manual validation |
| Time limit exceeded | Medium | High | Profiling, optimization, time management |
| Minimax too slow | High | Medium | Iterative deepening, depth limits |
| Poor bot performance | Medium | Medium | Weight tuning, strategy comparison |
| Edge case crashes | Low | High | Comprehensive error handling |

### Contingency Plans

**If Order Minimax too slow:**
- Reduce default depth to 1
- Use hybrid approach (heuristic early, minimax late)
- Implement move count pruning more aggressively

**If scoring algorithm incorrect:**
- Add extensive test cases
- Manual validation against game rules
- Create visual debugging tool

**If bot performance poor:**
- Run weight tuning experiments
- Compare against baseline strategies
- Analyze game transcripts for patterns

---

## Success Metrics

### Quantitative

- **Order Bot Average Score:** 75-90 points against hybrid Chaos
- **Chaos Bot Effectiveness:** Limits Order to < 75 points
- **Response Time:** 100% of moves under 2 seconds
- **Test Coverage:** > 90% line coverage
- **Performance:** All benchmarks met

### Qualitative

- **Code Quality:** Clean, maintainable, well-documented
- **Bot Behavior:** Makes reasonable, explainable moves
- **User Experience:** Fast, reliable, predictable
- **Extensibility:** Easy to add new strategies

---

## Timeline Summary

| Phase | Duration | Dependencies | Deliverable |
|-------|----------|--------------|-------------|
| 1. Foundation | 3-4 days | None | Common components, all tests passing |
| 2. Order Heuristic | 2-3 days | Phase 1 | Working Order bot |
| 3. Chaos Gap | 2 days | Phase 1 | Working Chaos bot |
| 4. Integration | 1 day | Phases 2-3 | Full game capability |
| 5. Order Minimax | 3-4 days | Phase 1 | Advanced Order bot |
| 6. Chaos Spoiler | 2-3 days | Phase 1 | Advanced Chaos bot |
| 7. Hybrid & Tuning | 2-3 days | Phases 5-6 | Optimized strategies |
| 8. Polish | 2-3 days | All phases | Production ready |

**Total Estimated Time:** 17-25 days

---

## Getting Started

### Recommended First Steps

1. **Set up project structure**
   ```bash
   npm init
   npm install --save-dev typescript @types/node jest @types/jest
   npx tsc --init
   ```

2. **Create type definitions** (`src/common/types.ts`)
   - Start with interfaces and types
   - Ensure TypeScript compiles

3. **Implement coordinate system** (`src/common/coordinates.ts`)
   - Simple, no dependencies
   - Easy to test
   - Foundation for everything else

4. **Implement scoring algorithm** (`src/common/scoring.ts`)
   - Most critical component
   - Test exhaustively
   - Validate against game rules

5. **Implement move generation** (`src/common/moveGeneration.ts`)
   - Core game logic
   - Many edge cases to test

### Development Workflow

1. **Write specification** (already done!)
2. **Write tests first** (TDD approach)
3. **Implement functionality**
4. **Verify tests pass**
5. **Profile performance**
6. **Refactor if needed**
7. **Document**
8. **Move to next component**

### Daily Progress Tracking

Create `progress.md` to track:
- [ ] Daily completed tasks
- [ ] Current phase
- [ ] Blockers
- [ ] Performance metrics
- [ ] Next steps

---

## Conclusion

This implementation plan provides a clear, phased approach to building the Entropy bot system. Start with the foundation, build incrementally, test thoroughly, and optimize continuously. The modular architecture allows each component to be developed and tested independently while maintaining clean interfaces between components.

**Key Principles:**
1. **Test-driven development:** Write tests before implementation
2. **Incremental progress:** Complete one phase before starting next
3. **Performance first:** Monitor timing from day one
4. **Quality gates:** Don't proceed with failing tests
5. **Document as you go:** Comments and docs prevent technical debt

Good luck with the implementation!
