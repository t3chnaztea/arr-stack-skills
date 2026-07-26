---
name: arr-connect
description: >-
  Use when pointing a coding agent at Radarr, Sonarr, Prowlarr, or a *arr
  companion for the first time or in a fresh session: "connect to Radarr",
  "talk to Sonarr's API", find the API key, hit /api/v3, check whether a
  movie or show is already in the library, confirm a title has a file on
  disk, work out why an API write reported success but changed nothing, or
  why `ls` on the server says the media folder does not exist. Covers the
  API-key lanes, the v3 endpoint map, the container-path trap, the
  read-after-write rule, and identifier-based existence checks. Foundation
  skill: every other arr-* skill assumes its connection pattern and doctrine.
  Not for quality profile design (arr-quality-profiles) or building the
  instance inventory (arr-context-map).
compatibility: Radarr v4/v5 and Sonarr v3/v4 (API v3 on both). Prowlarr, Lidarr, and Readarr expose the same shape with different resource names. Recipes are plain curl.
---

# Arr Connect

How to reach a *arr instance, what its API will lie to you about, and the two
checks that stop most agent damage before it happens. Every other `arr-*`
skill assumes this.

## The one-minute setup

Every arr app uses a single API key, no OAuth, no session. Find it in
**Settings > General > Security**, or read it out of `config.xml` in the app's
config directory:

```bash
grep -o '<ApiKey>[^<]*</ApiKey>' /path/to/config/config.xml
```

Keep it in the environment, never inline:

```bash
export ARR_URL=http://arr-host:7878        # Radarr 7878, Sonarr 8989, Prowlarr 9696
export ARR_KEY=...                          # from config.xml or the UI
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/system/status" | jq '{appName,version}'
```

If that returns app name and version, you are connected. A 401 means the key
is wrong; a connection refused means the port or host is wrong; an HTML
response means you hit a reverse proxy that is not forwarding `/api`.

The key can also travel as `?apikey=` in the query string. Prefer the header:
query strings land in access logs and shell history.

## The endpoint map

Both apps expose API v3 regardless of the app's own version number. Sonarr v4
is still `/api/v3`. This surprises people; do not go hunting for `/api/v4`.

Shared shape across apps:

| Purpose | Radarr | Sonarr |
|---|---|---|
| Library list | `GET /api/v3/movie` | `GET /api/v3/series` |
| One item | `GET /api/v3/movie/{id}` | `GET /api/v3/series/{id}` |
| Add | `POST /api/v3/movie` | `POST /api/v3/series` |
| Search metadata | `GET /api/v3/movie/lookup?term=` | `GET /api/v3/series/lookup?term=` |
| Files | `GET /api/v3/moviefile?movieId=` | `GET /api/v3/episodefile?seriesId=` |
| Quality profiles | `GET /api/v3/qualityprofile` | same |
| Root folders | `GET /api/v3/rootfolder` | same |
| Queue | `GET /api/v3/queue` | same |
| Trigger work | `POST /api/v3/command` | same |
| History | `GET /api/v3/history` | same |
| Health | `GET /api/v3/health` | same |

Sonarr adds `GET /api/v3/episode?seriesId=` and the per-episode monitor
endpoint, which have no Radarr equivalent because a movie is one item.

`POST /api/v3/command` is how you make the app *do* something rather than
just record something. The body is `{"name": "<CommandName>"}` plus command
arguments. `RefreshMovie`, `MoviesSearch`, `RefreshSeries`, `SeriesSearch`,
`MissingEpisodeSearch`, `RescanFolders`. The response is a command record with
an id; poll `GET /api/v3/command/{id}` for `status` if you need to know it
finished. A command returning `queued` means it was accepted, not that it
found anything.

## Trap: root folders are container paths

This one has cost real data.

An arr running in a container reports its root folders as absolute paths
(`/movies`, `/tv`, `/mnt/Movies`). Those are paths **inside the container**.
The host almost never has the same tree, because the container was started
with a bind mount that rewrites the prefix.

So this happens: you ask "is this movie actually on disk", you SSH to the
host, you run `ls /mnt/Movies/Some Movie (2011)`, and the shell says **No such
file or directory**. That reads as "the media is gone". It is not. It is the
wrong path.

Resolve the mapping before trusting any host-side file check:

```bash
docker inspect <arr-container> \
  --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}'
```

Now you know that host `/srv/media` presents as `/mnt` in the container, and
the real location is `/srv/media/Movies/Some Movie (2011)`.

Better still: **do not ask the filesystem.** For "does this have a file", the
authoritative source is the arr's own API:

```bash
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/movie/$ID" \
  | jq '{hasFile, sizeOnDisk, path, relativePath: .movieFile.relativePath}'
```

Use host paths only when you need to inspect actual bytes (ffprobe, checksum),
and derive them from `docker inspect` each time rather than remembering them.

## Trap: existence checks by title produce false negatives

"Do we already have Movie X" answered by matching titles against a library
listing will tell you no when the answer is yes. Library titles carry
year suffixes, edition tags, hand edits, and dropped punctuation. `Terminator
2 Judgment Day` does not string-match `Terminator 2: Judgment Day`.

