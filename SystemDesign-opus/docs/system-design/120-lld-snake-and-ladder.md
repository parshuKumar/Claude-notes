# 120 — Design a Snake and Ladder Game
## Category: LLD Case Study

---

## What is this?

Snake and Ladder is a classic board game: players roll a die and race along a numbered board from cell 1 to cell 100. Ladders shoot you upward (a shortcut), snakes drag you back down (a setback), and the first player to land exactly on the last cell wins.

As an LLD problem it is a gentle one — the game rules are tiny and everyone already knows them. That is exactly why interviewers use it: with no rule-complexity to hide behind, your **object modelling** is on full display. Think of it as a scaffolding exercise before the harder games (like the chess design in [119 — Chess Game](./119-lld-chess-game.md)).

---

## Why does it matter?

**Interview angle.** Snake and Ladder tests one thing above all: can you find the *clean abstraction*? The killer insight is that a snake and a ladder are the same object — a "jump" from one cell to another. A snake jumps you backward, a ladder jumps you forward. A weak candidate writes a `Snake` class and a `Ladder` class with duplicated logic; a strong candidate writes one `Jump` and moves on. Interviewers watch for exactly this deduplication.

**Real-work angle.** The pattern recurs constantly: two features look different on the surface but share a structure. Discounts and surcharges (both are "price adjustments"), promotions and penalties, upvotes and downvotes. If you can spot "these are one thing with a sign flipped," you write half as much code and it has half as many bugs. Snake and Ladder is the toy version of that skill.

It is also a great place to practise *restraint*. The problem is simple, so heavy patterns are over-engineering. Recall the "patternitis" warning from [30 — Factory Method](./30-pattern-factory-method.md): a pattern you don't need is a liability, not a badge.

---

## The core idea — explained simply

### Analogy: a hallway of numbered doors with trapdoors and elevators

Picture a long hallway with 100 numbered doors laid end to end. You start before door 1 and want to reach door 100. Each turn you roll a die and walk that many doors forward.

Most doors are boring — you just stop there and wait for your next turn. But a few doors are special:

