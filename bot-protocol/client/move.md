---
title: Move request
---

# Move request

The client sends a move request before each piece. The request carries the current board and the piece to place.

Every move request is authoritative. The client sends the board width, height, and rows each time, and the bot must not cache them from an earlier request. Some modes join boards during a round, so the width can grow.

Messages are one JSON object per line. See [transport](<transport>) for framing and flushing.

## Glossary

| Term | Meaning |
|------|---------|
| Board | The settled cells. The current piece is not part of it. |
| Visible board | The rows the player can see. Its height is `board.height`. |
| Row | One horizontal line of cells. `board.rows[0]` is the bottom row. |
| Cell | One square. `null` is empty. Any string is occupied. |
| Current piece | The piece the bot is placing. |
| Queue | The upcoming pieces the client exposes. |
| Attack multiplier | The multiplier applied to attack. 0 means no attack. |
| Live value | A value that can change during the round. |

## Message

```json
{"command":"move","data":{"board":{"width":4,"height":3,"rows":[[null,null,null,null],[null,null,null,null],[null,null,null,null]]},"current":{"type":"T","rotation":0,"cells":[[1,1],[0,1],[2,1],[1,2]]},"queue":[],"attackMultiplier":0}}
```

The example uses a small board so the line stays readable. A real board is wider and taller.

Every field is required. Empty and not applicable values are explicit: an empty queue is `[]`, and no attack is `0`.

## Fields

| Field | Meaning |
|------|---------|
| `board.width` | Current width. |
| `board.height` | Visible height. Rows at this index and above are hidden. |
| `board.rows` | Settled cells, bottom row first. |
| `current.type` | Piece type. |
| `current.rotation` | Rotation state, 0 at spawn. |
| `current.cells` | Occupied cells as `[x, y]`, x from the left and y from the bottom. |
| `queue` | Upcoming piece types, nearest first. Empty when there is no preview. |
| `attackMultiplier` | Current attack multiplier. 0 when there is no attack. |

## Board

`board.rows` is a 2D array, indexed `rows[y][x]`. `board.rows[0]` is the bottom row, and `board.rows[1]` is the row above it. `x` starts at 0 on the left.

`board.height` is the number of visible rows. The rows at index `height` and above are hidden rows above the visible board. The client may send them or stop at the visible height. `rows` must contain at least `height` rows. The bot uses `height` as the lockout boundary, not the number of rows in the array.

The board holds settled cells only. The current piece is separate. The client does not merge the piece into the board.

A cell is `null` when empty. Any string means occupied. The bot only needs to know that the cell blocks movement. The platform page defines what each string means, such as garbage or a locked piece.

## Current piece

`current.cells` lists the occupied cells of the piece as `[x, y]`, with `x` from the left and `y` from the bottom. The cells use the same grid as `board.rows`. They may be above the visible board; their `y` can be `height` or greater.

`current.rotation` is the rotation state index, starting at 0 for the spawn state. The platform page defines the states and the kicks between them.

Sending absolute cells means the bot does not need the platform's piece origin or bounding box to find the piece on the board.

## Neutral values

`queue` lists the upcoming piece types, nearest first. The current piece is not in the queue. An empty array means the client exposes no preview, or there are no more pieces. The bot must not assume that a next piece exists.

`attackMultiplier` is the multiplier applied to attack for this move. `0` means the round has no attack, or the bot should ignore attack. If the platform has attack, the client sends the current multiplier.

## Reply

The bot replies with one move message. The client applies the move and sends the next request after the piece locks.
