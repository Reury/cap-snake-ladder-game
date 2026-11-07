# Snake & Ladder — Week 3 Submission
CAP Developer Challenge — Multiplayer & Events

---

## 1. Two players after `confirmMove` — one `Waiting`, one `Playing`

Captured in Postman (see attached screenshot) against session `c29462b8-d52a-41b3-b767-04e978d2416d`:

```
GET http://localhost:4004/odata/v4/game/Players?$select=name,TurnStatus,position,turnOrder

{
  "value": [
    { "name": "Bob the cat", "TurnStatus": "Playing", "position": 1, "turnOrder": 1 },
    { "name": "Alice",       "TurnStatus": "Waiting", "position": 6, "turnOrder": 1 }
  ]
}
```

`confirmMove` rotated the turn without any manual PATCH — CAP applied `@to: #Waiting` to Alice, and the handler's turn-rotation logic activated Bob directly.

---

## 2. `mcpRollDice` returning `{roll, position, event}` — not null

Called over the MCP JSON-RPC transport (not OData), against the same session's player `Bob the cat` (`a3edb677-ab81-49cc-8f70-0881daa62931`):

```
curl -X POST http://localhost:4004/mcp/game \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"mcpRollDice","arguments":{"playerID":"a3edb677-ab81-49cc-8f70-0881daa62931"}}}'
```

Response:
```json
{
  "result": {
    "structuredContent": {
      "action": "mcpRollDice",
      "kind": "function",
      "result": { "roll": 5, "position": 6, "event": "normal" }
    }
  }
}
```

Real data, not `null` — proves the MCP action tools are wired correctly and `this.send()` is using the object form (`{event, entity, params}`), not the string form that silently misroutes the player ID.

`tools/list` confirms all 7 expected tools are exposed: `query`, `describe`, `mcpStartGame`, `mcpRollDice`, `mcpConfirmMove`, `mcpBoardState`, `mcpBoardMap`.

---

## 3. Querying game state via MCP

Queried the live game directly over the MCP endpoint using the `mcpBoardState` tool:

```
curl -X POST http://localhost:4004/mcp/game \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"mcpBoardState","arguments":{"sessionID":"c29462b8-d52a-41b3-b767-04e978d2416d"}}}'
```

```json
{
  "result": [
    { "name": "Alice",       "position": 6, "TurnStatus": "Playing", "lastRoll": 5 },
    { "name": "Bob the cat", "position": 6, "TurnStatus": "Waiting", "lastRoll": 5 }
  ]
}
```

**Who's winning?** It's currently a tie — both players are on square 6. It's Alice's turn to roll next (`Playing`), while Bob the cat waits.

---

## 4. Turn log (stretch — `mcpBoardState` + `TurnLog`)

```
GET http://localhost:4004/odata/v4/game/TurnLog?$filter=session_ID eq c29462b8-d52a-41b3-b767-04e978d2416d&$orderby=turnNumber
```

```json
[
  { "turnNumber": 1, "player_ID": "bb090386-...", "diceRoll": 5, "fromSquare": 1, "toSquare": 6, "eventType": "normal" },
  { "turnNumber": 2, "player_ID": "a3edb677-...", "diceRoll": 5, "fromSquare": 1, "toSquare": 6, "eventType": "normal" }
]
```

**Commentary:** Alice opens with a roll of 5, moving from square 1 to 6. Bob the cat answers with an identical roll of 5, landing on the very same square. Dead even after round one — no snakes or ladders in play yet.

---

## Notes on implementation

- Two independent `@flow.status` machines run side by side: `GameSessions.SessionStatus` (`Lobby → InProgress → Complete`) governs the game lifecycle, `Players.TurnStatus` governs each player's turn.
- `confirmMove` checks `player.position === 100` and delegates to `winGame` via `this.send({event:'winGame', entity:'Players', params:[{ID}]})` before doing any turn-rotation work.
- `rollDice` persists `prevPosition` and `lastEventType` so `confirmMove` can write an accurate `TurnLog` row without re-deriving state.