- Some doors hide an **elevator** behind them. Open it and you are instantly whisked *up* to a higher-numbered door. (That's a ladder.)
- Some doors hide a **trapdoor**. Open it and you slide *down* to a lower-numbered door. (That's a snake.)

The crucial observation: an elevator and a trapdoor are *the same machine* — a teleporter with an entrance door and an exit door. The only difference is whether the exit number is higher or lower than the entrance.

| Hallway thing | Game concept | Code concept |
|---|---|---|
| A numbered door | A cell on the board | index in `1..100` |
| Walking forward | Moving your token | `position + roll` |
| The die you roll | The dice | `Dice.roll()` |
| Elevator / trapdoor | Ladder / snake | a `Jump(start, end)` |
| Elevator (exit > entrance) | Ladder | `end > start` |
| Trapdoor (exit < entrance) | Snake | `end < start` |
| Reaching door 100 | Winning | `position === size` |
| A player's token | A player | `Player.position` |
| The referee calling turns | The game loop | `Game.play()` |

Once you see snakes and ladders as one `Jump` type, the whole design collapses into something you can hold in your head: a **board** is a grid of cells plus a lookup table of jumps; a **turn** is "roll, walk, maybe teleport, check if you won."

---

## Key concepts inside this topic

We'll follow the standard 7-step LLD method (the same one from [111 — LLD Approach Framework](./111-lld-approach-framework.md)): requirements, nouns, verbs, class diagram, code, patterns, extensions.

### 1. Requirements & clarifying questions

**Functional requirements.**
- A board of `N` cells, numbered `1..N` (usually `N = 100`).
- The board has some **snakes** (send a player down) and some **ladders** (send a player up). Each has a start cell and an end cell.
- Two or more **players** take turns, in a fixed rotation, rolling one or more dice and moving forward by the total.
- If a player lands on a **ladder bottom** or a **snake head**, they immediately move to the other end of that jump.
- The first player to reach the final cell (`N`) **wins** and the game ends.

**Non-functional / design goals.**
- Clean, deduplicated modelling (snakes and ladders share a type).
- Deterministic, reproducible demo (so we can test it) via injectable dice rolls.
- Easy to extend (new board layouts, new special-cell types, more dice).

**Clarifying questions to ask the interviewer** (asking these signals seniority):
1. **Board size?** Fixed at 100, or configurable? (We'll make it configurable, default 100.)
2. **How many dice?** One standard six-sided die, or multiple summed? (Configurable, default 1.)
3. **Exact finish required?** If a roll would overshoot the last cell, does the player *bounce back*, or simply *not move* and wait? (Both are common. We'll implement "don't move on overshoot," and note where bounce-back would slot in.)
4. **Bonus roll on a 6?** Some variants grant another turn. (Out of scope for the base design; shown as an extension.)
5. **Can a jump chain?** If a ladder drops you on a snake head, do you take the snake too? (Standard rule: no — one jump per landing. We'll follow that.)
6. **Can two players occupy the same cell?** Yes, no collisions. (Some variants send a player back on collision — an extension.)

### 2. Identify core objects (nouns)

Reading the requirements and circling the nouns:

- **Game** — orchestrates turns and decides the winner.
- **Board** — knows its size and holds the jump lookup.
- **Cell** — a single position `1..N`. (Often just an integer; we keep it lightweight but conceptually it's a cell.)
- **Jump** — a teleport from `start` to `end`. **Snakes and ladders are both Jumps.** This is the modelling win.
- **Player** — has a name and a current position.
- **Dice** — rolls one or more dice and returns the total.

Notice what we *didn't* create: no separate `Snake` and `Ladder` classes with copy-pasted move logic. A snake is just a `Jump` where `end < start`; a ladder is a `Jump` where `end > start`. One type, two flavours.

### 3. Behaviours (verbs) + who owns them

| Behaviour | Owner | Why it lives there |
|---|---|---|
| `roll()` → total | `Dice` | Only the dice knows how many dice and their faces. |
| `getJumpAt(cell)` | `Board` | The board owns the map of which cells have jumps. |
| `getNextPosition(current, roll)` | `Board` | Movement + overshoot + jump resolution is a board rule. |
| `moveTo(position)` | `Player` | A player owns its own position state. |
| `hasWon(boardSize)` | `Player` | "Am I at the end?" is about the player's state. |
| `play()` / `nextTurn()` | `Game` | The game owns turn rotation and the win check. |
| announce events | `Observer(s)` | Reporting is a *separate concern* from game rules. |

The ownership split matters: the `Board` is a pure rules engine (no I/O, no turn logic), the `Game` is the coordinator, and *printing* is pushed out to observers so the rules stay clean. That separation is the Single Responsibility Principle from [13 — OOP Four Pillars](./13-oop-four-pillars.md) doing its job.

### 4. Class diagram

```
                    ┌──────────────────────────┐
                    │           Game           │
                    │--------------------------│
                    │ - board : Board          │
                    │ - dice  : Dice           │
                    │ - players : Queue<Player>│
                    │ - observers : Observer[] │
                    │--------------------------│
                    │ + play()                 │
                    │ + addObserver(o)          │
                    │ - takeTurn(player)        │
                    └───┬──────────┬────────┬───┘
             ◇ owns 1   │  ◇ owns 1│        │ ◇ owns 2..*
          ┌─────────────┘          │        └──────────────┐
          ▼                        ▼                        ▼
   ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
   │    Board     │        │     Dice     │        │    Player    │
   │--------------│        │--------------│        │--------------│
   │ - size       │        │ - count      │        │ - name       │
   │ - jumps:Map  │        │ - roller()   │        │ - position   │
   │--------------│        │--------------│        │--------------│
   │ +getJumpAt() │        │ + roll()     │        │ + moveTo()   │
   │ +getNextPos()│        └──────────────┘        │ + hasWon()   │
   └──────┬───────┘                                └──────────────┘
          │ ◇ owns 0..* (in a Map keyed by start cell)
          ▼
   ┌──────────────┐
   │     Jump     │   snake: end < start
   │--------------│   ladder: end > start
   │ - start      │
   │ - end        │
   │--------------│
   │ + isSnake()  │
   │ + isLadder() │
   └──────────────┘

   BoardFactory ....builds....▶ Board (with its Jumps)
   Observer (interface) ◁---- ConsoleObserver   (Game notifies observers)
```

Read it as: a `Game` **owns** (the `◇` aggregation diamond) one `Board`, one `Dice`, a queue of `Player`s, and a list of `Observer`s. The `Board` **owns** a map of `Jump`s. A `BoardFactory` constructs boards, and the `Game` notifies `Observer`s of events.

### 5. Full JavaScript implementation

Everything below is one runnable file. Copy it into `snake-ladder.js` and run `node snake-ladder.js`.

```js
'use strict';

// -------------------------------------------------------------------------
// Dice — owns the "roll" behaviour.
// We inject a `roller` function so the demo is REPRODUCIBLE. In production
// you'd default to Math.random; for tests/demos you pass a fixed sequence.
// -------------------------------------------------------------------------
class Dice {
  /**
   * @param {number} count   how many dice to roll (default 1)
   * @param {number} faces   faces per die (default 6)
   * @param {function():number} roller  returns a single die face 1..faces.
   *        Defaults to a real random die. Inject your own for determinism.
   */
  constructor(count = 1, faces = 6, roller = null) {
    this.count = count;
    this.faces = faces;
    this._roller = roller || (() => 1 + Math.floor(Math.random() * faces));
  }

  /** Roll all dice and return the SUM. */
  roll() {
    let total = 0;
    for (let i = 0; i < this.count; i++) total += this._roller();
    return total;
  }
}

// -------------------------------------------------------------------------
// Jump — the KEY abstraction. A snake AND a ladder are both a Jump.
// A ladder has end > start (go up); a snake has end < start (go down).
// One class, no duplicated movement logic.
// -------------------------------------------------------------------------
class Jump {
  constructor(start, end) {
    if (start === end) throw new Error('A jump cannot start and end on the same cell');
    this.start = start;
    this.end = end;
  }
  isLadder() { return this.end > this.start; }
  isSnake() { return this.end < this.start; }
  label() { return this.isLadder() ? 'ladder' : 'snake'; }
}

// -------------------------------------------------------------------------
// Player — owns its own position state.
// -------------------------------------------------------------------------
class Player {
  constructor(name) {
    this.name = name;
    this.position = 0; // 0 means "off the board", before cell 1
  }
  moveTo(position) { this.position = position; }
  hasWon(boardSize) { return this.position === boardSize; }
}

// -------------------------------------------------------------------------
// Board — a pure rules engine. Knows its size and its jumps.
// Owns the jump lookup and the movement rule (walk + overshoot + jump).
// No printing, no turn logic — that keeps it single-purpose and testable.
// -------------------------------------------------------------------------
class Board {
  constructor(size, jumps = []) {
    this.size = size;
    this._jumps = new Map(); // start cell -> Jump
    for (const jump of jumps) this.addJump(jump);
  }

  addJump(jump) {
    if (jump.start < 1 || jump.start > this.size) {
      throw new Error(`Jump start ${jump.start} is off the board`);
    }
    if (this._jumps.has(jump.start)) {
      throw new Error(`Two jumps cannot share start cell ${jump.start}`);
    }
    this._jumps.set(jump.start, jump);
    return this;
  }

  /** Is there a snake head or ladder bottom at this cell? Returns Jump or null. */
  getJumpAt(cell) {
    return this._jumps.get(cell) || null;
  }

  /**
   * Given a current position and a die roll, compute the resulting position
   * AND report what happened. Returns { position, overshoot, jump }.
   *   - overshoot: true if the roll would pass the last cell (player stays put)
   *   - jump: the Jump taken on landing, or null
   */
  getNextPosition(current, roll) {
    const landing = current + roll;

    // Overshoot rule: a roll past the final cell does not move the player.
    // (To implement "bounce back" instead, reflect: landing = size - (landing - size).)
    if (landing > this.size) {
      return { position: current, overshoot: true, jump: null };
    }

    // Landed exactly or short. Now check for a snake/ladder at the landing cell.
    const jump = this.getJumpAt(landing);
    if (jump) {
      return { position: jump.end, overshoot: false, jump };
    }
    return { position: landing, overshoot: false, jump: null };
  }
}

// -------------------------------------------------------------------------
// Observer pattern (recall 41 — Observer). Reporting is a SEPARATE concern
// from the game rules, so we push events out to observers instead of
// littering the Game with console.log. Swap ConsoleObserver for a GUI, a
// test spy, or a network broadcaster without touching game logic.
// -------------------------------------------------------------------------
class GameObserver {
  onMove(_player, _from, _to, _roll) {}
  onJump(_player, _jump) {}
  onOvershoot(_player, _roll) {}
  onWin(_player) {}
}

class ConsoleObserver extends GameObserver {
  onMove(player, from, to, roll) {
    console.log(`  ${player.name} rolled ${roll}: ${from} -> ${to}`);
  }
  onJump(player, jump) {
    const verb = jump.isLadder() ? 'climbed a LADDER' : 'hit a SNAKE';
    const arrow = jump.isLadder() ? '↑' : '↓';
    console.log(`    ${player.name} ${verb} ${arrow} ${jump.start} => ${jump.end}`);
  }
  onOvershoot(player, roll) {
    console.log(`  ${player.name} rolled ${roll}: overshoot, stays put`);
  }
  onWin(player) {
    console.log(`\n*** ${player.name} reached the final cell and WINS! ***`);
  }
}

// -------------------------------------------------------------------------
// Game — owns turn orchestration and the win check.
// Uses a queue (array with shift/push) so turn order cycles naturally.
// -------------------------------------------------------------------------
class Game {
  constructor(board, dice, players) {
    this.board = board;
    this.dice = dice;
    this.players = [...players]; // used as a rotating queue
    this.observers = [];
    this.winner = null;
  }

  addObserver(observer) {
    this.observers.push(observer);
    return this;
  }

  _notify(method, ...args) {
    for (const o of this.observers) o[method](...args);
  }

  /** Run one player's turn. Returns true if that player won. */
  takeTurn(player) {
    const roll = this.dice.roll();
    const from = player.position;
    const result = this.board.getNextPosition(from, roll);

    if (result.overshoot) {
      this._notify('onOvershoot', player, roll);
      return false;
    }

    // Report the raw move first, then any jump, so the story reads in order.
    const landedCell = result.jump ? result.jump.start : result.position;
    this._notify('onMove', player, from, landedCell, roll);
    if (result.jump) this._notify('onJump', player, result.jump);

    player.moveTo(result.position);

    if (player.hasWon(this.board.size)) {
      this.winner = player;
      this._notify('onWin', player);
      return true;
    }
    return false;
  }

  /** Main loop: cycle the queue, take turns, stop when someone wins. */
  play(maxTurns = 1000) {
    let turns = 0;
    while (!this.winner && turns < maxTurns) {
      const player = this.players.shift(); // dequeue front
      const won = this.takeTurn(player);
      this.players.push(player);           // re-enqueue at back
      turns++;
    }
    if (!this.winner) console.log('\nReached turn limit with no winner (safety stop).');
    return this.winner;
  }
}

// -------------------------------------------------------------------------
// BoardFactory (recall 30 — Factory Method). Board construction — deciding
// which snakes and ladders exist — is a separate responsibility from the
// board's runtime behaviour. The factory centralises "what a board looks
// like" so we can offer several named layouts.
// -------------------------------------------------------------------------
class BoardFactory {
  static classic100() {
    const ladders = [
      new Jump(2, 38), new Jump(7, 14), new Jump(8, 31),
      new Jump(15, 26), new Jump(28, 84), new Jump(36, 44),
      new Jump(51, 67), new Jump(71, 91), new Jump(78, 98),
    ];
    const snakes = [
      new Jump(16, 6), new Jump(46, 25), new Jump(49, 11),
      new Jump(62, 19), new Jump(64, 60), new Jump(74, 53),
      new Jump(89, 68), new Jump(92, 88), new Jump(95, 75),
    ];
    return new Board(100, [...ladders, ...snakes]);
  }

  /** A tiny board, handy for tests. */
  static small(size = 20) {
    return new Board(size, [
      new Jump(3, 11),  // ladder
      new Jump(6, 17),  // ladder
      new Jump(14, 4),  // snake
      new Jump(19, 8),  // snake
    ]);
  }
}

// -------------------------------------------------------------------------
// main() — REPRODUCIBLE demo. We inject a fixed sequence of die faces so
// the game plays out identically every run, proving the logic.
// -------------------------------------------------------------------------
function main() {
  console.log('=== Snake and Ladder — reproducible demo ===\n');

  // A canned sequence of single-die rolls. When the sequence runs out we
  // loop it, so the game always terminates deterministically.
  const scriptedRolls = [
    6, 2, 5, 3, 4, 6, 1, 5, 2, 3,
    4, 6, 5, 2, 3, 1, 4, 6, 2, 5,
    3, 4, 6, 1, 2, 5, 3, 6, 4, 2,
  ];
  let idx = 0;
  const roller = () => scriptedRolls[idx++ % scriptedRolls.length];

  const board = BoardFactory.classic100();
  const dice = new Dice(1, 6, roller); // one die, faces 1..6, scripted
  const players = [new Player('Alice'), new Player('Bob'), new Player('Carol')];

  const game = new Game(board, dice, players);
  game.addObserver(new ConsoleObserver());

  const winner = game.play();

  console.log('\nFinal positions:');
  for (const p of players) console.log(`  ${p.name}: cell ${p.position}`);
  console.log(`\nWinner: ${winner ? winner.name : 'none'}`);
}

// Run when executed directly (node snake-ladder.js), not when imported.
if (require.main === module) main();

module.exports = { Dice, Jump, Player, Board, Game, BoardFactory, ConsoleObserver };
```

**Why the injected roller matters.** `Math.random()` is fine for a real game, but a demo that prints a different story every run is useless for teaching or testing. By passing `roller` into `Dice`, we make the whole game deterministic without changing a single game rule. This is dependency injection buying us testability — the same trick you'd use to test any code that touches "randomness," time, or the network.

### 6. Design patterns used and WHY

Three ideas earn their place here — and notably, *no others do*. Restraint is part of the answer.

| Pattern / idea | Where | Why it helps |
|---|---|---|
| **The `Jump` abstraction** | `Jump` class | The real modelling win. Snakes and ladders are one type differing only by sign of `end - start`. No duplicated movement code, no `if (snake) ... else if (ladder) ...`. |
| **Factory** | `BoardFactory` | Board *construction* (which snakes/ladders exist) is a different job from board *behaviour*. The factory names layouts (`classic100`, `small`) and keeps the wiring out of `main`. Recall [30 — Factory Method](./30-pattern-factory-method.md). |
| **Observer** | `GameObserver` / `ConsoleObserver` | Reporting is decoupled from rules. The `Game` emits events; observers decide what to do (print, log to a test spy, broadcast over a socket). Recall [41 — Observer](./41-pattern-observer.md). |

**Be honest that this problem is deliberately simple.** The lesson is *clean modelling plus restraint*, not pattern-stacking. You could bolt on Strategy for movement rules, State for turn phases, Command for undo — and for base Snake and Ladder that would be textbook over-engineering (the "patternitis" from [30 — Factory Method](./30-pattern-factory-method.md)). Reach for a pattern when a real force demands it, not to decorate. Even Observer here is borderline: if the interviewer only wants `console.log`, say so and drop it. Knowing when *not* to add a pattern is a senior signal.

The clean split also leans on the four pillars from [13 — OOP Four Pillars](./13-oop-four-pillars.md): **encapsulation** (each class guards its own state — `Player.position`, `Board._jumps`), and **single responsibility** (Board = rules, Game = orchestration, Observer = reporting).

### 7. Extensions the interviewer asks for

The point of these is to show your design *absorbs change without rewrites*.

- **"Add multiple dice / a bonus roll on a 6."** Multiple dice: already supported — `new Dice(2)` sums two dice, no other change. Bonus roll: in `Game.play`, after `takeTurn`, if the last roll was a 6, don't rotate the queue (let the same player go again). That's a few lines in the loop; the board and player are untouched.
- **"Add different board configurations via a Factory."** Already there — add `BoardFactory.hardMode()` or load a layout from JSON. The `Game` doesn't care which layout it got, because it only calls `board.getNextPosition`.
- **"Add a 'crocodile' or other special cell type."** This is where the generic model pays off. Introduce a `Cell` with a `SpecialEffect`, or subclass `Jump` into effect types. A crocodile that eats a player (back to start) is just an effect that sets `position = 0`. Because movement already flows through `getNextPosition`, you extend the *effect table*, not the game loop. (This is the Strategy pattern arriving *only when a real force demands it*.)
- **"Send a player back on collision (two on one cell)."** After `player.moveTo`, scan for other players on that cell and reset them. Localised to `takeTurn` — one new rule, nothing else moves.
- **"Make it multiplayer over a network."** The Observer seam is your friend: add a `SocketObserver` that broadcasts each event to clients, and feed remote rolls through the injectable `roller`. The core `Game`/`Board`/`Jump` stay identical — only the I/O edges change.

Each extension touches *one seam*. That is the sign of a design that modelled the domain honestly.

---

## Visual / Diagram description

A single turn as a flow you can redraw on a whiteboard:

```
   ┌─────────────────────────────────────────────────────────┐
   │  Game.play() loop                                        │
   │                                                          │
   │   dequeue player  ──►  Dice.roll()  ──►  roll = e.g. 4   │
   │        ▲                                     │           │
   │        │                                     ▼           │
   │        │              Board.getNextPosition(pos, roll)   │
   │        │                     │                           │
   │        │            ┌────────┴─────────┐                 │
   │        │            ▼                  ▼                 │
   │        │      pos+roll > size?   pos+roll ≤ size         │
   │        │            │                  │                 │
   │        │       ┌────┘             ┌────┴─────┐           │
   │        │       ▼                  ▼          ▼           │
   │        │   OVERSHOOT        jump at cell?   no jump      │
   │        │   stay put            │              │          │
   │        │       │          ┌────┘              ▼          │
   │        │       │          ▼            move to cell      │
   │        │       │   move to jump.end          │          │
   │        │       │   (snake ↓ / ladder ↑)      │          │
   │        │       └──────────┬─────────────────┘           │
   │        │                  ▼                              │
   │        │         position === size ?                    │
   │        │            │ yes         │ no                   │
   │        │            ▼             ▼                      │
   │        │        WIN, stop    enqueue player ─────────────┘
   │        └───────────────────────────┘
   └─────────────────────────────────────────────────────────┘
```

Read top-to-bottom: dequeue a player, roll, ask the board for the next position. The board branches three ways — overshoot (stay), a jump on the landing cell (teleport up or down), or a plain move. Then check for a win; if none, put the player back at the tail of the queue and loop. Every branch is one method call, which is why the code stays short.

---

## Real world examples

### Ludo King / online board-game apps (representative)

Mobile board-game platforms model Snake-and-Ladder-style boards exactly this way: a fixed grid plus a lookup table of "special transitions." Representative of such systems, the board layout is data (often JSON) so new boards ship without new code, and an event stream (equivalent to our observers) drives animations and multiplayer sync. The rules engine stays tiny; the complexity lives in networking and rendering.

### Game engines and the "entity + effect" pattern (conceptual)

Beyond this one game, engines like Unity or Godot generalise our `Jump` insight into an **entity/component** model: a tile isn't a `Snake` or a `Ladder` subclass, it's a plain cell that *has an effect component*. Conceptually, our `getJumpAt` → apply-effect flow is the toy version of "query the tile for its components, apply them." Same idea, larger scale.

### Interview banks (LeetCode "Snakes and Ladders", problem 909)

The problem shows up in coding interviews as a *shortest-path* variant (fewest dice rolls to reach the end, solved with BFS over the board graph). That's a different question, but it shares our core model: cells as nodes, jumps as edges that reroute you. Recognising the same board model behind both the OO-design and the graph-algorithm framings is a nice cross-connection to mention.

---

## Trade-offs

**Modelling snakes and ladders as one `Jump` type:**

| Pros | Cons |
|---|---|
| Zero duplicated movement logic | Slightly less "literal" — a junior reader might expect a `Snake` class |
| New jump-like features (crocodile, teleporter) slot in easily | Sign-based distinction (`end < start`) is implicit; needs a clear `isSnake()` helper |
| Half the code, half the bug surface | You must document the convention so nobody adds a same-cell jump |

**Injecting the dice roller (dependency injection):**

| Pros | Cons |
|---|---|
| Fully reproducible demos and tests | One more constructor argument |
| Swap random ↔ scripted ↔ network with no rule changes | Slight indirection for a first-time reader |

**Using the Observer pattern for reporting:**

| Pros | Cons |
|---|---|
| Rules stay free of `console.log` | Overkill if you only ever print to console |
| Add GUI/network/test-spy outputs without touching rules | Extra indirection; more classes to read |

**The sweet spot:** keep the `Jump` abstraction and the injectable roller — they're cheap and pay off immediately. Add Observer and Factory only when the interviewer signals they want extensibility (multiplayer, multiple boards); otherwise say out loud "for this scope, plain `console.log` and inline construction are fine," and let the simplicity stand. Naming the restraint is itself the senior move.

---

## Common interview questions on this topic

### Q1: "Why model a snake and a ladder as the same class?"

**Hint:** Because they *are* the same operation — a teleport from one cell to another — differing only in direction. A ladder has `end > start`, a snake has `end < start`. Two classes would duplicate all the movement logic and force `if (snake) / else if (ladder)` branches everywhere. One `Jump` with `isSnake()`/`isLadder()` helpers is DRY and makes new "jump-like" features (crocodile, teleporter) trivial.

### Q2: "How do you handle a roll that overshoots the final cell?"

**Hint:** First clarify the variant — "stay put" or "bounce back." For stay-put: in `getNextPosition`, if `current + roll > size`, return the current position unchanged. For bounce-back: reflect the overshoot, `landing = size - (landing - size)`. The key point is that this rule lives in *one place* (the board), so switching variants is a one-line change.

### Q3: "Where does the turn-order logic live, and why?"

**Hint:** In the `Game`, using a queue (dequeue the front player, take their turn, enqueue them at the back). It lives in `Game` because turn rotation is orchestration, not a board rule and not player state. Keeping it out of `Board` lets the board stay a pure, testable rules engine.

### Q4: "The interviewer says: now a ladder can drop you onto a snake head. Chain the jumps?"

**Hint:** First clarify — standard rules say *no chaining* (one jump per landing), which is what we implement. If they *want* chaining, loop in `getNextPosition`: after applying a jump, check the new cell for another jump and repeat (guarding against infinite loops with a visited set). Note the risk: a poorly designed board could cycle forever, so you'd cap or validate it.

### Q5: "How would you make this testable and reproducible?"

**Hint:** Inject the die roller. Instead of hard-coding `Math.random`, pass a function that returns die faces. In tests/demos, feed a scripted sequence so the game plays out identically every run. This is dependency injection isolating the one non-deterministic dependency — the same technique you'd use for time or network calls.

---

## Practice exercise

### Build it and add the "crocodile" cell

Type the implementation above into `snake-ladder.js` and run it with `node snake-ladder.js` to confirm you get a deterministic winner. Then extend it (about 30 minutes):

1. Add a **crocodile** special cell: landing on it sends the player all the way back to cell `1` (or `0`, off the board). Decide whether to model it as a `Jump(cell, 1)` (reusing your abstraction!) or as a new effect type — and justify your choice in a comment.
2. Add a **bonus roll on a 6**: if a player rolls a 6, they take another turn immediately (don't rotate the queue). Cap consecutive bonus rolls at 3 to avoid a runaway turn.
3. Add a second `BoardFactory.hardMode()` layout with more snakes than ladders, and run the same three players on it with the same scripted rolls.

**What to produce:** the modified file, plus a two-line written note answering "did the crocodile fit into `Jump`, or did it need a new type — and what does that tell you about the abstraction?" If it fit cleanly, your model was honest; if it strained, note exactly where.

---

## Quick reference cheat sheet

- **The one insight:** a snake and a ladder are the *same object* — a `Jump(start, end)`. Ladder = `end > start`, snake = `end < start`.
- **Board = rules engine.** It owns the jump lookup and `getNextPosition`; no printing, no turn logic.
- **Game = orchestrator.** It owns the turn queue and the win check; it delegates movement to the board.
- **Dice = injectable.** Pass a `roller` function for reproducible demos/tests; default to `Math.random` in production.
- **Overshoot rule** lives in one place — clarify "stay put" vs "bounce back" with the interviewer first.
- **One jump per landing** is the standard rule (no chaining) unless told otherwise.
- **Factory** builds named board layouts, separating construction from behaviour. (Recall 30.)
- **Observer** decouples reporting from rules — swap console for GUI/network/test-spy. (Recall 41.)
- **Restraint is the lesson:** this problem is simple; don't bolt on Strategy/State/Command you don't need — that's patternitis.
- **Encapsulation & SRP** (recall 13): each class guards its own state and does one job.
- **Turn order = a queue:** `shift()` to dequeue, `push()` to re-enqueue at the back.
- **Extensions touch one seam each** — dice count, board layout, special cells, networking — proof of an honest model.
- **Cross-connection:** LeetCode 909 solves the same board as a BFS shortest-path problem.
- **Rule of thumb:** find the shared abstraction first, inject your non-determinism, and add patterns only when a real force demands them.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [119 — LLD Chess Game](./119-lld-chess-game.md) | The harder game-modelling case study; Snake and Ladder is the gentler warm-up with the same object-modelling muscles. |
| **Next** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The step-by-step method (requirements → nouns → verbs → classes → code → patterns → extensions) that we followed here. |
| **Related** | [30 — Factory Method](./30-pattern-factory-method.md) | The `BoardFactory` that builds board layouts, and the source of the "patternitis / don't over-engineer" warning. |
| **Related** | [41 — Observer](./41-pattern-observer.md) | The event-announcement mechanism that decouples reporting from the game rules. |
| **Related** | [13 — OOP Four Pillars](./13-oop-four-pillars.md) | Encapsulation and single-responsibility — why Board, Game, and Observer are cleanly separated. |
