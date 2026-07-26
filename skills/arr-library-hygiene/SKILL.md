---
name: arr-library-hygiene
description: >-
  Use when the library and the app disagree about what should exist:
  episodes stuck permanently "missing", a season pack re-grabbed every
  night, "episode file already imported" looping in the queue, a search
  that returns queued but finds nothing, seasons that re-monitor
  themselves after a metadata refresh, deleting a file so a better one can
  be grabbed, downsizing an oversized file, or clearing a stuck queue.
  Also covers what an automated queue cleaner does and does not need.
  Assumes arr-connect for the API pattern and its warning that deletes are
  permanent. Not for choosing what quality to grab (arr-quality-profiles)
  or indexer and download client faults (arr-downloads).
compatibility: Sonarr v3/v4 for the per-episode monitor endpoints; Radarr v4/v5 for the movie-file equivalents. Queue-cleaner notes reflect Cleanuparr-style tools that drive the arr API.
---

# Arr Library Hygiene

The `monitored` flag drives every automated search. When it drifts from what
you intended, the app either hunts for things it should ignore or sits silent
when you expect it to work. Both look like bugs elsewhere.

Assumes `arr-connect`. In particular: deletes are permanent, there is no
recycle bin unless you configured one, and a command returning `queued` means
accepted, not successful.

## Three ways monitoring drifts

### 1. Segment-aired shows create permanent phantom missing episodes

Cartoons and sketch shows that aired as segments are often catalogued with
each segment as its own episode entry, while your files are whole broadcast
episodes. The app counts the extra segment entries as missing forever. Every
night it re-searches, grabs a season pack, fails to import with *episode file
already imported*, and refills the queue. The queue length looks alarming and
never drops.

Diagnose by comparing counts, not by reading the queue:

```bash
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/series/$SID?includeSeriesStats=true" \
  | jq '{title, episodeCount: .statistics.episodeCount,
         episodeFileCount: .statistics.episodeFileCount,
         pct: (.statistics.episodeFileCount / .statistics.episodeCount * 100 | floor)}'
```

A gap that never closes across days, on a show that finished airing years ago,
is this pattern rather than a sourcing problem.

Fix: unmonitor only the monitored-but-file-less episodes, then clear the stuck
queue.

```bash
# the episodes that will never arrive
IDS=$(curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/episode?seriesId=$SID" \
  | jq -c '[.[] | select(.monitored and (.hasFile | not)) | .id]')

curl -sf -X PUT -H "X-Api-Key: $ARR_KEY" -H 'Content-Type: application/json' \
  -d "{\"episodeIds\": $IDS, \"monitored\": false}" \
  "$ARR_URL/api/v3/episode/monitor"
```

For a finished show this stops the grabbing and keeps every existing file. For
a still-airing show, do not do this: you would unmonitor genuinely missing
episodes. Check the series status first.

### 2. Deleting episode files auto-unmonitors them

`DELETE /api/v3/episodefile/{id}` flips those episodes to `monitored: false`.
It does this per episode, so the season and series flags still read as
monitored and everything looks fine in the UI. A subsequent search matches
zero episodes and reports success.

This is the "I deleted the bad files so it would grab better ones, and nothing
happened" case. The delete worked. The re-arm step is missing.

Correct sequence when pruning files for a re-grab:

```bash
# 1. delete the files
curl -sf -X DELETE -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/episodefile/$FID"

# 2. RE-ARM. this is the step everyone skips
curl -sf -X PUT -H "X-Api-Key: $ARR_KEY" -H 'Content-Type: application/json' \
  -d "{\"episodeIds\": [$EID], \"monitored\": true}" \
  "$ARR_URL/api/v3/episode/monitor"

# 3. now search
curl -sf -X POST -H "X-Api-Key: $ARR_KEY" -H 'Content-Type: application/json' \
  -d "{\"name\": \"MissingEpisodeSearch\", \"seriesId\": $SID}" \
  "$ARR_URL/api/v3/command"
```

Diagnostic when a search returns `queued` and the indexers stay quiet: count
how many episodes are actually monitored. Zero means step 2 was skipped.

```bash
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/episode?seriesId=$SID" \
  | jq '[.[] | select(.monitored)] | length'
```

### 3. A metadata refresh can re-monitor seasons you turned off

The daily series refresh re-syncs from the metadata provider. When the
provider restructures a series or adds episodes, seasons you deliberately
unmonitored have been observed flipping back to monitored, sometimes within a
day. If an automated queue cleaner is running, this produces a delete-and-
re-search loop that repeats on the cleaner's interval.

Practical consequences:

- **Never trust a previous session's outcome.** When asked "did that stay
  unmonitored", re-query and inspect `seasons[].monitored` rather than citing
  what you set last week.
