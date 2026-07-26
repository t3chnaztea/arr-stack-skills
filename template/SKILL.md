---
name: arr-yourskill
description: >-
  Use when [the specific situations, symptoms, and phrases someone would
  actually type when they hit this problem]. Pack in the error strings and
  the wrong-sounding complaints ("it keeps re-downloading", "the search
  finds nothing"), because those are what an agent matches on. Describe
  ONLY when to use it, never summarize the workflow inside. Keep the whole
  frontmatter under 1024 characters. End by naming what this skill is NOT
  for, pointing at the sibling skill that covers it.
compatibility: State what the skill needs: which apps, which API version, whether it assumes a container deployment, a VPN sidecar, or a template sync.
---

# Arr Yourskill

One or two sentences: what this covers, and that it assumes the connection
pattern and read-after-write doctrine from `arr-connect`. Cross-reference
siblings by name (`arr-context-map`, `arr-quality-profiles`,
`arr-library-hygiene`, `arr-downloads`, `arr-playback`) instead of repeating
their content.

## Guidelines for a good arr skill

Delete this section in your real skill; it is guidance for authoring.

- **Name:** `arr-<area>`, lowercase and hyphens only. The directory name MUST
  equal the frontmatter `name`.
- **Description:** starts with "Use when…", lists concrete triggers, symptoms,
  and error strings, ends with a "not for X, use Y" pointer. No workflow
  summary: an agent will follow the description instead of reading the body.
- **Markdown only.** This collection ships doctrine and fenced curl recipes,
  not scripts. The agent writes whatever throwaway code a task needs, fresh,
  against the actual instance. Recipes stay honest where scripts rot.
- **Original prose only.** Do not paste from the app wikis or the TRaSH
  guides. Write the lesson those do not teach and link them as the canonical
  reference.
- **Parameterize everything instance-specific.** `$ARR_URL` and `$ARR_KEY`
  from the environment, never a literal key. `<arr-host>`, never a real IP or
  hostname. Profile ids shown as examples must be flagged as examples, because
  ids are per-install and an agent will otherwise use yours.
- **No inventory.** No real library contents, no root folder listings from
  your own box, no indexer names, no VPN provider account details. If an
  example needs a title, use a well-known one to illustrate a category, not a
  dump of what you own.
- **Verify against ground truth.** Every write recipe ends by reading the
  resource back from a fresh GET and showing before and after. Arr PUTs
  silently no-op; a skill that does not verify teaches a bad habit.
- **Say which version you checked and when.** Endpoints and behaviours move
  between releases. An undated claim cannot be maintained.
- Keep `SKILL.md` under ~500 lines; push heavy detail into `references/*.md`.

## Overview

What this is and the core principle, in one or two sentences.

## When to use

Symptoms and situations, as bullets. Then when NOT to use it.

## [Your sections]

Quick-reference recipes, one worked example, and the specific traps you
learned by getting it wrong. One excellent worked example beats five generic
ones. If a trap cost you an evening, say so and say what the symptom looked
like before you understood it: the symptom is what the next person searches
for.
