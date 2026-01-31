# Common Components Specification

Shared utilities, evaluators, and scoring algorithms used by all bot implementations.

## Interface Definitions

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

interface Coordinates {
  row: number; // 0-6
  col: number; // 0-6
}

interface Move {
  from: number; // board index
  to: number;   // board index
}

interface OrderMove {
  move: string; // "a1-b1" or "pass"
}

interface ChaosMove {
  move: Position; // "a1"
}
```

---

## Coordinate System Utilities

### Board Indexing
- **Rows:** a-g (top to bottom) → 0-6 internally
- **Columns:** 1-7 (left to right) → 0-6 internally
- **Array index:** `row * 7 + col` where row/col are 0-indexed
- Example: "c4" → row=2, col=3 → index=17

### Core Functions

```typescript
function positionToIndex(pos: Position): number {
  // "c4" -> 17
  const row = pos.charCodeAt(0) - 'a'.charCodeAt(0); // c -> 2
  const col = parseInt(pos[1]) - 1; // 4 -> 3
  return row * 7 + col;
}

function indexToPosition(idx: number): Position {
  // 17 -> "c4"
  const row = Math.floor(idx / 7);
  const col = idx % 7;
  return String.fromCharCode('a'.charCodeAt(0) + row) + (col + 1);
}

function indexToCoords(idx: number): Coordinates {
  return {
    row: Math.floor(idx / 7),
    col: idx % 7
  };
}

function coordsToIndex(row: number, col: number): number {
  return row * 7 + col;
}

function isValidCoord(row: number, col: number): boolean {
  return row >= 0 && row < 7 && col >= 0 && col < 7;
}
```

---

## Move Generation

### Legal Move Generator

```typescript
const DIRECTIONS = [
  { dRow: -1, dCol: 0 },  // Up
  { dRow: 1, dCol: 0 },   // Down
  { dRow: 0, dCol: -1 },  // Left
  { dRow: 0, dCol: 1 }    // Right
];

function* generateLegalMoves(state: BoardState): Generator<Move> {
  for (let fromIdx = 0; fromIdx < 49; fromIdx++) {
    if (state[fromIdx] === '') continue; // No piece here
    
    const { row: fromRow, col: fromCol } = indexToCoords(fromIdx);
    
    // Check all 4 rook-style directions
    for (const direction of DIRECTIONS) {
      let row = fromRow;
      let col = fromCol;
      
      while (true) {
        row += direction.dRow;
        col += direction.dCol;
        
        if (!isValidCoord(row, col)) break; // Out of bounds
        
        const toIdx = coordsToIndex(row, col);
        if (state[toIdx] !== '') break; // Blocked by another piece
        
        yield { from: fromIdx, to: toIdx };
      }
    }
  }
}

function getAllLegalMoves(state: BoardState): Move[] {
  return Array.from(generateLegalMoves(state));
}

function getLegalMovesForPiece(state: BoardState, fromIdx: number): Move[] {
  const moves: Move[] = [];
  const { row: fromRow, col: fromCol } = indexToCoords(fromIdx);
  
  if (state[fromIdx] === '') return moves; // No piece here
  
  for (const direction of DIRECTIONS) {
    let row = fromRow;
    let col = fromCol;
    
    while (true) {
      row += direction.dRow;
      col += direction.dCol;
      
      if (!isValidCoord(row, col)) break;
      
      const toIdx = coordsToIndex(row, col);
      if (state[toIdx] !== '') break;
      
      moves.push({ from: fromIdx, to: toIdx });
    }
  }
  
  return moves;
}
```

### Move Application

```typescript
function applyMove(state: BoardState, move: Move): BoardState {
  const newState = [...state];
  newState[move.to] = newState[move.from];
  newState[move.from] = '';
  return newState;
}

function applyPlacement(state: BoardState, position: number, color: Color): BoardState {
  const newState = [...state];
  newState[position] = color;
  return newState;
}

