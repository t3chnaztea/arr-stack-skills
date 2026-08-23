---
name: arr-downloads
description: >-
  Use when nothing is arriving, or arriving too slowly: searches that find
  no results, releases grabbed but never downloaded, torrents stuck at zero
  peers or a few KB/s, Usenet posts that fail to repair, downloads that
  complete but never import, a VPN-wrapped client that stops working after
  a restart, or requests from Seerr (or its Overseerr/Jellyseerr
  predecessors) or Ombi that show up as declined or failed. Covers the search-to-import chain, indexer and
  download client checks, VPN port forwarding, and where to look when the
  fault is upstream of the arr entirely. Assumes arr-connect for the API
  pattern. Not for what quality to accept (arr-quality-profiles) or stuck
  monitoring flags (arr-library-hygiene).
compatibility: Radarr v4/v5, Sonarr v3/v4, Prowlarr. Client notes cover qBittorrent-style torrent clients and SABnzbd/NZBGet-style Usenet clients. VPN section assumes a sidecar VPN container sharing a network namespace.
---

# Arr Downloads

Nothing arrived. The chain from "user wants a thing" to "file is in the
library" has six links, and the useful skill is finding which one broke
instead of restarting everything.

```
request front-end -> arr decides -> indexer search -> release grabbed
   -> download client -> import into library
```

Walk it in order. Assumes `arr-connect`.

## Start with health and history

Two calls answer most of these before you form a theory.

```bash
# the app has usually already diagnosed it
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/health" \
  | jq '.[] | {source, type, message}'

# what actually happened recently, per title
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/history?pageSize=30" \
  | jq '.records[] | {date, eventType, sourceTitle, data: (.data.reason // .data.message // empty)}'
```

`eventType` tells you exactly how far down the chain a title got:
`grabbed` then nothing means the client never took it or never finished.
`downloadFolderImported` means it worked. `downloadFailed` and
`importFailed` name their own problem.

## Link 1: the request front-end

If users say requests are being **declined**, check whether they actually
failed. Request front-ends often surface a failed hand-off to the arr with
wording that reads as a rejection.

The most common cause is a stale quality profile id in the front-end's server
settings, usually after profiles were renumbered. The front-end's log carries
the real error: adding the item failed because the quality profile does not
exist. Fix the server settings, then **retry** the failed requests rather than
asking users to re-request, because re-requesting hits the duplicate guard.
Details in `arr-quality-profiles`.

## Link 2: is the arr even searching

A search command returning `queued` means accepted. It does not mean anything
was searched for. The most common reason a search matches nothing is that the
target is not monitored, which `arr-library-hygiene` covers in depth.

```bash
# Sonarr: how many episodes are actually armed
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/episode?seriesId=$SID" \
  | jq '[.[] | select(.monitored)] | length'
```

## Link 3: indexers

Run an interactive search and read the rejection reasons. This is the single
most informative thing in the whole chain and it is one call:

```bash
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/release?movieId=$ID" \
  | jq '.[] | select(.rejected) | {title, quality: .quality.quality.name, size, rejections}'
```

If the array is **empty**, the indexers returned nothing and the problem is
sourcing. If it is **full of rejected releases**, the indexers are fine and
your profile, size limits, or format scores are refusing them. Those are
opposite problems and the queue looks identical for both.

Check the indexers themselves:

```bash
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/indexer" \
  | jq '.[] | {name, enable, priority, protocol: .protocol}'
```

An indexer that keeps failing gets automatically disabled for a backoff
period, which presents as "it worked yesterday". Health warnings name it.

**Source diversity is a real fix.** Content that has aged past Usenet
retention will never complete no matter how many times you blocklist and
retry. If a category of titles consistently fails on one protocol, add
indexers on the other protocol rather than tuning the retry behaviour. In
practice large 4K Usenet posts fail repair far more often than 1080p posts of
the same title, because completeness degrades with size and age, so a library
that leans 4K needs a torrent path.

## Link 4: download client

```bash
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/downloadclient" \
  | jq '.[] | {name, implementation, enable, protocol}'
```

Then look at the queue for items that were grabbed but are not progressing:

```bash
curl -sf -H "X-Api-Key: $ARR_KEY" "$ARR_URL/api/v3/queue?pageSize=100" \
  | jq '.records[] | {title, status, sizeleft, timeleft, errorMessage}'
```

### Torrents at zero peers: check port forwarding first

A torrent client behind a VPN with no forwarded port can only make outbound
connections. You get a handful of peers, throughput in the tens of KB/s, and
everything looks "connected" because the VPN reports healthy. The VPN being
healthy and the torrent path being usable are different claims.

Three things to check, in order:

1. **Is port forwarding enabled at all** in the VPN container's configuration.
   It is commonly off by default.
2. **Does your VPN provider offer forwarding in the region you selected.**
   Several major providers do not offer it in US regions specifically. A
   perfectly healthy connection to an unsupported region will never forward a
   port. Switch to a supported region.
3. **Is the forwarded port actually propagated to the client.** Providers
   issue a port on a lease and it changes. The usual pattern is a hook script,
   fired by the VPN container when the port is assigned, that calls the
   torrent client's preferences API to set the listening port. If that hook is
   missing or broken, you have a forwarded port nobody told the client about.

The fix is usually a region change plus enabling forwarding plus the hook, and
throughput moves by two orders of magnitude when it lands.

### Restarting a VPN-wrapped client can leave it down

When the torrent client shares the VPN container's network namespace and
depends on its health check, a redeploy forces a fresh tunnel negotiation. If
that negotiation fails, the VPN sits unhealthy and the client stays gated off
entirely. You have now converted a slow download into no downloads.

Prefer changing configuration over redeploying, keep a copy of the last
working compose or config before editing, and treat "restart it" as a real
intervention rather than a free diagnostic step on this particular container.

### Usenet repair failures

Failed par2 repair on large posts is a retention and completeness problem, not
a client misconfiguration. Confirm by testing whether smaller releases of the
same title complete. If they do, stop tuning the client.

## Link 5: grabbed but never imported

`importBlocked` and `importFailed` in the queue mean the download finished and
the move into the library failed. Causes, roughly in frequency order:

- **Path mapping.** The client reports a completed path the arr cannot see, or
  can see at a different prefix. This is the container-path problem from
  `arr-connect`, arriving from the other direction. Compare the client's
  download directory as the client sees it against the same directory as the
  arr sees it, using `docker inspect` on both.
- **Permissions.** The arr cannot move or delete in the completed directory.
  Check the effective uid of both containers, not the directory's owner name.
- **Already imported.** Usually the phantom-episode loop in
  `arr-library-hygiene`, not a genuine duplicate.

Clearing a stuck queue and the blocklisting behaviour that goes with it is in
`arr-library-hygiene`.

## When to stop looking at the arr

Some symptoms are upstream of the arr entirely and will waste your time if you
treat them as arr faults:

- **Metadata lookups timing out** while the rest of the API is fine: the .NET
  IPv6 stall, in `arr-connect`.
- **Slow throughput on everything, all protocols, all titles**: network or
  storage, not the arr.
- **Files arriving and playing badly**: `arr-playback`. The download chain did
  its job.
