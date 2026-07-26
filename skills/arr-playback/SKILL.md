---
name: arr-playback
description: >-
  Use when playback quality is the real complaint but the arr is the real
  cause: a 4K file that stutters or shows grey glitches in one player but
  plays fine in another, "why is this transcoding when direct play should
  work", constant CPU or GPU load during playback, subtitles that force a
  re-encode, audio that gets converted every time, files that will not
  direct-play on a streaming box, or deciding which codecs, HDR formats,
  and audio tracks a profile should prefer for the hardware you actually
  own. Also covers post-import remux pipelines and their integrity gates.
  Assumes arr-quality-profiles for how to express these preferences. Not
  for missing files or monitoring (arr-library-hygiene).
compatibility: Applies to any arr feeding a transcoding media server (Plex, Jellyfin, Emby). Client behaviour notes are about streaming boxes and set-top players generally, not one vendor.
---

# Arr Playback

The arr decides what lands on disk. The media server then has to hand that
file to a client. If the two disagree, the server transcodes, and transcoding
is where the stutter, the fan noise, and the "4K is broken" reports come from.

This skill is the feedback loop that most profile guides omit: **tune the
grab to the hardware that will play it**, not to an abstract quality ideal.

## Diagnose before you touch a profile

Every transcode has a reason and the media server will tell you what it was.
Get the playback decision before theorising:

- Check whether the session is **direct play**, **direct stream**, or
  **transcode**. Only the last one re-encodes video.
- If it is a transcode, find which stream forced it: video, audio, or subtitle.
- Reproduce with subtitles off, then with a different audio track. Each
  toggle isolates one cause in about thirty seconds.

Then look at the file itself:

```bash
ffprobe -v error -show_entries stream=index,codec_type,codec_name,profile,pix_fmt \
  -of compact "$FILE"
```

Two signatures worth separating early, because they look identical to a viewer:

- **Transcode session, GPU or CPU busy**: a format problem. This skill.
- **Direct play session, no encoder load, stutter anyway**: not a format
  problem. Look at the client's thermals, the network path, or storage
  throughput. Changing quality profiles will not help and will waste a day.

## The big one: image subtitles force a full re-encode

Bluray and UHD sources carry PGS subtitles, which are images rather than text.
Media servers generally cannot overlay an image subtitle track onto a video
stream the way they can with text. So when subtitles are enabled, the server
**burns them into the picture**, and burning in means re-encoding the entire
video: 4K HEVC, HDR and all.

The tell is exact: the title stutters or shows momentary grey or garbled
frames **with subtitles on**, plays perfectly **with subtitles off**, and
plays perfectly in a client that renders PGS itself. Players that do their own
subtitle rendering direct-play the same file that the media server chokes on,
which is why "it works in the other app" is a diagnosis rather than a mystery.

It also degrades over time within a single playback, because the encoder
falls further behind as it goes.

Three responses, in order of durability:

1. **Convert on import.** A post-import remux that drops PGS tracks and
   converts text subtitles to a container-native text format (`mov_text` for
   MP4) makes files direct-play with subtitles on. See the remux section
   below.
2. **Prefer sources with text subtitles** in your profile scoring where a
   choice exists.
3. **Turn subtitles off**, which is not a fix, but it does confirm the
   diagnosis in seconds.

Do not respond to this by reverting your quality profiles to a template
default. Common defaults prefer lossless audio and permit HDR formats that add
*more* transcode triggers, so the reset makes playback worse while feeling
like a clean slate.

## HDR format choices are client-dependent

Dolby Vision profiles that carry no HDR10 fallback force a tone-map on any
display that cannot handle DV, and a tone-map is a full transcode. On a setup
whose display does HDR10 or HDR10+ but not DV, a DV-only release is the worst
possible grab even though it reads as the most premium.

Express this in custom format scores rather than hoping:

- Block DV-without-HDR10-fallback outright. Score it strongly negative.
- Treat DV profiles that carry an HDR10 base as neutral: they play as HDR10 on
  a non-DV display and cost nothing.
