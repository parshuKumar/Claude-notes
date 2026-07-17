# 119 — Design a Chess Game
## Category: LLD Case Study

---

## What is this?

Chess is the classic "each thing behaves differently" modelling problem. You have one board, two players, and six kinds of piece — and every piece type moves by its **own** rules. A rook slides in straight lines, a bishop on diagonals, a knight jumps in an L.

The design challenge is not the chess rules themselves — it's **where those rules live**. The whole art of this problem is making the `Board` and `Game` code never care *what kind* of piece it's holding. It just asks the piece "can you move here?" and the piece answers for itself. That's polymorphism, and this problem is the purest teaching example of it in LLD.

---

## Why does it matter?

**Interview angle.** Chess (with Parking Lot, Elevator, and Snake & Ladder) is a top-5 object-modelling interview question. The interviewer is watching for one thing above all: do you reach for **polymorphism** (an abstract `Piece` with subclasses), or do you write a giant `switch (piece.type)`? The first answer signals you understand OOP; the second signals you've memorised syntax but not design. Everything else — check detection, special moves — is secondary follow-up.

**Real-work angle.** "A family of things that share an interface but each behave differently" is one of the most common shapes in real software: payment methods, notification channels, file exporters, shipping-rate calculators, database drivers. If you can model chess pieces cleanly, you can model all of those the same way. The anti-pattern (a fat conditional that everyone edits every time a new type appears) is a real maintenance disease, and chess is where you learn to cure it.

---

## The core idea — explained simply

### Analogy: the theatre director who never learned the choreography

Imagine a stage director running a show with six kinds of dancer. A **bad** director memorises every dancer's routine and shouts each step personally: "Rook, go straight! Bishop, go diagonal! Knight, jump!" If a new dancer joins, the director must re-learn everything and rewrite the whole script.

A **good** director just says "Everyone, take your next step." Each dancer *knows their own choreography*. The director doesn't need to know or care how a knight moves — the knight does. Adding a new dancer means teaching *that dancer* their steps; the director's script never changes.

The good director is your `Game` class. The dancers are `Piece` subclasses. "Take your step" is the polymorphic method `canMove(...)`.

| Story element | Chess design concept |
|---|---|
| The director | `Game` / `Board` — orchestrates, never knows piece internals |
| "Take your step" | `piece.canMove(board, from, to)` — one call, many behaviours |
| Each dancer knows its routine | Each `Piece` subclass implements its own `canMove` |
| Adding a new dancer | Adding a new subclass — zero changes to `Game`/`Board` |
| The bad director's script | A `switch (piece.type)` — the anti-pattern to avoid |

The single most important lesson of this whole document: **the board asks; the piece answers.** Hold onto that.

---

## Key concepts inside this topic

We follow the standard 7-step LLD method (see [111 — LLD Approach Framework](./111-lld-approach-framework.md)).

### 1. Requirements & clarifying questions

Always narrow scope out loud before coding. State your assumptions.

**Functional requirements (the core we will build):**
- An 8×8 board.
- Two players (White and Black) who alternate turns; White moves first.
- Each of the six piece types moves according to its own rules.
- **Move validation** — a move is legal only if *all three* hold:
  1. It follows that piece's movement rule.
  2. The destination is not occupied by one of *your own* pieces (and, for sliding pieces, the path between is clear).
  3. It does not leave *your own king* in check.
- **Capture** — moving onto an enemy piece removes it.
- **Detect check, checkmate, and stalemate.**
- **Track game state** (whose turn, is the game over, who won).

**Clarifying questions to ask the interviewer:**
- Do we need the **special moves** — castling, en passant, and pawn promotion? (They're fiddly. I'll design so they *fit*, but implement the core first.)
- Do we need **timers / a chess clock**? (Blitz vs. classical.)
- Do we need **undo / move history / PGN export**?
- Single process (two humans at one keyboard) or networked multiplayer? (Changes nothing in the object model; only the transport.)
- Do we need an **AI opponent**?

For this document: build the core rules + check detection correctly, keep checkmate/stalemate *simplified but correct in spirit*, and design so the extensions slot in. That is exactly the right interview scope.

### 2. Identify core objects (the nouns)

Underline the nouns in the requirements — they become your classes.

| Object | Responsibility |
|---|---|
| `Game` | Top-level controller: turns, status, orchestrates a move end-to-end |
| `Board` | The 8×8 grid; get/set pieces, path checks, find king, "is square attacked?" |
| `Position` | A value object `(row, col)` — an address on the board |
| `Piece` (abstract) | Base for all pieces; declares the abstract `canMove(...)` |
| `King`, `Queen`, `Rook`, `Bishop`, `Knight`, `Pawn` | Concrete pieces, each with its own movement rule |
| `Player` | A participant; holds a `Color` |
| `Move` | A record: from, to, piece moved, piece captured |

