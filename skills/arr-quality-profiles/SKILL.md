---
name: arr-quality-profiles
description: >-
  Use when working on what Radarr or Sonarr is allowed to grab and when it
  stops: quality profiles, custom formats, cutoffs, upgrade behaviour, or
  Recyclarr and TRaSH guide templates. Triggers include "it keeps
  re-downloading the same movie", "endless upgrades", "why won't it grab
  this release", "it grabbed a worse version", "set up a 4K profile",
  "my custom format scores keep reverting", "cutoff must be an allowed
  quality or group", "release rejected below profile minimum", or
  designing a profile for anime, vintage TV, or a specific streaming
  service. Assumes the connection pattern and read-after-write rule from
  arr-connect. Not for monitoring flags or deleting and re-grabbing files
  (arr-library-hygiene), and not for transcode-driven format choices
  (arr-playback).
compatibility: Radarr v4/v5 and Sonarr v3/v4. Recyclarr sections apply if you sync TRaSH guide templates; the traps apply either way.
---

# Arr Quality Profiles

A quality profile answers two questions: what may this app grab, and when may
it stop looking. Most "the arr is behaving insanely" reports are one of those
two answers being wrong in a way the UI does not make obvious.

Assumes `arr-connect` for the API pattern. Read the whole profile object,
modify, PUT it back, read it again.

## The model, briefly

A profile has an **allowed quality list** (an ordered ladder, with optional
groups), a **cutoff** (the rung at which "good enough" is reached), a set of
**custom format scores**, and a **cutoffFormatScore** (the format score at
which format-driven upgrading stops).

Two independent stop conditions, and both must be reachable:

- Quality cutoff reached, and
- Format score at or above `cutoffFormatScore`.

If either is unreachable by any release that actually exists, the app never
considers the item done and keeps searching forever.

```bash
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/qualityprofile" \
  | jq '.[] | {id, name, cutoff, cutoffFormatScore, upgradeAllowed,
               allowed: [.items[] | select(.allowed) | (.quality.name // .name)]}'
```

Run that first, every time. Profile ids are per-instance. Any document that
tells you "use profile 6" is describing someone else's install.

## Trap: cutoffFormatScore left at 10000 causes endless upgrade storms

This is the highest-value item in this skill.

Several popular profile templates ship `cutoffFormatScore: 10000` as a
placeholder meaning "effectively never stop upgrading on format". If no
release in existence can score 10000 under your custom format set, the app
treats every item as permanently cutoff-unmet. It re-searches nightly,
re-grabs marginally different releases, churns your download client, and
rewrites files that were already fine.

The symptom people report is "it keeps downloading the same movie over and
over" or "my seedbox is constantly busy and nothing changes".

Diagnose:

```bash
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/qualityprofile" \
  | jq '.[] | select(.cutoffFormatScore >= 1000) | {id,name,cutoffFormatScore}'
```

Fix by setting `cutoffFormatScore` to a value a satisfactory release can
actually reach. To pick it, look at what your existing good files score:

```bash
# Radarr: what did the current file score under this profile?
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/movie/$ID" \
  | jq '{title, score: .movieFile.customFormatScore, profile: .qualityProfileId}'
```

Set the cutoff a little below the score a release you would be happy with
achieves. Two things to watch:

- **The right number depends on your custom format ladder's scale.** A profile
  whose formats are scored in the tens and hundreds wants a cutoff in the low
  hundreds. A profile built on a ladder that scores in the thousands (anime
  sets commonly do) needs a proportionally higher cutoff, or you will stop
  web-to-Bluray upgrades that you wanted.
- **Verify per profile, not globally.** Mixed ladders in one instance are
  normal and a single number will not fit all of them.

After changing it, confirm the storm actually stopped: watch the queue and
history over 24 hours rather than declaring victory on the PUT response.

## Trap: a cutoff is not a cap

A 1080p cutoff does not prevent 2160p. It means "stop *actively searching for
better* once you have 1080p". If 2160p is still in the allowed list, a 2160p
release remains a legal upgrade and the app will take it.

If you want a genuine ceiling, remove the higher qualities from the allowed
list. This matters when disk, bandwidth, or a client that cannot play 4K is
the real constraint.

Corollary: **an arr will not downgrade in place.** Once a large file is
imported, upgrade protection ranks it above anything smaller, so switching the
item to a lower profile and searching grabs nothing. Downsizing requires
deleting the file first or re-encoding locally. That sequence, and its
guardrails, is in `arr-library-hygiene`.

## Trap: Sonarr cutoffs must be group ids

Sonarr groups qualities (`WEB 1080p` contains WEBDL-1080p and WEBRip-1080p).
If you set the cutoff to the quality id when that quality lives inside a
group, the write fails with:

```
Cutoff must be an allowed quality or group
```

Read the profile's own `items` array and use the id of the enclosing group,
not the inner quality:

```bash
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/qualityprofile/$PID" \
  | jq '.items[] | select(.allowed) | {id, name, quality: .quality.name,
        contains: [.items[]?.quality.name]}'
```

