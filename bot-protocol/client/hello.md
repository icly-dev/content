---
title: Hello
---

# Hello

A session starts with a hello. The client sends it before any other call, and the bot keeps the session id it carries.

## Glossary

| Term | Meaning |
|------|---------|
| Session | One client's conversation with the bot, from hello to bye. |
| Session id | The identifier the client sends with every call in a session. |

## Transport

The client writes each message to the bot's standard input. The bot replies on its standard output. Messages are JSON objects, with `command` naming the message and `data` carrying its payload.

## The message

| Direction | Name | Carries |
|-----------|------|---------|
| Client to bot | hello | Session id |

```json
{"command": "hello", "data": "session id here"}
```

The client chooses the session id and puts it in `data`. Every later client call carries the same id, so the bot can match the call to the session.

## What the bot does

- Reads the session id from `data`.
- Keeps the id for the life of the session.
- Matches later client calls to the session by that id.

The session ends when the client sends [bye](<bye>).