- Positively score the HDR format your display actually implements.

Revisit these scores when you change displays. They encode a hardware fact,
not a preference, and they are the first thing that goes stale after a TV
upgrade.

One more wrinkle: **a container remux strips DV signalling**. If a post-import
pipeline rewrites files into another container, DV metadata may not survive
it, which is another reason to prefer an HDR10 base rather than depending on
DV making it to the screen.

## Codec and audio traps

**A codec your client cannot hardware-decode forces a transcode regardless of
everything else.** AV1 is the current common case: many streaming boxes in
service today have no AV1 decoder, so an AV1 release transcodes on every play
and no remux can fix it. The fix is re-grabbing in a codec the client decodes,
usually HEVC. If your profile has started accepting AV1, that is worth
checking before blaming the server.

**Server-side decode has holes too.** Hardware decoders advertise broad codec
support and then fail on specific profiles. Ten-bit H.264 is a well-known gap
on several integrated and entry-level discrete GPUs: the codec is on the
support list, the bit depth is not, and the result is either a silent fallback
to CPU or an outright transcode failure. When transcoding fails only for a
subset of files that share a `pix_fmt`, suspect a profile gap rather than a
broken install.

**Audio is the quiet transcoder.** Object-based and lossless tracks (TrueHD,
DTS-X, and friends) commonly get converted for clients that cannot take them
or for anything not on a full passthrough path. If your client tops out at
lossy multichannel, scoring lossless audio highly guarantees a conversion on
every play. Score the format your chain actually passes through.

## Post-import remux pipelines

Rewriting imported files into a container that direct-plays on your clients is
a large win: it fixes subtitles, normalises containers, and removes a class of
compatibility failure permanently. It is also a pipeline that silently deletes
your originals, so its integrity gate matters more than its speed.

**Gate trap: container duration headers lie.** The obvious integrity check is
"does the output duration match the input duration", read from the container
header (`format=duration`). Many MKV rips carry an inflated header: it claims
minutes more than where the video stream actually ends. A clean remux then
fails the gate, the file gets struck, and healthy titles pile up at the
failure cap while the pipeline reports problems it does not have.

Compare the **true end of the video stream** instead, and keep the header as a
fallback:

```bash
# true end: last video packet's presentation timestamp
ffprobe -v error -select_streams v:0 -show_entries packet=pts_time \
  -of csv=p=0 "$FILE" | tail -1

# header claim, for comparison
ffprobe -v error -show_entries format=duration -of csv=p=0 "$FILE"
```

A gate built on the last video packet still catches genuine truncation, which
is the thing you actually wanted to catch.

**Separate the two failure classes** when files are stuck at a failure cap.
They need opposite responses:

1. **Inflated header** produces a false gate failure on a perfectly good
   remux. Confirm by comparing header duration to last-packet time. Fix the
   gate, then retry the struck files.
2. **A genuinely corrupt source** produces a real encoder error: invalid NAL
   unit size, missing picture in access unit, conversion failed. The remux is
   correctly refusing. The fix is a re-grab, not a code change.

Reproduce a failing file with full stderr before deciding which class it is.
Pipelines usually log only the tail of the error, which is rarely the line
that identifies the cause.

**Codecs the pipeline cannot handle should be skipped, not failed.** Older
codecs (XviD, VC-1) belong in a skip list so they remain visible as re-grab
candidates rather than accumulating as failures.

**Sequencing matters.** Newly grabbed files are the ones most likely to still
be in the problem format, so process newest first. That shrinks the window
between "grabbed" and "playable without transcoding" to a single run.

## The rule to carry away

When someone reports a playback problem, resist the urge to change the quality
profile first. Get the playback decision, isolate the offending stream, and
confirm the file's actual codecs. Only then does a profile change make sense,
and when it does, it should encode a fact about your hardware that you can
state out loud: this display does HDR10 and not DV, this box has no AV1
decoder, this chain passes lossy multichannel and not TrueHD.

Profiles written that way stay correct. Profiles copied from a guide describe
someone else's living room.