Check by **identifier**, which is what the app actually keys on:

```bash
# Radarr, by TMDB id
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/movie?tmdbId=550" | jq '.[0] | {id,title,hasFile}'

# Sonarr, by TVDB id
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/series?tvdbId=121361" | jq '.[0] | {id,title,statistics}'
```

An empty array is a real "not in the library". Adding a duplicate is also a
free existence check: the add endpoint rejects it with a validator error
naming the existing item rather than creating a second copy.

If you must match against a title (because you are cross-referencing a media
server rather than the arr), match on year plus a fuzzy or substring title,
never an exact normalized string.

## Doctrine: read after every write

**Arr PUTs silently no-op.** A malformed or partial update object frequently
returns `200 OK` with a body that echoes what you sent, while the app stored
something else or nothing at all. The response body is not evidence.

The rule for every write:

1. `GET` the resource. Keep the whole object.
2. Modify the fields you intend to change **on that object**.
3. `PUT` the whole object back, not a fragment.
4. `GET` it again, fresh, and diff the field you changed.
5. Report the before and after values.

```bash
# read
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/movie/$ID" > /tmp/movie.json
jq '.qualityProfileId' /tmp/movie.json          # before

# modify the full object and write it back
jq '.qualityProfileId = 7' /tmp/movie.json \
  | curl -sf -X PUT -H "X-Api-Key: $ARR_KEY" -H 'Content-Type: application/json' \
      --data-binary @- "$ARR_URL/api/v3/movie/$ID" > /dev/null

# verify from a fresh read, not from the PUT's echo
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/movie/$ID" | jq '.qualityProfileId'
```

Partial-object PUTs are the single most common way an agent "successfully"
wipes a setting it never intended to touch. Send the object you read.

## Doctrine: deletes are permanent unless you configured otherwise

Neither Radarr nor Sonarr ships with a recycle bin enabled. `deleteFiles=true`
on a delete endpoint erases from disk immediately. There is no undo, no trash
folder, and no confirmation beyond the query parameter you typed.

Before any delete that touches files:

- Confirm the item id resolves to the title you think it does. Print the title.
- Check `hasFile` and `sizeOnDisk` so you know what you are destroying.
- Consider setting a recycle bin path in **Settings > Media Management** first.
- For anything bulk, write the list of targets out and have a human read it
  before the destructive pass runs.

Deleting is the one operation where "the agent was confident" is not good
enough. See `arr-library-hygiene` for the delete-then-regrab pattern, which
exists precisely because deletes cannot be walked back.

## When metadata lookups hang

Symptom: `GET /api/v3/series/lookup?term=x` returns a JSON body containing
`"Http request timed out"`, and commands like `RefreshSeries` sit at
`status: started` forever with no log activity. The library and API work fine
otherwise.

Cause, in most containerized Linux setups: the app's .NET HTTP client prefers
AAAA records and stalls until timeout when the IPv6 path does not actually
route. It hits any app that calls a dual-stack metadata provider. Apps that
reach a v4-happy provider on a different code path stay unaffected, which is
why one arr can break while its sibling is fine.

Diagnose in this order:

```bash
# 1. is the provider itself up? test from the host, not the container
curl -so /dev/null -w '%{http_code} %{time_total}s\n' https://<metadata-provider>/

# 2. does the container hold global v6 addresses with a v6 default route?
docker exec <arr-container> ip -6 addr show scope global | wc -l
```

The durable fix is an environment variable on the container that tells .NET
to stop trying v6, rather than disabling IPv6 across the host:

```
DOTNET_SYSTEM_NET_DISABLEIPV6=1
```

Set it in the container's environment and recreate the container. It survives
image upgrades, which host-level sysctl tuning often does not: container
platforms regenerate their network config on update and can hand your bridges
a fresh ULA pool, bringing the stall right back. If a sibling .NET arr app
starts stalling later, apply the same variable there.

Do not confuse this with a genuine provider outage. Test the provider from the
host first; if it answers in under two seconds and the container still hangs,
it is the v6 stall.

## What to check when something is wrong

```bash
# health warnings the app has already worked out for you
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/health" | jq '.[] | {source,type,message}'

# is the download client reachable and are indexers answering
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/downloadclient" | jq '.[] | {name,enable}'
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/indexer" | jq '.[] | {name,enable}'

# what the app actually did recently
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/history?pageSize=20" \
  | jq '.records[] | {date,eventType,sourceTitle}'
```

`/api/v3/health` is underused. It surfaces broken root folders, unreachable
clients, and indexer failures that would otherwise present as "searches find
nothing", and it costs one call.

## Where to go next

- `arr-context-map`: build the instance inventory (profile ids, root folders,
  path mappings) that every other skill needs and none of them can guess.
- `arr-quality-profiles`: what the app grabs and why it will not stop upgrading.
- `arr-library-hygiene`: monitoring drift, deletes, re-grabs, queue cleanup.
- `arr-downloads`: indexers, clients, and the request front-end.
- `arr-playback`: why the file you grabbed makes your player transcode.
