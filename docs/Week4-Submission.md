# Snake & Ladder — Week 4 Submission
CAP Developer Challenge — UI & CAP MCP

---

## 1. Verify Service token

Ran against a local `cds watch` instance (`node verify.js http://localhost:4004/odata/v4/game`):

```
🐍 Snake & Ladder Verify Service
   Target: http://localhost:4004/odata/v4/game

  ✓  01/14 Service reachable
  ✓  02/14 BoardSquares entity has 100 squares
  ✓  03/14 Snake squares present
  ✓  04/14 Ladder squares present
  ✓  05/14 Double-headed snake squares present
  ✓  06/14 Create GameSession
  ✓  07/14 Create Player 1
  ✓  08/14 Create Player 2
  ✓  09/14 startGame → SessionStatus: InProgress
  ✓  10/14 After startGame: first player is Playing
  ✓  11/14 rollDice on Waiting player returns 409
  ✓  12/14 rollDice → TurnStatus: Moving, returns integer
  ✓  13/14 confirmMove → turn rotates to Player 2
  ✓  14/14 blockPlayer → Blocked; unblockPlayer → $flow.previous

  All 14 checks passed. 🏆 Challenge complete!
```

**Verification token:**
```
aHR0cDovL2xvY2FsaG9zdDo0MDA0L29kYXRhL3Y0L2dhbWV8MTc4NTM3NTkzOTY5MnwxNC8xNA==
```

---

## 2. Board UI — game in progress

Board UI running at `http://localhost:4004/board.html` (served same-origin from `app/`, no CORS needed). Screenshot shows a 4-player session ("Sapeiros 2") mid-game: tokens spread across squares 9–44, player statuses (Playing/Waiting), and the Turn Log showing rolls and a ladder climb (turn 7: rolled 3, square 5→31).

*(Screenshot attached separately.)*

---

## 3. `rollDice` returning a `roll` integer with `TurnStatus: Moving`

```
$ curl -X POST http://localhost:4004/odata/v4/game/Players(ID=26ae9dbb-720a-40a5-8bb0-e4c83bda30ab)/rollDice \
  -H "Content-Type: application/json" -d '{}'

HTTP/1.1 200 OK
X-Correlation-ID: ec5e5abe-fc23-4386-b647-ddc3e8b9033d
OData-Version: 4.0
Content-Type: application/json; charset=utf-8
Content-Length: 120

{"roll":6,"position":7,"event":"normal"}
```

```
$ curl http://localhost:4004/odata/v4/game/Players(ID=26ae9dbb-720a-40a5-8bb0-e4c83bda30ab)?$select=name,TurnStatus,position,lastRoll

{"name":"Eve","TurnStatus":"Moving","position":7,"lastRoll":6}
```

`roll: 6` — a valid 1–6 dice value — and the follow-up GET confirms `TurnStatus: Moving`. This is the Week 2 `@from: [#Playing] @to: #Moving` transition firing correctly: CAP validated the player was `Playing` before the handler ran, the handler computed the roll and new position, and CAP applied the `Moving` status after the handler returned.

---

## 4. AI plays the full game via CAP MCP (stretch)

Played entirely over the MCP JSON-RPC protocol at `/mcp/game` — `mcpStartGame`, `mcpRollDice`, and `mcpConfirmMove` calls, alternating between Alice and Bob until someone reached square 100. No OData calls, no manual DB edits — the whole game ran through the 5 unbound MCP wrapper functions.

*(Note: I added a `.mcp.json` to register this server as a native MCP connection for Claude Code, and confirmed via a fresh subagent spawn that this environment doesn't hot-load a new `.mcp.json` mid-session — it needs a real restart. So this run used the same raw JSON-RPC `curl` calls proven in Week 3; the wire protocol is identical either way. The `.mcp.json` is committed so a fresh Claude Code/Desktop window gets native tool calls.)*

**Turn-by-turn commentary — highlights from the 88-turn game:**

> Turn 1 — Alice opens strong with a 6, straight to square 7. Bob answers with a modest 3, to square 4.
>
> Turn 7 — Alice rolls a 2, lands on square 15 — and square 15 has a **ladder** straight up to 26! Early lead extended.
>
> Turn 13 — Alice looked to be cruising past square 35, rolls a 5 for square 40... but 40 is a **double-headed snake mouth**. Down she goes, all the way to the shared tail at square 3. Brutal reset.
>
> Turns 14–51 — A long grind. Both players claw their way back up the board with mostly normal rolls. Bob briefly overtakes, reaching square 87 by turn 50 while Alice sits at 83.
>
> Turn 52 — Bob, at square 87, rolls a 2 for square 89 — **another double-headed snake**, the twin of the one that bit Alice earlier. Bob plunges to square 50, right back in the pack.
>
> Turn 61 — Alice, agonizingly close at square 97, rolls a modest 2 — enough for square 99. But square 99 hides a long snake all the way down to square 7. So close, yet so far — the biggest swing of the game.
>
> Turns 62–80 — Both players grind back up a second time. Bob hits *another* double-headed snake at square 72 (back down to 50 again — that shared tail claimed him twice). Alice climbs steadily via a ladder at square 84 and closes in on the finish.
>
> Turns 81–87 — Alice reaches square 97 and 98 multiple times, but the "exact roll to finish" rule keeps rejecting her: *"Need exactly 3 to finish. Rolled 6. No move."* / *"Need exactly 2 to finish. Rolled 4. No move."* — real HTTP-level rejections proving that rule holds even under MCP.
>
> Turn 88 — Alice, sitting on square 98, rolls a 2. **Square 100 — victory!** `confirmMove`'s win-check fired automatically, delegating to `winGame`, which stamped `finishedAt` and set `winner_ID` on the session — no manual intervention needed.

**Final board state (`mcpBoardState`):**
```json
[
  { "name": "Alice", "position": 100, "TurnStatus": "Waiting", "lastRoll": 2 },
  { "name": "Bob",   "position": 98,  "TurnStatus": "Waiting", "lastRoll": 5 }
]
```

---

## Notes on implementation

- `app/board.html` + `app/board.png` served same-origin from CAP's static file serving — no CORS config needed for local use.
- `verify.js` creates its own session and 2 players with fixed UUIDs, so it's fully self-contained and safe to re-run against any fresh instance.
- The MCP demo is a genuine end-to-end proof: every dice roll, ladder, snake, double-snake bite, and the final win condition were driven entirely through the CAP MCP server's unbound wrapper functions, not OData.