**Enums** (`Object.freeze` in JS):
- `Color` — `WHITE` / `BLACK`.
- `GameStatus` — `ACTIVE` / `CHECK` / `CHECKMATE` / `STALEMATE`.

### 3. Behaviours (verbs) + ownership — the central design decision

This is the heart of the problem. List the verbs and decide **who owns each**:

| Behaviour | Owner | Why |
|---|---|---|
| "Can I legally move from A to B?" | **The `Piece`** | Each type's rule differs — this MUST be polymorphic |
| "Is the path between A and B clear?" | `Board` | Only the board sees all squares; sliding pieces reuse it |
| "Where is the king of color C?" | `Board` | The board holds the grid |
| "Is square S attacked by color C?" | `Board` | Needed for check detection; asks every enemy piece `canMove` |
| "Run one full turn (validate, simulate, apply, switch)" | `Game` | The orchestrator |

The decision that makes or breaks this design: **movement rules belong to the piece, not to the game.** Recall the four pillars of OOP from [13 — OOP Four Pillars](./13-oop-four-pillars.md) — this is **polymorphism** (one method name, many behaviours) resting on **inheritance** (all pieces share a base) and **abstraction** (`Piece.canMove` is abstract; callers don't know or care which subclass answers).

**The anti-pattern — do NOT do this:**

```js
// ❌ The giant switch. Every piece rule crammed into the Game/Board.
function canMove(piece, from, to) {
  switch (piece.type) {
    case 'ROOK':   return isStraightLine(from, to);
    case 'BISHOP': return isDiagonal(from, to);
    case 'KNIGHT': return isLShape(from, to);
    case 'QUEEN':  return isStraightLine(from, to) || isDiagonal(from, to);
    case 'KING':   return isOneSquare(from, to);
    case 'PAWN':   return isPawnMove(from, to);  // ...and it gets worse
  }
}
```

Why it's bad: this one function knows *everything*. Adding a fairy-chess piece means editing it. Two engineers touching two piece types collide in the *same* function. It violates the **Open/Closed Principle** — the code is not closed to modification. The switch is duplicated everywhere a piece rule is needed (moving, check detection, AI).

**The clean version — polymorphism:**

```js
// ✅ The board just asks. The piece answers.
if (piece.canMove(board, from, to)) { /* ... */ }
```

`Game` and `Board` contain **zero** `if (piece.type === ...)` checks. Adding a new piece = adding one subclass, touching nothing else. That is the lesson.

### 4. Class diagram

```
                        ┌──────────────────────────┐
                        │           Game           │
                        │--------------------------│
                        │ - board: Board           │
                        │ - players: Player[2]     │
                        │ - currentTurn: Color     │
                        │ - status: GameStatus     │
                        │ - history: Move[]        │
                        │--------------------------│
                        │ + makeMove(from, to)     │
                        │ + isInCheck(color)       │
                        │ + updateStatus()         │
                        └───────┬──────────┬───────┘
                                │ has-a    │ has-a
                     ┌──────────▼───┐  ┌───▼─────────┐
                     │    Board     │  │   Player    │
                     │--------------│  │-------------│
                     │ - grid[8][8] │  │ - color     │
                     │--------------│  │ - name      │
                     │ + getPiece() │  └─────────────┘
                     │ + movePiece()│
                     │ + isPathClear│         ┌──────────────┐
                     │ + findKing() │  uses   │     Move     │
                     │ + isSquare-  │────────▶│--------------│
                     │   Attacked() │         │ from, to     │
                     └──────┬───────┘         │ pieceMoved   │
                            │ holds many      │ pieceCaptured│
                            ▼                 └──────────────┘
                   ┌─────────────────┐
                   │  «abstract»     │        ┌──────────────┐
                   │     Piece       │◀───────│   Position   │
                   │-----------------│  uses  │--------------│
                   │ - color: Color  │        │ row, col     │
                   │ - hasMoved      │        └──────────────┘
                   │-----------------│
                   │ + canMove()  ▲abstract   ← the polymorphic method
                   └───┬───┬───┬───┴───┬───┬───┐
                       │   │   │       │   │   │
              ┌────────┘ ┌─┘  ┌┘       └┐  └─┐ └────────┐
              ▼          ▼    ▼         ▼    ▼          ▼
          ┌──────┐  ┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐
          │ King │  │Queen ││ Rook ││Bishop││Knight││ Pawn │
          └──────┘  └──────┘└──────┘└──────┘└──────┘└──────┘
           each overrides canMove() with ITS OWN rule
```

The vertical fan-out under `Piece` is the diagram that matters. Six subclasses, one shared abstract method, each overriding it. `Game` and `Board` point at `Piece` — never at a concrete subclass.

### 5. Full JavaScript implementation

Runnable end-to-end. Piece rules are correct; check/checkmate is deliberately simplified where noted.

```js
'use strict';

// ---------- Enums (frozen objects) ----------
const Color = Object.freeze({ WHITE: 'WHITE', BLACK: 'BLACK' });

const GameStatus = Object.freeze({
  ACTIVE: 'ACTIVE',
  CHECK: 'CHECK',
  CHECKMATE: 'CHECKMATE',
  STALEMATE: 'STALEMATE',
});

const opposite = (color) => (color === Color.WHITE ? Color.BLACK : Color.WHITE);

// ---------- Position: an immutable (row, col) value object ----------
class Position {
  constructor(row, col) {
    this.row = row;
    this.col = col;
    Object.freeze(this);
  }
  equals(other) { return this.row === other.row && this.col === other.col; }
  // 'e2' style algebraic notation, purely for pretty printing
  toString() { return `${'abcdefgh'[this.col]}${8 - this.row}`; }
}

// ---------- Piece: the abstract base. THE STAR OF THE SHOW ----------
class Piece {
  constructor(color) {
    if (new.target === Piece) {
      throw new Error('Piece is abstract — instantiate a concrete subclass');
    }
    this.color = color;
    this.hasMoved = false; // needed later for castling / pawn double-step
  }

  // Abstract: every subclass MUST override this with its own movement rule.
  // Returns true only if the geometry of from->to is legal for THIS piece.
  // (Occupancy of the destination by own piece is checked by the caller.)
  canMove(board, from, to) {
    throw new Error(`${this.constructor.name} must implement canMove()`);
  }

  // One-letter symbol for board printing. White upper, black lower.
  symbol() {
    const s = this.letter();
    return this.color === Color.WHITE ? s.toUpperCase() : s.toLowerCase();
  }
  letter() { return '?'; }
}

// ---------- Rook: straight lines ----------
class Rook extends Piece {
  letter() { return 'R'; }
  canMove(board, from, to) {
    const straight = from.row === to.row || from.col === to.col;
    // must move in a rank or file, and nothing may block the way
    return straight && !from.equals(to) && board.isPathClear(from, to);
  }
}

// ---------- Bishop: diagonals ----------
class Bishop extends Piece {
  letter() { return 'B'; }
  canMove(board, from, to) {
    const diagonal = Math.abs(from.row - to.row) === Math.abs(from.col - to.col);
    return diagonal && !from.equals(to) && board.isPathClear(from, to);
  }
}

// ---------- Queen: rook OR bishop. Note we REUSE the same geometry. ----------
class Queen extends Piece {
  letter() { return 'Q'; }
  canMove(board, from, to) {
    const straight = from.row === to.row || from.col === to.col;
    const diagonal = Math.abs(from.row - to.row) === Math.abs(from.col - to.col);
    return (straight || diagonal) && !from.equals(to) && board.isPathClear(from, to);
  }
}

// ---------- Knight: the L-shape, and it JUMPS (no path check) ----------
class Knight extends Piece {
  letter() { return 'N'; } // 'N' because K is taken by King
  canMove(board, from, to) {
    const dr = Math.abs(from.row - to.row);
    const dc = Math.abs(from.col - to.col);
    // (2,1) or (1,2). No isPathClear — knights leap over anything.
    return (dr === 2 && dc === 1) || (dr === 1 && dc === 2);
  }
}

// ---------- King: one square in any direction ----------
class King extends Piece {
  letter() { return 'K'; }
  canMove(board, from, to) {
    const dr = Math.abs(from.row - to.row);
    const dc = Math.abs(from.col - to.col);
    return dr <= 1 && dc <= 1 && !from.equals(to);
    // (Castling is an extension — see section 7.)
  }
}

// ---------- Pawn: the most special-cased piece ----------
class Pawn extends Piece {
  letter() { return 'P'; }
  canMove(board, from, to) {
    // White moves UP the board (row decreasing), Black moves DOWN.
    const dir = this.color === Color.WHITE ? -1 : 1;
    const startRow = this.color === Color.WHITE ? 6 : 1;
    const dr = to.row - from.row;
    const dc = to.col - from.col;
    const target = board.getPiece(to);

    // 1) Straight forward one square onto an empty square
    if (dc === 0 && dr === dir && !target) return true;

    // 2) Straight forward two squares from the starting rank, path clear
    if (dc === 0 && dr === 2 * dir && from.row === startRow && !target) {
      const middle = new Position(from.row + dir, from.col);
      return !board.getPiece(middle);
    }

    // 3) Diagonal capture of an ENEMY piece
    if (Math.abs(dc) === 1 && dr === dir && target && target.color !== this.color) {
      return true;
    }
    // (En passant and promotion are extensions — see section 7.)
    return false;
  }
}

// ---------- Move: an immutable record of what happened ----------
class Move {
  constructor(from, to, pieceMoved, pieceCaptured = null) {
    this.from = from;
    this.to = to;
    this.pieceMoved = pieceMoved;
    this.pieceCaptured = pieceCaptured;
  }
}

// ---------- Board: the grid + all spatial queries ----------
class Board {
  constructor() {
    // grid[row][col]; row 0 is rank 8 (Black's back rank), row 7 is rank 1.
    this.grid = Array.from({ length: 8 }, () => Array(8).fill(null));
  }

  static inBounds(pos) {
    return pos.row >= 0 && pos.row < 8 && pos.col >= 0 && pos.col < 8;
  }

  getPiece(pos) { return this.grid[pos.row][pos.col]; }
  setPiece(pos, piece) { this.grid[pos.row][pos.col] = piece; }

  // Move a piece; return whatever was captured (or null). Low-level, no rules.
  movePiece(from, to) {
    const piece = this.getPiece(from);
    const captured = this.getPiece(to);
    this.setPiece(to, piece);
    this.setPiece(from, null);
    if (piece) piece.hasMoved = true;
    return captured;
  }

  // Are all squares strictly BETWEEN from and to empty? For sliding pieces.
  isPathClear(from, to) {
    const stepR = Math.sign(to.row - from.row);
    const stepC = Math.sign(to.col - from.col);
    let r = from.row + stepR;
    let c = from.col + stepC;
    while (r !== to.row || c !== to.col) {
      if (this.grid[r][c] !== null) return false; // blocked
      r += stepR;
      c += stepC;
    }
    return true;
  }

  findKing(color) {
    for (let r = 0; r < 8; r++) {
      for (let c = 0; c < 8; c++) {
        const p = this.grid[r][c];
        if (p instanceof King && p.color === color) return new Position(r, c);
      }
    }
    return null;
  }

  // Is `pos` attacked by any piece of `byColor`? Ask every enemy piece.
  // Note: we use canMove for attack detection. This is correct for all pieces
  // EXCEPT pawns (which attack diagonally but move straight). We special-case it.
  isSquareAttacked(pos, byColor) {
    for (let r = 0; r < 8; r++) {
      for (let c = 0; c < 8; c++) {
        const p = this.grid[r][c];
        if (!p || p.color !== byColor) continue;
        const from = new Position(r, c);
        if (p instanceof Pawn) {
          // pawn attacks the two forward diagonals only
          const dir = p.color === Color.WHITE ? -1 : 1;
          if (from.row + dir === pos.row && Math.abs(from.col - pos.col) === 1) {
            return true;
          }
        } else if (p.canMove(this, from, pos)) {
          return true;
        }
      }
    }
    return false;
  }

  print() {
    const lines = [];
    for (let r = 0; r < 8; r++) {
      let line = `${8 - r} `;
      for (let c = 0; c < 8; c++) {
        const p = this.grid[r][c];
        line += (p ? p.symbol() : '.') + ' ';
      }
      lines.push(line);
    }
    lines.push('  a b c d e f g h');
    return lines.join('\n');
  }
}

// ---------- Player ----------
class Player {
  constructor(name, color) {
    this.name = name;
    this.color = color;
  }
}

// ---------- Game: the orchestrator that runs one full turn ----------
class Game {
  constructor() {
    this.board = new Board();
    this.players = [new Player('White', Color.WHITE), new Player('Black', Color.BLACK)];
    this.currentTurn = Color.WHITE;
    this.status = GameStatus.ACTIVE;
    this.history = [];
    this._setup();
  }

  // Factory-style initial layout (see section 6 — Factory pattern).
  _setup() {
    const back = [Rook, Knight, Bishop, Queen, King, Bishop, Knight, Rook];
    for (let c = 0; c < 8; c++) {
      this.board.setPiece(new Position(0, c), new back[c](Color.BLACK));
      this.board.setPiece(new Position(1, c), new Pawn(Color.BLACK));
      this.board.setPiece(new Position(6, c), new Pawn(Color.WHITE));
      this.board.setPiece(new Position(7, c), new back[c](Color.WHITE));
    }
  }

  isInCheck(color) {
    const kingPos = this.board.findKing(color);
    if (!kingPos) return false;
    return this.board.isSquareAttacked(kingPos, opposite(color));
  }

  // The full validated turn. Returns { ok, reason }.
  makeMove(from, to) {
    if (this.status === GameStatus.CHECKMATE || this.status === GameStatus.STALEMATE) {
      return { ok: false, reason: 'game is over' };
    }
    if (!Board.inBounds(from) || !Board.inBounds(to)) {
      return { ok: false, reason: 'off the board' };
    }

    const piece = this.board.getPiece(from);

    // (a) Is there a piece, and is it the current player's?
    if (!piece) return { ok: false, reason: 'no piece at source' };
    if (piece.color !== this.currentTurn) {
      return { ok: false, reason: `it is ${this.currentTurn}'s turn` };
    }

    // (b) Destination must not hold one of YOUR OWN pieces.
    const dest = this.board.getPiece(to);
    if (dest && dest.color === piece.color) {
      return { ok: false, reason: 'destination occupied by your own piece' };
    }

    // (c) The move must obey THIS piece's rule. Pure polymorphism — the Game
    //     has no idea what kind of piece it is. It just asks.
    if (!piece.canMove(this.board, from, to)) {
      return { ok: false, reason: 'illegal move for that piece' };
    }

    // (d) Simulate the move and reject it if it leaves OUR king in check.
    const captured = this.board.movePiece(from, to);
    const leavesKingInCheck = this.isInCheck(piece.color);
    if (leavesKingInCheck) {
      this.board.movePiece(to, from);        // undo the piece move
      if (captured) this.board.setPiece(to, captured); // restore the victim
      return { ok: false, reason: 'move leaves your king in check' };
    }

    // Move is fully legal — commit it.
    this.history.push(new Move(from, to, piece, captured));
    this.currentTurn = opposite(this.currentTurn);
    this._updateStatus();
    return { ok: true, reason: captured ? `captured ${captured.symbol()}` : 'ok' };
  }

  // Simplified-but-correct-in-spirit status update.
  // We compute: is the side-to-move in check, and do they have ANY legal move?
  //   in check  + no legal moves  -> CHECKMATE
  //   not check + no legal moves  -> STALEMATE
  //   in check  + has legal moves -> CHECK
  //   otherwise                    -> ACTIVE
  _updateStatus() {
    const color = this.currentTurn;
    const inCheck = this.isInCheck(color);
    const hasMove = this._hasAnyLegalMove(color);
    if (inCheck && !hasMove) this.status = GameStatus.CHECKMATE;
    else if (!inCheck && !hasMove) this.status = GameStatus.STALEMATE;
    else if (inCheck) this.status = GameStatus.CHECK;
    else this.status = GameStatus.ACTIVE;
  }

  // Brute force: does `color` have at least one move that is legal AND does not
  // leave their king in check? O(pieces * 64) — fine for a teaching model.
  _hasAnyLegalMove(color) {
    for (let r = 0; r < 8; r++) {
      for (let c = 0; c < 8; c++) {
        const p = this.board.grid[r][c];
        if (!p || p.color !== color) continue;
        const from = new Position(r, c);
        for (let tr = 0; tr < 8; tr++) {
          for (let tc = 0; tc < 8; tc++) {
            const to = new Position(tr, tc);
            const target = this.board.getPiece(to);
            if (target && target.color === color) continue;
            if (!p.canMove(this.board, from, to)) continue;
            // simulate
            const cap = this.board.movePiece(from, to);
            const bad = this.isInCheck(color);
            this.board.movePiece(to, from);
            if (cap) this.board.setPiece(to, cap);
            if (!bad) return true; // found one legal move
          }
        }
      }
    }
    return false;
  }
}

