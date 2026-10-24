---
title: Remote helpers
---

# Remote helpers

A tuning or comparison run can use the idle machines around it. The machine running the run is the host: it holds the configuration, builds the battle schedule, and collects the results. Other machines run as helpers, connect to the host, and play the battles it hands them. Each helper uses its own engine, so a run scales with the cores on the network instead of the cores in one machine.

## Glossary

| Term | Meaning |
|------|---------|
| Host | The machine that starts the run and owns the schedule, the scores, and the output files. |
| Helper | A machine that connects to a host and plays the battles it is given. |
| Engine fingerprint | A value derived from a fixed self-play battle, used to confirm that both sides run the same engine. |
| Requeue | Returning a helper's unfinished battles to the host's queue when the helper disconnects. |

## The handshake

A helper opens a connection and identifies itself with a protocol version and an engine fingerprint. The fingerprint is derived from a short, deterministic self-play battle, so a different engine build, ruleset, or evaluation changes it. A mismatch is refused: a battle played on a different engine would not measure the same thing as the host's own battles.

The protocol assumes a trusted network such as a home LAN or a private tailnet. There is no encryption and no authentication, so a helper should only be offered to a host you would trust with your machine time.

## How work moves

The host schedules the run as usual and hands out battles from that schedule. A helper asks for work, plays the battles it receives with its local engine, and reports the scores back. The host counts a battle as running the moment it is handed out, so no battle goes to two helpers. A battle depends only on the theta pair, its seed pair, and the options, so a run that uses helpers produces the same results as one that does not.

A helper that loses the host, or reaches the end of a run, reports how many battles it still had in flight and goes back to waiting. The next run on that host is joined without restarting the helper. The host requeues the battles that were still in flight, so a disconnect never leaves a hole in the schedule. Both sides discard work for a run that has already ended.

Helpers do not need to be told the host's address: a helper can look for a host on its local network and connect when it finds exactly one.

## What the run shows

The host and each helper report the connection state, how many battles the helper has finished and has in flight, and how many battles had to be requeued. The host's progress view also distinguishes its own battles from the ones played remotely, since only its own battles consume its cores.

## Limits

Helpers run the same ruleset, platform, and engine build as the host, which the handshake enforces. Results stay valid because the schedule is fixed, but a helper that disconnects mid-battle costs that battle's time: the host schedules it again. A slow helper only holds up its own battles, though the run can end with a few stragglers still running.