function formatOrderMove(move: Move | null): OrderMove {
  if (move === null) {
    return { move: "pass" };
  }
  
  const from = indexToPosition(move.from);
  const to = indexToPosition(move.to);
  return { move: `${from}-${to}` };
}

function formatChaosMove(position: number): ChaosMove {
  return { move: indexToPosition(position) };
}
```

---

## Scoring Algorithm

### Complete Board Scoring

```typescript
function calculateFullBoardScore(state: BoardState): number {
  let totalScore = 0;
  
  // Score all 7 rows
  for (let row = 0; row < 7; row++) {
    const rowColors = getRow(state, row);
    totalScore += scoreSequence(rowColors);
  }
  
  // Score all 7 columns
  for (let col = 0; col < 7; col++) {
    const colColors = getColumn(state, col);
    totalScore += scoreSequence(colColors);
  }
  
  return totalScore;
}

function getRow(state: BoardState, row: number): Color[] {
  const colors: Color[] = [];
  for (let col = 0; col < 7; col++) {
    colors.push(state[row * 7 + col]);
  }
  return colors;
}

function getColumn(state: BoardState, col: number): Color[] {
  const colors: Color[] = [];
  for (let row = 0; row < 7; row++) {
    colors.push(state[row * 7 + col]);
  }
  return colors;
}
```

### Sequence Scoring

```typescript
function scoreSequence(sequence: Color[]): number {
  // Work only with filled positions
  const filled = sequence.filter(c => c !== '');
  
  if (filled.length < 2) return 0;
  
  let totalScore = 0;
  
  // Check all possible palindrome substrings
  for (let len = 2; len <= filled.length; len++) {
    for (let start = 0; start <= filled.length - len; start++) {
      const substring = filled.slice(start, start + len);
      
      if (isPalindrome(substring)) {
        totalScore += scorePalindrome(substring);
      }
    }
  }
  
  return totalScore;
}