// ---------- main(): prove it runs ----------
function main() {
  const game = new Game();
  const P = (algebraic) => {
    // 'e2' -> Position. col from a-h, row from 8..1.
    const col = 'abcdefgh'.indexOf(algebraic[0]);
    const row = 8 - Number(algebraic[1]);
    return new Position(row, col);
  };

  const play = (fromStr, toStr) => {
    const res = game.makeMove(P(fromStr), P(toStr));
    const tag = res.ok ? 'OK ' : 'REJECTED';
    console.log(`\n${tag}: ${fromStr}->${toStr}  (${res.reason})  [status=${game.status}]`);
    console.log(game.board.print());
  };

  console.log('Initial position:');
  console.log(game.board.print());

  play('e2', 'e4'); // white pawn double-step        -> legal
  play('e7', 'e5'); // black pawn double-step        -> legal
  play('a1', 'c3'); // white ROOK moving DIAGONALLY  -> REJECTED (illegal geometry), still white's turn
  play('g1', 'f3'); // white knight jumps out        -> legal
  play('b8', 'c6'); // black knight                  -> legal
  play('f1', 'c4'); // white bishop out (e2 vacated) -> legal
  play('d7', 'd6'); // black pawn one step           -> legal
  play('c4', 'f7'); // white bishop CAPTURES the f7 pawn (diagonal) -> legal

  console.log('\nMove history length:', game.history.length);
}

