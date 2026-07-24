# Grandmaster AI — How It Works

A detailed, step-by-step explanation of the single-file chess app in `index.html`.

The app is a browser-based chess game where a human plays against the **Stockfish** chess engine. It uses three third-party libraries:

| Library | Purpose |
| --- | --- |
| [chess.js](https://github.com/jhlywa/chess.js) | Pure-JS chess rules engine: tracks the game state, validates moves, detects check / checkmate / draw, maintains move history, exports FEN. |
| [chessground](https://github.com/lichess-org/chessground) | The interactive visual board (the same one Lichess uses). Renders pieces, handles drag-and-drop, highlights legal moves, plays move animations. |
| [stockfish.js](https://github.com/nmrugg/stockfish.js) + `stockfish.wasm` | The actual chess AI. A WebAssembly port of the Stockfish engine that runs in a Web Worker. |
| [PeerJS](https://peerjs.com/) | Peer-to-peer WebRTC connection for the "Play with Friend" mode. The host gets a PeerJS peer ID; the joiner dials it. |

UI styling is done with [Tailwind CSS](https://tailwindcss.com/) (loaded from CDN).

---

## 1. Page Layout

The HTML is a two-column layout on large screens, stacked on mobile:

- **Left column** — the chess board (`#board`) and a status pill above it ("Your Turn", "Stockfish is thinking...", "Game Over").
- **Right column** — three panels:
  1. **Game Settings** — choose your color, difficulty slider, and three action buttons (Undo, Flip, New Game).
  2. **Move History** — a scrollable list of moves in algebraic notation, two columns (white's move / black's move).
  3. **Debug Console** (hidden by default) — a textarea that logs every message received from the Stockfish worker. Useful for debugging.

There's also a hidden `#game-over-overlay` that fades in when the game ends.

---

## 2. Module-Level State

At the top of the `<script type="module">` block:

```js
let engine = null;          // the Stockfish Web Worker
let game = new Chess();     // chess.js game state
let board = null;           // chessground instance
let playerColor = 'white';  // which color the human plays
let boardOrientation = 'white'; // which way the board faces
let isEngineReady = false;  // becomes true after 'uciok'
let pendingAiMove = false;  // set if user moved before engine was ready
let suppressNextBestMove = false; // used by Undo to discard stale AI responses
let lastMove = undefined;   // [from, to] for the green "last move" highlight
```

---

## 3. Initialization (on `DOMContentLoaded`)

1. Build a chessground instance attached to `<div id="board">`.
2. Call `syncBoard({ animate: false })` to draw the starting position.
3. Call `initEngine()` to start the Stockfish worker.
4. Wire up all the button click handlers.

---

## 4. Board Rendering — `syncBoard(opts)`

`syncBoard` is the single source of truth for what the board shows. It does two things:

### 4.1 Configure chessground

It calls `board.set({...})` with a config object built from the current `game` state:

- **`fen: game.fen()`** — tells chessground which pieces are where.
- **`orientation`** — `'white'` or `'black'`, based on the user's preference (set by Flip).
- **`turnColor`** — whose turn it is (used for arrow / check indicator).
- **`check`** — true if the side to move is in check.
- **`lastMove`** — `[from, to]` of the last move, for the green/yellow highlight.
- **`movable.color`** — `playerColor` if it's the user's turn, otherwise `undefined` (locked).
- **`movable.dests`** — a `Map<square, [legalDestSquares]>` computed by `getLegalDests()`. This is what makes chessground draw the little dots on legal target squares.
- **`movable.events.after`** — a callback that fires when the user drops a piece: `onUserMove`.

### 4.2 Update the status pill

```js
if (game.game_over())               → "Game Over" (red)
else if (turn === playerColor)      → "Your Turn" (green)
else                                → "Stockfish is thinking..." (amber, pulsing)
```

---

## 5. The User Makes a Move — `onUserMove(orig, dest)`

chessground calls this when the user drops a piece. The flow:

1. **Promotion detection:** if the moving piece is a pawn and the destination is on the back rank, default-promote to a queen.
2. **Validate with chess.js:** `game.move({from, to, promotion})`. If it returns `null`, the move was illegal — call `syncBoard({ animate: false })` to snap the piece back and return.
3. **Update internal state:** store the move in `lastMove` for the highlight.
4. **Re-render:** call `syncBoard({ animate: true })` so chessground animates the move.
5. **Update the move log.**
6. **If the game isn't over, ask the AI to move:** call `requestBestMove()`. If it is over, call `handleGameOver()`.

---

## 6. Asking Stockfish for a Move — `requestBestMove()`

```js
engine.postMessage(`position fen ${game.fen()}`);  // current position
engine.postMessage(`go depth ${depth + 2}`);        // search
```

- The "depth" is `userDifficulty + 2` (the slider's raw value is interpreted as a skill offset, but `go depth` uses an absolute depth, so a level-5 slider means "search 7 plies deep").
- If the engine isn't ready yet (`isEngineReady === false`), the request is deferred by setting `pendingAiMove = true`. The engine will drain this queue when it sends `uciok`.

## 7. Stockfish Replies — `handleEngineMessage(event)`

The worker is line-oriented (UCI protocol). Every line of text it produces arrives as a string in `event.data`.

- **`uciok`** — first message after `uci`. Sets `isEngineReady = true`, applies the current skill level, and either drains a queued move request or syncs the empty starting board.
- **`bestmove <long-algebraic>`** — the engine finished searching. We extract the move (e.g. `e2e4`, `e7e8q` for a promotion to queen) and call `makeAiMove()`. If `suppressNextBestMove` is true (Undo is in progress), the message is dropped.
- **Anything starting with `info `** is ignored except for being optionally logged.
- **Any non-`bestmove` line clears `suppressNextBestMove` defensively**, in case `stop` didn't produce one and the flag would otherwise eat a real response.

## 8. Stockfish Plays the Move — `makeAiMove(bestMove)`

1. Parse the UCI move (`e2e4` → from=`e2`, to=`e4`; if 5 chars, the 5th is the promotion piece).
2. Apply it to `game` via `game.move(...)`. If chess.js rejects it (shouldn't happen), log and return.
3. Update `lastMove`, animate, update the move log.
4. If the game is now over, show the game-over overlay. Otherwise it's the user's turn — `syncBoard` will set the status to "Your Turn" and re-enable `movable.color`.

---

## 9. The "Difficulty" Slider

```js
function applySkillLevel() {
    const depth = parseInt(difficultySlider.value, 10); // 1..10
    const skillLevel = (depth - 1) * 2;                 // 0, 2, 4, ... 18
    engine.postMessage(`setoption name Skill Level value ${skillLevel}`);
}
```

`Skill Level` is Stockfish's UCI option (0 = weakest, 20 = strongest). The map is:
- Slider 1 → Skill 0 (very weak, makes blunders)
- Slider 5 → Skill 8 (default)
- Slider 10 → Skill 18 (very strong, near-maximum)

The label "Level N" displayed in the UI is the slider's raw value, not the underlying Stockfish skill.

---

## 10. Undo — `undoMove()`

Undoing in this app is tricky because there are usually **two moves on the stack** for each "round" — one by you, one by the AI. The goal is: after pressing Undo, it's your turn and the position is one full ply-pair earlier (or one ply earlier if you undo right after your own move, before the AI has replied).

### The logic

```js
if (turn === playerColor) {
    // AI just replied → undo AI's move AND the user's preceding move
    game.undo();
    if (game.history().length > 0) game.undo();
} else {
    // User just moved, AI hasn't yet → undo only the user's move
    game.undo();
}
```

### The race condition

If the user clicks Undo **while the engine is still thinking** (very common — the button is right there after you move), the in-flight search will eventually send a `bestmove`. That `bestmove` would be applied to the *pre-undo* position, corrupting the board and locking the user's input.

To prevent this:

1. We set `suppressNextBestMove = true` and send `stop` to the worker.
2. When the next `bestmove` arrives, we discard it and clear the flag.
3. Any other engine output (like `info depth ...`) also clears the flag defensively.

### After the undo

After undoing, we determine whose turn it now is. If it's the AI's, we send a fresh `position fen ...` and `go depth N` so Stockfish plays the next move automatically. The `position fen` is important because it fully resets the engine's internal state to the new position, regardless of whether the previous search has fully wound down.

---

## 11. New Game — `resetGame()`

1. Replace the chess.js game with a fresh one.
2. Clear `lastMove` and the move log.
3. Hide the game-over overlay.
4. Tell the engine to start a new game (`ucinewgame`).
5. Resync the board.
6. If the user picked **Black**, immediately request the AI's first move (otherwise White/AI opens).

## 12. Flip Board — `flipBoard()`

Calls `board.toggleOrientation()` (chessground's built-in) and flips our local `boardOrientation` variable so `syncBoard` continues to apply the right orientation on every update.

## 13. Choose Color — `setPlayerColor(color)`

Updates `playerColor`, toggles the active state on the White/Black buttons, and calls `resetGame()`. If the new color is Black, the AI makes the first move.

---

## 14. Game Over — `handleGameOver()`

Determines the result and shows the modal:

- **Checkmate:** announces the winner (the side *not* to move).
- **Draw:** generic "Draw!" — could be stalemate, threefold, 50-move, insufficient material, etc. (chess.js's `in_draw()` covers all of these.)
- **Otherwise:** "Game ended." (safety fallback)

The "Play Again" button just calls `resetGame()`.

---

## 15. UCI Protocol Cheat Sheet

For reference, here are the only UCI messages this app sends and handles:

**Sent by the app:**

| Command | When |
| --- | --- |
| `uci` | Once, at startup, to initialize the engine. |
| `setoption name Skill Level value N` | After `uciok` and whenever the slider changes. |
| `ucinewgame` | At the start of every new game. |
| `position fen <FEN>` | Before every `go` — sets the position to search. |
| `go depth N` | Asks the engine to search to depth N plies. |
| `stop` | Sent on Undo, to halt any in-progress search. |

**Received from the engine:**

| Response | Meaning |
| --- | --- |
| `uciok` | Engine is ready for commands. |
| `info ...` | Search progress (depth, score, principal variation). Logged but not acted on. |
| `bestmove <move>` | Search is done. The `<move>` is in long algebraic notation (e.g. `e2e4`, `e7e8q`). |

---

## 16. Multiplayer — Play with Friend (PeerJS)

In addition to playing against Stockfish, the app supports peer-to-peer play between two humans. There is no server: the two browsers talk directly via WebRTC, with PeerJS's public broker only used to negotiate the initial connection (it never sees the game moves).

### Flow

1. One player clicks **Play with Friend** in the right-hand panel. A modal opens.
2. They click **Create Room**. The app generates a 6-character room code (uppercase, ambiguous characters like `I`, `O`, `0`, `1` are excluded for readability) and registers itself on the PeerJS broker with peer ID `chess-<roomcode>`. A short invite link like `https://your-site.com/?room=ABC123` is also generated.
3. They share the code (or link) with their friend out of band (chat, email, etc.).
4. The friend opens the same page. If they used the link, the app auto-opens the modal with the code pre-filled. Otherwise they click **Play with Friend** → **Join with Room ID** → enter the code → **Connect**.
5. The joiner's browser asks the PeerJS broker to look up `chess-<roomcode>`. The broker returns the host's WebRTC connection details; the two browsers negotiate a direct connection.
6. When the connection is `open`, both sides enter **friend mode**:
   - The host plays **White**.
   - The joiner plays **Black**.
   - The board is automatically oriented so each player sees their own pieces at the bottom.
   - The AI controls (difficulty slider, "Play as" buttons) are hidden, because they don't apply.

### Move synchronization

When Player A drags a piece:

1. `onUserMove` applies the move locally to the chess.js game.
2. Instead of calling `requestBestMove()`, it sends a `{type: 'move', from, to, promotion}` message over the data connection.
3. The peer receives it in `handlePeerMessage`, applies the move to its own chess.js game, and re-renders the board.
4. `syncBoard` on the peer sees it's now that peer's turn and unlocks the board for them.

The chess.js game state on both sides is always identical, so no state syncing is needed beyond the move messages.

### Messages exchanged

| Type | Direction | Meaning |
| --- | --- | --- |
| `hello` | both → both | Sent on connect. Carries the sender's color. |
| `move` | sender → peer | A move was played locally. Includes `from`, `to`, optional `promotion`. |
| `newgame` | either → other | Both sides reset their chess.js game. |
| `undo-request` | requester → peer | Asks the peer to undo. Auto-accepted on receipt. |
| `undo-done` | acceptor → requester | Confirms the undo; both sides then re-render. |
| `resign` | either → other | Concedes the game. |

### Room code generation

```js
const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789'; // 32 chars (no I, O, 0, 1)
```

6 characters from a 32-char alphabet gives ≈ 1 billion possible codes, so collisions are vanishingly rare. The code is embedded in the PeerJS peer ID (`chess-abc123`) so the joiner can look it up without out-of-band registration.

### Leaving friend mode

Clicking the friend button while in friend mode tears down the connection (`conn.close()` + `peer.destroy()`), restores the AI controls, and starts a fresh local game against Stockfish.

---

## 17. Data Flow Summary

```
User drags piece
       │
       ▼
chessground → onUserMove(orig, dest)
       │
       ├─ game.move(...)         ← chess.js validates
       ├─ syncBoard()            ← chessground re-renders
       └─ requestBestMove()      ← post 'position fen' + 'go depth'
                                      │
                                      ▼
                              Stockfish worker
                                      │
                                      ▼
                  'bestmove e2e4' arrives in handleEngineMessage
                                      │
                                      ▼
                              makeAiMove('e2e4')
                                      │
                                      ├─ game.move(...)    ← chess.js applies
                                      ├─ syncBoard()       ← chessground animates
                                      └─ status → "Your Turn"
```

The chess.js `game` object is the single source of truth for the game state. chessground only knows what to draw; the engine only knows what to search. All three are kept in sync through `syncBoard()` and the small set of functions that mutate the game.