function isPalindrome(seq: Color[]): boolean {
  const len = seq.length;
  for (let i = 0; i < Math.floor(len / 2); i++) {
    if (seq[i] !== seq[len - 1 - i]) {
      return false;
    }
  }
  return true;
}
```

### Pattern Recognition & Scoring

```typescript
function getPalindromeSignature(seq: Color[]): string {
  // Convert concrete pattern to abstract signature
  // Example: ['R', 'G', 'R'] -> "ABA"
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

function scorePalindrome(seq: Color[]): number {
  const signature = getPalindromeSignature(seq);
  return PALINDROME_SCORES[signature] || seq.length;
}

// Complete scoring table from game rules
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

### Optimized Scoring

```typescript
// Cache for repeated sequence scoring
const sequenceScoreCache = new Map<string, number>();

function scoreSequenceCached(sequence: Color[]): number {
  const key = sequence.join('');
  
  if (sequenceScoreCache.has(key)) {
    return sequenceScoreCache.get(key)!;
  }
  
  const score = scoreSequence(sequence);
  
  // Limit cache size to prevent memory issues
  if (sequenceScoreCache.size > 1000) {
    const firstKey = sequenceScoreCache.keys().next().value;
    sequenceScoreCache.delete(firstKey);
  }
  
  sequenceScoreCache.set(key, score);
  return score;
}

// Incremental scoring: only rescore affected rows/columns
function calculateScoreDelta(
  oldState: BoardState,
  newState: BoardState,
  move: Move
): number {
  const { row: fromRow, col: fromCol } = indexToCoords(move.from);
  const { row: toRow, col: toCol } = indexToCoords(move.to);
  
  // Calculate old score for affected rows/cols
  let oldScore = 0;
  oldScore += scoreSequence(getRow(oldState, fromRow));
  oldScore += scoreSequence(getColumn(oldState, fromCol));
  
  if (toRow !== fromRow) {
    oldScore += scoreSequence(getRow(oldState, toRow));
  }
  if (toCol !== fromCol) {
    oldScore += scoreSequence(getColumn(oldState, toCol));
  }
  
  // Calculate new score for affected rows/cols
  let newScore = 0;
  newScore += scoreSequence(getRow(newState, fromRow));
  newScore += scoreSequence(getColumn(newState, fromCol));
  
  if (toRow !== fromRow) {
    newScore += scoreSequence(getRow(newState, toRow));
  }
  if (toCol !== fromCol) {
    newScore += scoreSequence(getColumn(newState, toCol));
  }
  
  return newScore - oldScore;
}
```

---

## Board Analysis Utilities

### Bag Inference

```typescript
function inferRemainingColors(state: BoardState): Map<Color, number> {
  const remaining = new Map<Color, number>([
    ['R', 7], ['G', 7], ['Y', 7], ['B', 7],
    ['W', 7], ['K', 7], ['P', 7]
  ]);
  
  for (const color of state) {
    if (color !== '') {
      remaining.set(color, (remaining.get(color) || 0) - 1);
    }
  }
  
  return remaining;
}

function getColorDistribution(state: BoardState): Map<Color, number> {
  const distribution = new Map<Color, number>();
  
  for (const color of state) {
    if (color !== '') {
      distribution.set(color, (distribution.get(color) || 0) + 1);
    }
  }
  
  return distribution;
}
```

### Game Phase Detection

```typescript
function determineGamePhase(state: BoardState): 'early' | 'mid' | 'late' {
  const filledCount = state.filter(c => c !== '').length;
  
  if (filledCount < 16) return 'early';   // 0-15 pieces
  if (filledCount < 35) return 'mid';     // 16-34 pieces
  return 'late';                           // 35-49 pieces
}

function getEmptyPositions(state: BoardState): number[] {
  const empty: number[] = [];
  for (let i = 0; i < 49; i++) {
    if (state[i] === '') {
      empty.push(i);
    }
  }
  return empty;
}

function getFilledPositions(state: BoardState): number[] {
  const filled: number[] = [];
  for (let i = 0; i < 49; i++) {
    if (state[i] !== '') {
      filled.push(i);
    }
  }
  return filled;
}
```

### Pattern Analysis

```typescript
interface PatternPotential {
  almostComplete: number;  // 1 move away
  twoMove: number;         // 2 moves away
  threeMove: number;       // 3 moves away
}

function analyzePatternPotential(state: BoardState): PatternPotential {
  let almostComplete = 0;
  let twoMove = 0;
  let threeMove = 0;
  
  // Check all rows
  for (let row = 0; row < 7; row++) {
    const rowColors = getRow(state, row);
    const potential = analyzeSequencePotential(rowColors);
    almostComplete += potential.almostComplete;
    twoMove += potential.twoMove;
    threeMove += potential.threeMove;
  }
  
  // Check all columns
  for (let col = 0; col < 7; col++) {
    const colColors = getColumn(state, col);
    const potential = analyzeSequencePotential(colColors);
    almostComplete += potential.almostComplete;
    twoMove += potential.twoMove;
    threeMove += potential.threeMove;
  }
  
  return { almostComplete, twoMove, threeMove };
}

function analyzeSequencePotential(sequence: Color[]): PatternPotential {
  let almostComplete = 0;
  let twoMove = 0;
  let threeMove = 0;
  
  const emptyCount = sequence.filter(c => c === '').length;
  
  // Examples of patterns:
  // R_R (1 empty between same colors) = almostComplete
  // R__R (2 empties) = twoMove
  // RG_GR (1 empty, forms palindrome) = almostComplete
  
  // Simplified heuristic: count based on empty positions
  if (emptyCount === 1) {
    almostComplete += checkNearPalindromes(sequence);
  } else if (emptyCount === 2) {
    twoMove += checkNearPalindromes(sequence);
  } else if (emptyCount === 3) {
    threeMove += checkNearPalindromes(sequence);
  }
  
  return { almostComplete, twoMove, threeMove };
}

function checkNearPalindromes(sequence: Color[]): number {
  // Check if filling empties could create palindromes
  let count = 0;
  
  // Simple check: look for patterns like X_X, XY_YX, etc.
  const filled = sequence.filter(c => c !== '');
  
  if (filled.length >= 2) {
    // Check if first and last match (potential palindrome)
    if (filled[0] === filled[filled.length - 1]) {
      count++;
    }
  }
  
  return count;
}
```

### Board Control Metrics

```typescript
function calculateBoardControl(state: BoardState): number {
  const centerControl = calculateCenterControl(state);
  const edgeControl = calculateEdgeControl(state);
  const connectedGroups = calculateConnectedGroups(state);
  
  return centerControl * 2 + edgeControl * 1 + connectedGroups * 1.5;
}

function calculateCenterControl(state: BoardState): number {
  // Premium center squares: c4, d4, c5, d5
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
  
  return control;
}

function calculateEdgeControl(state: BoardState): number {
  let control = 0;
  
  // Check all edge positions
  for (let i = 0; i < 49; i++) {
    const { row, col } = indexToCoords(i);
    const isEdge = row === 0 || row === 6 || col === 0 || col === 6;
    
    if (isEdge && state[i] !== '') {
      control++;
    }
  }
  
  return control;
}

function calculateConnectedGroups(state: BoardState): number {
  let groups = 0;
  
  // Count same-color pieces on same row/column
  for (let row = 0; row < 7; row++) {
    const rowColors = getRow(state, row);
    groups += countColorGroups(rowColors);
  }
  
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
  
  // Count colors that appear 2+ times
  let groups = 0;
  for (const count of colorCounts.values()) {
    if (count >= 2) {
      groups++;
    }
  }
  
  return groups;
}
```

---

## Performance Utilities

### Time Management

```typescript
class TimeManager {
  private startTime: number;
  private timeLimit: number; // milliseconds
  
  constructor(timeLimitSeconds: number = 2.0) {
    this.timeLimit = timeLimitSeconds * 1000;
    this.startTime = Date.now();
  }
  
  reset(): void {
    this.startTime = Date.now();
  }
  
  elapsed(): number {
    return Date.now() - this.startTime;
  }
  
  remaining(): number {
    return this.timeLimit - this.elapsed();
  }
  
  hasTimeRemaining(): boolean {
    return this.elapsed() < this.timeLimit;
  }
  
  shouldStop(): boolean {
    return this.elapsed() >= this.timeLimit;
  }
}
```

### Caching Utilities

```typescript
class LRUCache<K, V> {
  private cache: Map<K, V>;
  private maxSize: number;
  
  constructor(maxSize: number = 1000) {
    this.cache = new Map();
    this.maxSize = maxSize;
  }
  
  get(key: K): V | undefined {
    if (!this.cache.has(key)) {
      return undefined;
    }
    
    // Move to end (most recently used)
    const value = this.cache.get(key)!;
    this.cache.delete(key);
    this.cache.set(key, value);
    
    return value;
  }
  
  set(key: K, value: V): void {
    // Delete if exists (to move to end)
    if (this.cache.has(key)) {
      this.cache.delete(key);
    }
    
    // Add to end
    this.cache.set(key, value);
    
    // Remove oldest if over limit
    if (this.cache.size > this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
  }
  
  clear(): void {
    this.cache.clear();
  }
  
  size(): number {
    return this.cache.size;
  }
}
```

---

## Constants

```typescript
export const COLORS: Color[] = ['R', 'G', 'Y', 'B', 'W', 'K', 'P'];
export const BOARD_SIZE = 7;
export const TOTAL_SQUARES = 49;
export const PIECES_PER_COLOR = 7;
export const TOTAL_PIECES = 49;
```