main();

module.exports = { Game, Board, Position, Piece, King, Queen, Rook, Bishop, Knight, Pawn, Color, GameStatus };
```

Run it with `node chess.js`. You'll see the board redraw after each move, the diagonal-rook attempt rejected, and the status tracked.

**Why the `main()` proves the design:** at no point does `main` or `Game` ask "what kind of piece is this?" The rook's diagonal move is rejected because `Rook.canMove` returned `false` — the rook rejected itself. That's the whole point.

### 6. Design patterns used and WHY

| Pattern | Where | Why it's the right call |
|---|---|---|
| **Polymorphism + Inheritance** | `Piece.canMove` overridden by six subclasses | THE star. One call site (`piece.canMove`), six behaviours. `Game`/`Board` never branch on type. Recall [13 — OOP Four Pillars](./13-oop-four-pillars.md). |
| **Strategy** | Each piece's movement rule *is* a strategy | The move rule is an interchangeable algorithm. Here the strategy is fused into the subclass, but you could pull each rule into a separate `MovementStrategy` object — see [42 — Strategy](./42-pattern-strategy.md). |
| **Factory** | `Game._setup()` builds the initial layout | Centralises "which class goes on which square" so construction logic lives in one place, not scattered. |
| **Visitor** *(alternative)* | Adding new *operations* across all piece types | See below. |

**The Strategy connection (topic 42).** Strategy says "encapsulate an algorithm and make it swappable." Our pieces *are* that: `canMove` is the swappable algorithm, chosen by the object's class at runtime. The difference from textbook Strategy is that we bind the algorithm via subclassing rather than composition. If you needed the *same* piece to change how it moves at runtime (a "berserk" mode, say), you'd switch to real composition: `piece.movementStrategy = new BerserkStrategy()`.

**The Visitor honest trade-off (topic 50).** Our inheritance design makes it cheap to add a **new piece** (one subclass, touch nothing) but expensive to add a **new operation over all pieces** (e.g. `toFEN()`, `pointValue()`, `renderSVG()`) — you'd edit all six subclasses. [50 — Visitor](./50-pattern-visitor.md) inverts that trade-off: put each operation in a `Visitor` and pieces `accept(visitor)`. Then a new operation is one new visitor class, but a new piece forces you to edit every visitor. This is the classic "expression problem." For chess, pieces are fixed (there are exactly six) and operations grow (move, score, render, serialise) — so honestly, **Visitor is arguably the better long-term choice** here. We use inheritance because it's the clearest teaching vehicle and the interviewer's expected answer; mentioning the Visitor trade-off out loud is what separates a strong candidate.

### 7. Extensions — "now add X"

Watch how the polymorphic core absorbs each request.

- **Castling / en passant / promotion (special moves).** These are moves that touch *more than one square* or *transform a piece*, so they don't fit the pure "from→to geometry" of `canMove`. Cleanest home: a `SpecialMove` check in `Game.makeMove` *before* the normal path, plus flags already present (`hasMoved` on King/Rook enables castling; a `lastMove` on `Game` enables en passant; promotion swaps the `Pawn` for a `Queen` when it reaches the back rank). The six `canMove` methods stay untouched — special cases live at the orchestration layer, not inside the pieces.

- **Move history + undo.** We already push `Move` records into `history`. Undo = the **Memento/Command** pattern: each `Move` already stores `pieceCaptured`, so `undo()` pops the last move, moves the piece back, and restores the captured piece (exactly the simulate-and-undo logic we used for check testing). Make `Move` a `Command` with `execute()`/`undo()` and you get unlimited undo for free.

- **A chess clock.** Add a `Clock` per player (`remainingMs`), start/stop it in `makeMove` on turn switch, and add a `TIMEOUT` outcome. Fully orthogonal — the piece hierarchy doesn't change at all.

- **An AI player.** An AI is just "pick a `from,to`." Generate all legal moves (we already have `_hasAnyLegalMove`; generalise it to *return* the list), score each resulting board with an evaluation function (material + position), and pick the best — that's minimax. The move *generator* leans entirely on the same polymorphic `canMove`. This is itself a Strategy: `RandomAI`, `MinimaxAI`, `HumanPlayer` all implement `chooseMove(game)`.

The pattern in every extension: the six pieces never change. New behaviour attaches at the `Game` layer or as a new companion class. That is the payoff of putting movement rules inside the pieces.

---

## Visual / Diagram description

Sequence of one validated move, `Game.makeMove(from, to)`:

```
 Caller        Game                 Board                Piece(e.g Rook)
   │            │                     │                        │
   │ makeMove ─▶│                     │                        │
   │            │ getPiece(from) ────▶│                        │
   │            │◀─── piece ──────────│                        │
   │            │  right turn? own?   │                        │
   │            │  dest not own? ────▶│ getPiece(to)           │
   │            │                     │                        │
   │            │ canMove(board,f,t) ─┼───────────────────────▶│
   │            │                     │  isPathClear(f,t) ◀────│
   │            │                     │──── true/false ───────▶│
   │            │◀──── true/false ────┼────────────────────────│
   │            │ movePiece(f,t) ────▶│  (simulate)            │
   │            │ isInCheck(mine)? ──▶│ isSquareAttacked(king) │
   │            │   if bad: undo move + restore captured       │
   │            │ else: commit, switch turn, updateStatus()    │
   │◀── result ─│                     │                        │
