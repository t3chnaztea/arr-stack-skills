---
name: arr-context-map
description: >-
  Use when an agent needs to know how THIS media stack is put together
  before it can safely act: which quality profile id means what, which root
  folder is for which kind of content, how container paths map to host
  paths, which download client handles which protocol, and which app is
  managed by a template sync. Triggers include "map my setup", "what
  profiles do I have", "build an inventory", "why did the agent pick the
  wrong profile", "document my arr stack", or starting work on an
  unfamiliar instance. Also covers keeping the map honest as the stack
  drifts. Assumes arr-connect for the API calls. Not for designing the
  profiles themselves (arr-quality-profiles).
compatibility: Radarr v4/v5, Sonarr v3/v4, and companions. The map is a plain Markdown file you own; nothing here writes to your apps.
---

# Arr Context Map

Every other skill in this collection describes behaviour that is true
everywhere. This one is about the facts that are true only on your instance,
and it exists because those are the facts an agent will otherwise invent.

The failure this prevents is specific: an agent reads a guide that says "use
the Balanced profile", finds no such name, picks the closest-sounding one, and
quietly puts a 4K film on a profile meant for a phone. Profile ids and names
are per-install. So are root folders, path mappings, and which app is
template-managed.

**This repo deliberately ships no inventory.** A complete map of your media
stack is a useful document and a bad thing to publish. You build yours, you
keep it local.

## What the map is

One Markdown file, checked in next to your own automation or kept beside your
agent's working directory. Not a database, not generated on every run. Written
once, corrected when it turns out to be wrong.

Suggested location: `docs/arr-map.md` in whatever repo holds your homelab
config, or wherever your agent already looks for project context.

## Section 1: instances

For each app: what it is, where it lives, and how the agent authenticates.

```markdown
## Instances

| App | URL | Key env var | Version verified |
|---|---|---|---|
| Radarr | http://arr-host:7878 | RADARR_KEY | 5.x, 2026-07 |
| Sonarr | http://arr-host:8989 | SONARR_KEY | 4.x, 2026-07 |
| Prowlarr | http://arr-host:9696 | PROWLARR_KEY | 1.x, 2026-07 |
```

Record the **env var name**, never the key. Note the version and the date you
checked, because half the traps in these skills are version-scoped and a map
that does not say when it was true is a map you cannot trust later.

## Section 2: quality profiles, with intent

Generate the raw list, then add the column the API cannot give you: what the
profile is *for*.

```bash
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/qualityprofile" \
  | jq -r '.[] | "\(.id)\t\(.name)\tcutoffFormatScore=\(.cutoffFormatScore)"'
```

```markdown
## Radarr profiles

| id | Name | Use it for | Notes |
|---|---|---|---|
| 12 | Standard HD | default for everything | cutoff 1080p, cutoffFormatScore 400 |
| 14 | Archive 4K | films worth the disk | 4K only, not the default |
| 17 | Anime | anime root folder only | format ladder runs on a 1000s scale |
```

The "use it for" column is the entire point. Without it an agent picks by
name, and profile names are marketing rather than instruction.

Note the `cutoffFormatScore` per profile. It is the value most likely to be
wrong (see `arr-quality-profiles`) and having it in the map means an upgrade
storm gets spotted from the document rather than from your bandwidth graph.

## Section 3: root folders, and which profile goes with each

Root folder and profile are correlated in almost every real setup, and nothing
in the API expresses the correlation. Write it down.

```bash
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/rootfolder" \
  | jq -r '.[] | "\(.id)\t\(.path)\t\(.freeSpace)"'
```

```markdown
## Root folders (Radarr)

| Path (as the app sees it) | Default profile | Content |
|---|---|---|
| /mnt/Movies | 12 Standard HD | general library |
| /mnt/Anime/Movies | 17 Anime | anime films |
| /mnt/Documentaries | 12 Standard HD | documentaries |
```

