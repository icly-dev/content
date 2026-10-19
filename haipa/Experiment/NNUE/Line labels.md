---
title: Line labels
---

# Line labels

**Status: refuted**

Two arms labeled each board with a statistic along the teacher's full line: the mean, v5, and the minimum, v6, of the hand eval values seen along it.

Shared vocabulary in [Glossary](<../../Engine/Glossary>). This page adds:

| Term | Meaning |
|------|---------|
| Line label | A statistic of hand eval values along the teacher's line. |

## The hypothesis

A line statistic sees several placements ahead, so the net should learn consequences the single-board value cannot express, while staying bounded: a mean or minimum over boards cannot grow with horizon the way accumulated reward does.

## What happened

The minimum anticorrelated with the hand eval: boards along the teacher's line are often temporary sacrifices, so the minimum rewards lines that dive. The mean was safer but still lost heavily in duels.

## The verdict

**Refuted.** Statistics along a line still re-import horizon-dependent structure in a softer form. The terminal value of the line is a different quantity, bounded and without the dive incentive; [Blending](<Blending>) tested it and refuted it too.