```

Read it top to bottom: `Game` gathers cheap facts (turn, ownership), then delegates the *geometry* question to the piece (which may call back into `Board.isPathClear`), then does the expensive **simulate → test check → maybe undo** dance. The arrow that carries the whole design is `canMove(board, f, t)` crossing from `Game` to `Piece` — `Game` on the left never learns the piece's type.

---

## Real world examples

### Lichess (open-source chess server)

Lichess's move engine (the `scalachess` library) models pieces and their move generation as distinct, type-directed rules rather than one mega-conditional — the same "each role generates its own moves" idea, just in Scala with immutable boards. It's a real, production, millions-of-games-a-day proof that the piece-owns-its-rule model scales. (Representative — based on the public open-source structure.)

### Stockfish (the world's strongest engine)

Stockfish generates moves with per-piece-type routines and precomputed attack tables (bitboards), and pieces carry a **point value** used by the evaluation function — a concrete example of the "add an operation across all pieces" (Visitor-flavoured) need discussed in section 6. Its design shows both axes of the expression problem in one codebase: fixed set of piece types, many operations (generate, evaluate, hash).

### Any "family of behaviours" service you'll build at work

Payment processors (`CardPayment`, `WalletPayment`, `UPIPayment` each implementing `charge()`), notification channels (`Email`, `SMS`, `Push` each implementing `send()`), file exporters (`CsvExporter`, `PdfExporter` each implementing `export()`). All are chess pieces wearing a suit: one interface, many self-contained implementations, and an orchestrator that never switches on type.

---

## Trade-offs

**Inheritance/polymorphism for pieces**

| Pros | Cons |
|---|---|
| Adding a new piece = one subclass, zero edits elsewhere | Adding a new *operation* means editing all six subclasses |
| `Game`/`Board` stay tiny and type-agnostic | Deep hierarchies can get rigid if rules share partial logic |
| Directly models the domain (a rook IS-A piece) | Runtime rule-swapping needs composition, not inheritance |

**Where check/checkmate detection lives**

| Approach | Pros | Cons |
|---|---|---|
| Brute-force "simulate every move, test check" (ours) | Simple, obviously correct | O(pieces × 64 × board-scan) — slow, but fine for humans |
| Precomputed attack tables / bitboards (Stockfish) | Blazing fast | Complex, hard to get right, overkill for an interview |

**The sweet spot:** an abstract `Piece` with polymorphic `canMove`, a thin type-agnostic `Game`/`Board`, and brute-force check detection. It's correct, readable, and interview-perfect. Reach for bitboards or the Visitor pattern only when the requirements (an engine, many cross-cutting operations) actually demand them — and say so out loud.

---

## Common interview questions on this topic

### Q1: "How would you model the different pieces?"
**Hint:** Abstract base class `Piece` with an abstract `canMove(board, from, to)`; six subclasses each override it with their own rule. Explicitly reject the `switch (piece.type)` anti-pattern and explain *why* (Open/Closed, duplication, merge conflicts). This is the answer they're actually testing for.

### Q2: "How do you make sure a move doesn't leave your own king in check?"
**Hint:** You can't know without trying it. **Simulate**: make the move on the board, ask `isInCheck(myColor)` (find my king, ask if any enemy piece attacks that square), and if the answer is yes, **undo** the move and reject it. Show the make-move → test → undo/restore-captured sequence.

### Q3: "How do you detect checkmate vs. stalemate?"
**Hint:** Both mean "the side to move has no legal move." The difference is *check*: no legal move **while in check** = checkmate; no legal move **while not in check** = stalemate. Generate all pseudo-legal moves, filter out those that leave your king in check; if none survive, branch on `isInCheck`.

### Q4: "Where do castling and en passant live? They don't fit `canMove(from, to)`."
**Hint:** Correct — they're multi-square / stateful moves, so they don't belong inside a piece's geometry rule. Handle them at the `Game.makeMove` orchestration layer using state you already track (`hasMoved` flags, the last move). The piece hierarchy stays clean.

### Q5: "Adding a new operation like `renderSVG()` on every piece is painful. Alternatives?"
**Hint:** That's the expression problem. Inheritance makes new *pieces* cheap but new *operations* expensive. The Visitor pattern ([50](./50-pattern-visitor.md)) flips it: `piece.accept(renderVisitor)`. Name the honest trade-off — pieces are fixed at six, so Visitor is defensible here.

---

## Practice exercise

### Build it and add promotion

1. Copy the section-5 code into `chess.js` and run it with `node chess.js`. Confirm the diagonal-rook move is rejected and the board redraws.
2. Add a **`Pawn` promotion** rule: in `Game.makeMove`, after committing a pawn move, if the pawn lands on the opponent's back rank (row 0 for White, row 7 for Black), replace it with a `new Queen(pawn.color)` on that square. Note where you put this — it should be in the `Game` orchestration layer, **not** inside `Pawn.canMove`.
3. Add a `Fairy` piece of your own invention (e.g. moves like a knight *or* a king) by writing **one** new subclass. Prove you did **not** touch `Game` or `Board` at all.
4. Write a scenario that ends in checkmate (the "Fool's Mate": `f2-f3, e7-e5, g2-g4, d8-h4#`) and confirm `game.status === 'CHECKMATE'`.