State the default-profile-per-root-folder rule explicitly, because **the apps
cannot enforce it for you.** There is no supported per-root-folder default
profile; attempts to set one through the root folder endpoint are rejected.
The rule lives in this document and in the operator's head, which is exactly
why an agent needs to be able to read it.

## Section 4: path mappings

The trap from `arr-connect`, written down once so nobody re-derives it under
time pressure.

```bash
docker inspect <arr-container> \
  --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}'
```

```markdown
## Path mappings

| Host | Radarr sees | Sonarr sees | Download client sees |
|---|---|---|---|
| /srv/media | /mnt | /mnt | (not mounted) |
| /srv/downloads | /downloads | /downloads | /data/downloads |

Host `ls` on an app-reported path will report "No such file or directory".
That is the wrong path, not missing media. Authoritative answer for
"does this have a file" is the arr API (`hasFile`, `sizeOnDisk`).
```

Include the download client's view. Half of all import failures are the two
views of the completed directory not lining up.

## Section 5: who owns the configuration

The question that decides whether an edit survives the night.

```markdown
## Configuration ownership

- Radarr custom format scores: managed by Recyclarr (`recyclarr.yml`,
  nightly at 04:00). Manual API/UI score edits ARE REVERTED. Change the
  config, then sync.
- Sonarr: NOT in recyclarr.yml (no `sonarr:` block). Hand-maintained.
  API/UI edits are durable.
- Profile structure (ladders, cutoffs, cutoffFormatScore) is hand-managed
  on both. A score sync does not revert it.
```

Verify rather than assume:

```bash
grep -nE '^(radarr|sonarr|lidarr|readarr):' /path/to/recyclarr.yml
```

Getting this backwards in either direction wastes real time: either you edit
in the wrong place and it reverts, or you rewrite a config file for an app
nobody is syncing.

## Section 6: download clients and protocols

```markdown
## Download clients

| Client | Protocol | Behind VPN | Notes |
|---|---|---|---|
| qbittorrent | torrent | yes, sidecar netns | port forwarding via hook script |
| sabnzbd | usenet | no | large 4K posts fail par2 often |
```

The "behind VPN" column matters because it changes the diagnostic path in
`arr-downloads` and because restarting a VPN-wrapped client is not a free
action.

## Section 7: an append-only gotchas log

The highest-value section and the one people skip. Every time the stack
surprises you, add a dated line. Never delete entries, only mark them
superseded.

```markdown
## Gotchas

- 2026-07-18: Requests failing as "declined" after profile renumbering.
  Request front-end held a deleted profile id. Fixed in its server settings,
  then retried the failed requests.
- 2026-06-28: Anime grabs silently blocked by a global release profile
  banning x265, contradicting the anime profile's +x265 score. Removed the
  global term, block per-profile instead.
```

Six months later this is the most useful part of the file, and it is the part
an agent can act on without asking you anything.

## Keeping it honest

A wrong map is worse than no map, because it is confidently wrong. Two habits
keep it usable:

**Date every claim.** Anything version-scoped gets the version and the date it
was verified. An entry with no date is a rumour.

**Re-derive before acting on anything destructive.** A map is a strong prior,
not a source of truth. Before a delete, a bulk profile change, or anything
touching files, re-query the live API for the specific ids involved. Profile
ids change when profiles are recreated. Root folders get added. Containers get
recreated with different mounts.

A quick drift check worth running when work on this stack resumes after a gap:

```bash
# do the profile ids in the map still resolve to the same names?
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/qualityprofile" \
  | jq -r '.[] | "\(.id)\t\(.name)"'
```

Diff that against your map's table. If an id moved, stop and fix the map
before doing anything else, and check the request front-end while you are
there.

## A note on what not to write down

Keep the map local and keep it boring. It does not need your API keys, your
external hostname, your indexer credentials, or your VPN account. Env var
names, internal URLs, ids, and paths are enough for an agent to work, and none
of it is interesting if the file leaks. Reference secrets by the name of the
variable that holds them and let the environment do its job.
