# Primer: casting on-demand HLS to a DLNA renderer

This explains four problems found while casting a recorded (on-demand) HLS video to a Meta Quest 2
running 4XVR as its DLNA receiver, and how each fix works. The problems surfaced one at a time:
each fix revealed the next.

## Background

**HLS.** A streaming video on a website is usually HLS. It is not one file, but:

- a **playlist** (`.m3u8`): a text list of parts, e.g. "part 1, part 2, ... part 216, each 4 seconds"
- **segments**: small video files of about 4 seconds each

A player reads the playlist and fetches the segments one by one.

**On-demand vs live.** In a live stream, new segments appear slowly, in real time, and only "now"
matters. In an on-demand (recorded) video, every segment already exists, and every byte matters,
in order.

**The helper in the middle.** The renderer cannot fetch from the site directly, because the site
requires request headers such as `Referer` and `Origin`. The helper fetches everything and adds
those headers.

**DLNA.** With DLNA the helper tells the renderer "play this URL" (`SetAVTransportURI` + `Play`), and
the renderer's own player does the rest. The helper then polls the renderer's state
(`GetTransportInfo`) every 2 seconds to watch the cast.

## Problem 1: the video kept restarting from the beginning

**Symptom.** Only the opening of the video played, over and over.

**Cause.** The DLNA path was built for live streams: streamlink pulls the stream and the helper pipes
it to the renderer as one continuous MPEG-TS feed. On an on-demand source streamlink has no live
edge to wait for, so it pulled the whole video (179 MB) in about 10 seconds and exited. Then:

1. The pipe holds at most 64 MB waiting for the renderer. Data arrived far faster than the renderer
   played it, so the oldest bytes were **dropped** to stay under the cap.
2. When streamlink exited, the helper treated it as a live dropout and **respawned from the
   start**, every 10 seconds.

The renderer got the start, a jumble from later in the video, then the start again.

**Fix.** For an on-demand source only:

- when the 64 MB queue is full, the reader **waits** for the renderer instead of dropping bytes
- when a full run ends, the stream **ends**, instead of respawning

**Why it works.** A live stream can afford to lose old data. A recorded video needs every byte,
exactly once, in order.

## Problem 2: you could not jump to a time

**Cause.** The live pipe deliberately tells the renderer the stream is live and not seekable:

| Signal | Value | Meaning to the renderer |
|---|---|---|
| `Content-Length` | absent | length unknown, nothing to scrub through |
| `Accept-Ranges` | `none` | cannot ask for "bytes from X" |
| `contentFeatures.dlna.org` | `DLNA.ORG_OP=00` | neither byte nor time seeking |

On a seek attempt the renderer reconnected, and each new connection started streamlink from the top.

**Fix.** For an on-demand HLS source, the helper hands the renderer **the playlist itself**, served
through its authenticating HLS proxy. Every URL in it is rewritten to come back through the helper,
which adds the site's headers to each fetch.

**Why it works.** With the playlist, the renderer's player knows "216 parts x 4 s = 14:25". To jump
to 10:00 it simply fetches part 150. The helper does no seeking; the player does, just as a browser
does.

**Related fix: pause and seek are not dropouts.** The watch loop counted any state other than
`PLAYING` or `TRANSITIONING` as a dropout and re-pushed the URL after ~6 s, which restarts a playlist
at 0:00. So pausing (`PAUSED_PLAYBACK`) or seeking reset playback. For a playlist, a pause now counts
as active, a re-push waits ~16 s, and the log names the state that triggered it.

## Problem 3: the playlist loaded, then playback failed

**Symptom.** 4XVR fetched the playlist and three segments, then reported a failure.

**Cause.** The segments were **AV1**, a newer and more efficient codec. The **Quest 2 has no AV1
decoder**. The site's master playlist labels the video `avc1` (H.264), but the segments are AV1. The
helper's classifier trusted the label, which is why the log said `vcodec=h264`. Probing a real
segment with `ffprobe` showed `av1`.

**Fix.** This CDN stores both encodings under similar names:

| Playlist | Codec | Quest 2 |
|---|---|---|
| `1080.av1.mp4.m3u8` | AV1 (fMP4 segments) | cannot decode |
| `1080.mp4.m3u8` | H.264 High + AAC (MPEG-TS segments) | plays |

The helper reads the real codec from the segments, not the master. If it is AV1, it tries the same
name without `.av1`, confirms that rendition is not AV1, and casts it instead.

**Why it works.** The renderer gets a codec its hardware decodes, and nothing is transcoded, so it
costs no CPU on the computer.

## Problem 4: the first 3 seconds looped

**Cause.** Logging every request showed that 4XVR reads each segment in **1 MiB byte ranges**:

```
seg-1-v1-a1.ts  Range: (none)            -> proxy sent 213944 bytes
seg-1-v1-a1.ts  Range: bytes=1048576-    -> proxy sent 213944 bytes
seg-1-v1-a1.ts  Range: bytes=2097152-    -> proxy sent 213944 bytes
...             up to bytes=33554432-
```

The proxy ignored `Range` and sent the **whole segment from its start** every time, with no
`Content-Length`. Segment 1 is only 214 KB, so 4XVR received it about 33 times in a row and played
its 3 seconds 33 times.

**Fix.** The proxy honours range requests:

| Request on a 213944-byte segment | Response |
|---|---|
| no `Range` | `200`, `Content-Length: 213944` |
| `bytes=100000-` | `206`, `Content-Range: bytes 100000-213943/213944` |
| `bytes=-1000` | `206`, the last 1000 bytes |
| `bytes=1048576-` | `416`, `Content-Range: bytes */213944` (past the end) |

A ranged request reads the segment whole (it is a few MB at most) and returns just the slice.

**Why it works.** 4XVR gets each byte once, and the `416` tells it the segment is complete, so it
moves on to the next one. Most players, including the Cast receiver, do not read segments in ranges,
which is why this never showed before.

## Summary

| Commit | Problem | Fix in one line |
|---|---|---|
| Play an on-demand source through to its end on a DLNA renderer | restart loop | wait instead of dropping; end, don't respawn |
| Serve a proxied segment by the byte range the player asks for | 3-second loop | answer "bytes from X" properly |
| Hand a DLNA renderer an on-demand HLS playlist so it can seek | could not seek | give the renderer the playlist; a pause is not a dropout |
| Cast the H.264 rendition of an AV1 HLS source to a DLNA renderer | playback failed | swap AV1 for the site's own H.264 rendition |

## Limits

- **The AV1 swap relies on the CDN's naming** (`.av1.` in the file name). A site that offers only
  AV1 still fails on a renderer without an AV1 decoder. The general fix would be transcoding to
  H.264, which the Google Cast path already does.
- **Live streams keep the live pipe**, which is right for them: there is nothing ahead to seek to.
- **Tested renderer:** 4XVR on a Quest 2, playing an MPEG-TS HLS playlist. fMP4 HLS on 4XVR is
  untested, since the only fMP4 source tried was AV1.