Groups have a `name` and a nested `items` array; bare qualities have
`quality.name` and no children. Radarr has the same grouping concept and the
same failure mode.

## Trap: quality-definition minimum sizes override the profile

There is a global size table (**Settings > Quality**) with min, preferred, and
max megabytes-per-minute per quality. It applies **regardless of profile**.

So a profile named "Any Quality" that allows everything will still reject a
small release, and the rejection reason in the log talks about size, not
profile. People spend an evening editing the profile that was never the
problem.

```bash
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/qualitydefinition" \
  | jq '.[] | {quality: .quality.name, minSize, preferredSize, maxSize}'
```

Before loosening it, note that the table is global: lowering a minimum to
rescue one vintage show also lets small files into everything else. Often the
right call is to accept the block and source a better release.

## Trap: a release profile can contradict your custom formats

Sonarr release profiles carry must-contain and must-not-contain term lists,
applied globally or by tag. A banned term there silently overrides a custom
format that *prefers* the same thing.

The classic version: a global release profile that bans `x265` or `HEVC` at
720p/1080p, sitting alongside custom formats that score x265 positively for
anime. Anime is largely HEVC-native, so every anime grab is blocked while the
scores insist x265 is preferred. Nothing in the UI shows the contradiction;
the log just says releases were rejected.

```bash
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/releaseprofile" \
  | jq '.[] | {id, name, tags, required, ignored}'
```

If you want a codec blocked for some profiles and preferred for others, do it
with **per-profile custom format scores**, not a global release profile. A
strongly negative score on the profiles that should refuse it, a positive
score on the profile that wants it. Scores are profile-scoped; release profile
terms are not.

## Trap: over-negative scores block the back catalogue

Scores around -10000 are conventionally "never grab this". Applied to formats
like *no release group parseable* or *older codec*, they quietly make
pre-streaming-era content ungrabbable, because a 1990s show's only surviving
releases have exactly those characteristics.

Symptom: modern shows work perfectly, vintage shows find nothing, and the
indexers clearly have results.

Two fixes, in order of preference:

1. Set the offending format to `0` on the profiles where it is
   counterproductive. Zero means neutral, not banned.
2. Keep a dedicated permissive profile for vintage or ended content: allow the
   older quality rungs (SD, DVD, HDTV), zero out the punitive format scores,
   set the cutoff at a rung that genuinely exists for that era.

Assign that profile to the shows that need it rather than loosening the
profiles your modern library depends on.

## Know who owns your scores before you edit them

If you sync TRaSH guide templates with Recyclarr (or similar), **manual custom
format score edits get reverted on the next sync**. An agent that "fixes" a
score through the API, verifies it, and reports success will be wrong by
morning.

Check what the sync config actually covers:

```bash
grep -nE '^(radarr|sonarr):' /path/to/recyclarr.yml
```

A config with only a `radarr:` block means Sonarr is hand-maintained. That
inverts the rule between the two apps in the same house: Sonarr edits through
the UI or API are durable, Radarr edits are temporary unless made in the
Recyclarr config.

Two practical consequences:

- Before editing a score, say which app is template-managed and edit in the
  right place. For a managed app, change `recyclarr.yml` and sync; for an
  unmanaged one, the API is fine.
- Recyclarr manages **custom format scores**, not profile structure. Quality
  ladders, cutoffs, and `cutoffFormatScore` set by hand are not reverted by a
  score sync. That is why a `cutoffFormatScore` fix sticks even on a managed
  instance, and it is worth stating explicitly so nobody "re-fixes" it.

Adopting a fresh template on an instance you have hand-tuned will overwrite
your ladder and can reset cutoffs to the placeholder discussed above. Diff
before you adopt.

## Renumbering profiles breaks everything downstream

Deleting and recreating profiles changes their ids. Anything holding an id
by value now points at nothing:

- Request front-ends store a default profile id per server. That is Seerr
  today (Overseerr and Jellyseerr merged into it in early 2026; an Overseerr
  install still running is archived software), or Ombi. When it goes stale, **every new request fails**, and the
  user-visible state often reads as "declined" rather than "failed". The
  server log carries the real error: adding the item failed because the
  quality profile does not exist.
- Automation scripts and saved API calls that hardcode an id.
- Import lists and per-item overrides.

After any profile add, delete, or renumber, walk the downstream consumers and
re-select valid profiles. For a request front-end, fix the server settings
first, then retry the failed requests: retrying re-sends them to the arr,
whereas re-requesting from scratch usually hits the duplicate guard.

## Designing a profile: the short version

1. Decide the ceiling. Put only qualities you would actually accept in the
   allowed list. This is your real cap.
2. Decide the cutoff rung. A quality that exists for this kind of content.
3. Set `cutoffFormatScore` to something a good release reaches. Never leave a
   placeholder.
4. Score formats to express preference, not prohibition, except for the few
   things you truly never want.
5. Check the global size table permits what you just allowed.
6. Assign it to one item, search, and read the log for what got rejected and
   why before you roll it out to the library.

Step 6 is the one people skip. The rejection reasons in the log are specific
and they will tell you which of steps 1 through 5 is wrong.
