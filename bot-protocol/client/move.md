---
title: Move request
---

# Move request

The client sends a move request before each piece. The request carries the current board and the piece to place, and the bot replies with a placement.

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
| Anchor | The `x` and `y` of a piece: the smallest x and the smallest y among its cells. |
| Shape | The cell offsets for a piece type at one rotation, normalized to a smallest x and y of 0. |
| Hold | The piece set aside for a later swap. |
| Queue | The upcoming pieces the client exposes. |
| Attack multiplier | The multiplier applied to attack. 0 means no attack. |
| Spin | A placement that the round's rules count as a spin. |
| Live value | A value that can change during the round. |

## Coordinates

`x` starts at 0 on the left, `y` starts at 0 at the bottom, and `rotation` starts at 0 for the spawn state. Each clockwise quarter turn adds 1 to `rotation`, modulo 4. `board.rows`, the current piece, and the reply all use this grid and index.

A piece's anchor is the pair of the smallest x and the smallest y among its occupied cells. The point `(x, y)` does not have to be occupied.

For a piece type and rotation, the shape is the set of cell offsets from the anchor. The offsets are normalized so their smallest x and smallest y are 0. The occupied cells are `(x + dx, y + dy)` for each offset. This works for any piece shape, including pentominoes.

The platform profile defines the shape for each piece type and rotation. The client and the bot use the same shapes.

## Message

```json
{"command":"move","data":{"board":{"width":4,"height":3,"rows":[[null,null,null,null],[null,null,null,null],[null,null,null,null]]},"current":{"type":"T","x":1,"y":1,"rotation":0},"hold":null,"canHold":true,"queue":[],"attackMultiplier":0}}
```

The example uses a small board so the line stays readable. A real board is wider and taller.

Every field is required. Empty and not applicable values are explicit: an empty queue is `[]`, an empty hold is `null`, and no attack is `0`.

## Fields

| Field | Meaning |
|------|---------|
| `board.width` | Current width. |
| `board.height` | Visible height. Rows at this index and above are hidden. |
| `board.rows` | Settled cells, bottom row first. |
| `current.type` | Piece type. |
| `current.x` | Smallest x among the piece's cells. |
| `current.y` | Smallest y among the piece's cells. |
| `current.rotation` | Rotation state. |
| `hold` | Held piece type, or `null` when hold is empty. |
| `canHold` | Whether the client allows a hold this turn. |
| `queue` | Upcoming piece types, nearest first. Empty when there is no preview. |
| `attackMultiplier` | Current attack multiplier. 0 when there is no attack. |

## Board

`board.rows` is a 2D array, indexed `rows[y][x]`, using the coordinates above. `board.rows[0]` is the bottom row, and `board.rows[1]` is the row above it.

`board.height` is the number of visible rows. The rows at index `height` and above are hidden rows above the visible board. The client may send them or stop at the visible height. `rows` must contain at least `height` rows. The bot uses `height` as the lockout boundary, not the number of rows in the array.

The board holds settled cells only. The current piece is separate. The client does not merge the piece into the board.

A cell is `null` when empty. Any string means occupied. The bot only needs to know that the cell blocks movement. The platform page defines what each string means, such as garbage or a locked piece.

## Current piece

The current piece uses the anchor defined above. The client sends `type`, `x`, `y`, and `rotation`, not cells. The platform profile defines the shape for the type and rotation, so the bot can reconstruct the occupied cells.

The anchor may be above the visible board; its `y` can be `height` or greater. The cells may also be above the visible board.

`current.rotation` is the rotation state index. The platform page defines the states and the kicks between them.

## Hold

`hold` is the piece in hold, or `null` when hold is empty. `canHold` is false when the client does not allow a hold this turn. The bot sees the held piece either way, but it must not set `hold` to true in the reply when `canHold` is false.

## Neutral values

`queue` lists the upcoming piece types, nearest first. The current piece is not in the queue. An empty array means the client exposes no preview, or there are no more pieces. The bot must not assume that a next piece exists.

`attackMultiplier` is the multiplier applied to attack for this move. `0` means the round has no attack, or the bot should ignore attack. If the platform has attack, the client sends the current multiplier.

## Reply

The bot replies with one move message.

```json
{"command":"move","data":{"x":1,"y":0,"rotation":2,"hold":false,"spin":"full"}}
```

| Field | Meaning |
|------|---------|
| `x` | Smallest x among the placed piece's cells. |
| `y` | Smallest y among the placed piece's cells. |
| `rotation` | Final rotation state. |
| `hold` | Whether to swap the current piece with hold before placing. |
| `spin` | Spin classification for the placement under the round's rules. |

Every reply field is required, including `hold` when it is false and `spin` when it is `none`.

The reply uses the same anchor as the request. The client derives the placed cells from the piece being placed, `rotation`, and the shape.

There is no piece field. When `hold` is false, the placed piece is the current piece. When `hold` is true, it is the held piece, or the first queued piece when hold is empty. The client rejects `hold: true` when `canHold` is false or there is no piece to place.

`spin` is one of:

| Value | Meaning |
|------|---------|
| `none` | Not a spin. |
| `mini` | A mini spin. |
| `full` | A full spin. On a platform with no mini split, every spin is full. |

The reply chooses a placement, not a path. After applying any hold, the client finds a legal way to reach it with the placed piece. If `spin` is `mini` or `full`, the path must end with a rotation and classify as that spin. If `spin` is `none`, the path must not be a spin. The reply does not name the last rotation, so the client tries the rotations the round supports.

If no such path exists, the client rejects the move. The bot must only return placements that are reachable with the stated spin.

The client then locks the piece and clears lines.
