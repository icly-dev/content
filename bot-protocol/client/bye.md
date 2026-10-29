---
title: Bye
---

# Bye

A session ends with a bye. The client sends it as its last call, with the session id, and the bot uses that id to find the session and close it.

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
| Client to bot | bye | Session id |

```json
{"command": "bye", "data": "session id here"}
```

The `data` field holds the id the client sent in [hello](<hello>).

## What the bot does

- Reads the session id from `data`.
- Finds the session stored under that id.
- Ends the session.
