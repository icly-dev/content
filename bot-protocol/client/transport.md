---
title: Transport
---

# Transport

The client starts the bot as a child process. The client writes to the bot's standard input, and the bot writes back on standard output. Both streams carry one JSON object per line, encoded as UTF-8.

## Glossary

| Term | Meaning |
|------|---------|
| Client | The process that controls the game and starts the bot. |
| Bot | The process that receives the game state and returns moves. |
| Message | One JSON object sent as one line. |

## Lifecycle

The client starts the bot before the round. The bot reads messages until the client closes its standard input, then exits. There is no separate end message.

## Framing

Every message is one JSON object followed by a newline. On the wire, a message is one line with no newline inside it. If an example is wrapped for display, the message is still one line.

A reader takes one line, parses it as JSON, and handles it. A pretty-printed object is not a valid message because the reader would see its first line as an incomplete object.

## Flushing

The writer must flush after every newline. A pipe does not flush on its own. If the bot buffers its reply, the client waits even though the bot has finished the work.

A missing newline fails the same way: the reader keeps waiting for the rest of the line.

## Logs

The bot writes logs to standard error, never to standard output. Standard output carries only protocol messages. A log line on standard output makes the next message unreadable.

## Ordering

The client sends one request and waits for the reply before sending the next. One request at a time keeps each reply matched to its request and keeps the pipes from filling.

## Latency

The streams are local pipes, not a network. A board message is a few kilobytes, so the transport cost is small compared with the bot's move choice. The usual causes of delay are an unflushed write, a missing newline, or a bot that is still computing.
