---
title: Game init
---

# Game init

A bot process handles one round. The client starts the process and sends `init` once, before the first move. The message carries the rules that stay the same for the round.

`init` does not carry the board. Board width, height, and cells can change during a round, so the board is state and travels with every [move request](<move>).

Messages are one JSON object per line. See [transport](<transport>) for framing, flushing, and process lifetime.

## Glossary

| Term | Meaning |
|------|---------|
| Platform | A named ruleset the bot supports, such as `ppt`. |
| Profile | The values that come with a platform, such as its rotation system and piece shapes. |
| Round | One game, from the first piece to the end. |
| Rotation system | The named set of piece rotations and kicks. |
| Lockout | A piece that locks with every cell above the visible board. |
| Rules | Platform-specific settings for the round. |

## Message

```json
{"command":"init","data":{"platform":"ppt","rotation":"SRS","mechanics":{"lockout":true},"rules":{}}}
```

Every field is required. A value that does not apply still has an explicit value. `rules` is an empty object when the platform has no overrides.

A second `init` in the same process is an error. The rules do not change during a round.

## Fields

| Field | Meaning |
|------|---------|
| `platform` | Profile id. The profile defines the platform's rules. |
| `rotation` | Rotation system for the round. |
| `mechanics.lockout` | Whether a piece that locks entirely above the visible board ends the round. |
| `rules` | Platform-specific settings. The platform page defines the keys. An empty object means no overrides. |

`platform` selects the profile. `rotation` and `mechanics.lockout` are always sent, even when the profile already fixes them. If a sent value disagrees with the profile, the bot rejects the message. This keeps the bot from guessing which values came from the client and which came from the profile.

`rules` is the one place where an absent key means "use the profile value." The generic fields do not depend on it; the platform-specific rules do. This keeps platform defaults out of the generic message.

## Lockout

A piece is locked out when all of its cells are above the visible board. A piece that is partly visible is not locked out.

`mechanics.lockout` is `true` when a lockout ends the round, and `false` when the round continues. Not every platform uses lockout, but every client can answer the question.

## What init does not carry

The board is not a rule. Its width, height, and rows are sent with every move request. This matters for modes that join boards during a round, where the width can grow.

The current piece, hold, queue, and attack multiplier are also live values. They belong to the move request. Platform-specific live values, such as gravity or incoming garbage, belong to the platform page.

## Reply

The bot replies with one of these:

```json
{"command":"ready","data":{}}
```

```json
{"command":"error","data":{"message":"unsupported platform"}}
```

The bot rejects an unknown `platform`, an unknown `rotation`, an unknown `rules` key, and a value that disagrees with the profile. It does not silently ignore a setting.

## Platforms

The first profile ids are:

| `platform` | Game |
|------|------|
| `tetrio` | TETR.IO |
| `ppt` | Puyo Puyo Tetris |
| `toj` | TOJ, including MisaMino bots that follow TOJ rules |

Each profile gets its own page for its fixed values and the keys it accepts in `rules`.