- For a decision you want to be permanent, prefer a stronger mechanism than a
  per-season toggle: remove the series, or set the series-level monitor option
  so new content is not picked up at all.

```bash
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/series/$SID" \
  | jq '{title, monitored, monitorNewItems, seasons: [.seasons[] | {seasonNumber, monitored}]}'
```

## Replacing a file with a better one

An arr will not downgrade in place, and will not re-grab something it already
considers satisfied. To deliberately replace a file with a *different* one
(usually smaller, occasionally a different edition), the existing file has to
go first.

This is destructive and it is where agents cause real damage. The guarded
sequence:

1. **Confirm a satisfactory replacement actually exists** before deleting
   anything. Run an interactive search and read the results.

   ```bash
   curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/release?movieId=$ID" \
     | jq '.[] | {title, quality: .quality.quality.name, size, customFormatScore,
                  rejected, rejections}'
   ```

2. **Check why candidates were rejected.** A release rejected *only* by
   upgrade protection will become grabbable once the file is gone. A release
   rejected for size, format score, or an indexer rule will still be rejected
   afterwards, and deleting leaves you with a gap.

3. **Set a profile that excludes the current quality**, otherwise the app may
   simply re-grab what you just deleted.

4. Delete the file, re-arm monitoring, then **grab the specific release by
   its guid** rather than firing a blind search. A blind search can pull a
   different cut, an extended edition, or a worse encode.

5. **Verify by polling the queue about 20 seconds later.** Checking
   immediately false-negatives because the grab has not registered yet.

If step 1 or 2 fails, skip the title. Leaving a large file in place is always
better than creating a hole in the library. Say which titles you skipped and
why.

For bulk passes, write the target list to a file, print it, and have a human
read it before anything is deleted. Mass irreversible media deletion is not an
operation to run on inference.

Edition traps worth knowing before a downsize pass: for some catalogue titles
the only 4K release is an extended cut that is *larger* than the standard
remux you were trying to shrink, so the swap is a net loss. Compare actual
sizes, not quality labels.

## Clearing a stuck queue

```bash
# what is actually in there and why
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/queue?pageSize=100" \
  | jq '.records[] | {title, status, trackedDownloadStatus, trackedDownloadState,
                      errorMessage, statusMessages}'
```

`trackedDownloadState: importBlocked` or `importFailed` is the app telling you
the download finished but could not be moved into the library. Common causes:
a permissions mismatch between the download directory and the library, a path
mapping the app cannot resolve, or the phantom-episode loop above.

To flush stuck items, remove them from the client and blocklist the releases
so the same bad grab is not immediately retried:

```bash
curl -sf -X DELETE -H "X-Api-Key: $ARR_KEY" \
  "$ARR_URL/api/v3/queue/bulk?removeFromClient=true&blocklist=true&skipRedownload=true"
```

Drop `skipRedownload` if you do want it to hunt a replacement. Keep it when
you are clearing a loop, or you restart the loop.

## What an automated queue cleaner actually needs

If you run a queue-cleaning companion, understand what drives it: **the arr
API, not the download client.** Failed-import detection reads the arr queue
for import-blocked states, strikes items over repeated passes, and on the
limit calls the arr's own queue delete with removal and blocklisting. The arr
then handles removal from whichever client held it.

Two consequences that save wasted effort:

- Such tools commonly support only torrent clients in their *client* config.
  That does not mean Usenet downloads go unhandled. Usenet items bypass the
  client-matching branch and are handled through the arr API path anyway. Do
  not go looking for a Usenet download-client integration that does not exist
  and is not needed.
- A per-app strike setting of `-1` usually means "inherit the global", not
  "disabled". Read the tool's own resolution logic before concluding a feature
  is off.

More blocklisting is not the fix for content that has genuinely aged out of
your sources. That is a source diversity problem: add indexers.

## Media server side effects

If a media server indexes your *download* directory as a library location
(the watch-while-downloading setup), in-flight downloads get indexed as extra
versions of the same title. When the arr imports and the download directory
disappears, a client can still select the now-dead version and fail playback
with a not-found error on the file part.

Resolve that with a library rescan and empty-trash, or by simply waiting for
the next scan. Do **not** resolve it by deleting the media entry through the
media server's API: on at least one major media server that call deletes the
underlying file from disk, and it re-points parts to renamed or moved files,
so an entry that looked orphaned a minute ago may now point at a live import.
That combination has destroyed freshly imported files.

Related: if you inspect a media server's SQLite database directly, copy the
write-ahead log alongside it or you will read stale rows and act on them.
Querying the API is safer than copying a live database.