Produce: the modified `chess.js`, and a one-paragraph note on *which* files/classes you had to touch for step 3 (answer: only your new subclass — that's the lesson).

---

## Quick reference cheat sheet

- **The one rule:** the board *asks* (`piece.canMove(...)`), the piece *answers*. `Game`/`Board` never switch on piece type.
- **Abstract `Piece`** with an abstract `canMove(board, from, to)`; six subclasses override it — that's **polymorphism** ([13](./13-oop-four-pillars.md)).
- **Anti-pattern to name and reject:** `switch (piece.type)` — violates Open/Closed, duplicates logic, causes merge conflicts.
- **Rook** = straight + path clear. **Bishop** = diagonal + path clear. **Queen** = either. **Knight** = L-shape, *jumps* (no path check). **King** = one square. **Pawn** = forward 1/2, capture diagonally, direction by color.
- **`isPathClear`** lives on `Board` (sliding pieces reuse it); **Knight skips it**.
- **Check** = your king's square is attacked by the enemy → `isSquareAttacked(findKing(me), enemy)`.
- **Legal move test:** simulate → `isInCheck(me)`? → undo if bad. Never mutate permanently before validating.
- **Checkmate** = in check + no legal move. **Stalemate** = not in check + no legal move.
- **Enums** via `Object.freeze`: `Color`, `GameStatus`.
- **Patterns:** Polymorphism (star), Strategy ([42](./42-pattern-strategy.md)) = the move rule, Factory = initial layout, Visitor ([50](./50-pattern-visitor.md)) = alternative for cross-piece operations.
- **Special moves** (castling, en passant, promotion) live in `Game.makeMove`, *not* inside pieces.
- **Undo** = Command/Memento; `Move` already stores `pieceCaptured`.
- **AI** = generate legal moves (reuse `canMove`) + evaluate + pick best; itself a Strategy.
- **Interview tell:** mentioning the expression-problem trade-off between inheritance and Visitor marks a strong candidate.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The 7-step method (requirements → nouns → verbs → diagram → code → patterns → extensions) we followed here |
| **Next** | [120 — LLD Snake and Ladder](./120-lld-snake-and-ladder.md) | Another board-game LLD; contrast its simpler, data-driven design with chess's polymorphic pieces |
| **Related** | [13 — OOP Four Pillars](./13-oop-four-pillars.md) | Polymorphism + inheritance + abstraction — the exact pillars the piece hierarchy is built on |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) | Each piece's movement rule is effectively a swappable strategy |
| **Related** | [50 — Visitor Pattern](./50-pattern-visitor.md) | The alternative design for adding operations across all piece types (the expression problem) |
