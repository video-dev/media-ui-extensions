- Feature Name: Live Edge
- Start Date: 2023-02-08
- PR: #7

# Summary

This is a proposal to provide well defined behaviors and an API for the "live edge" of live streaming media, including a "window" or "offset" from the live edge to account for the discrete and segmented nature of live edge updating in modern HTTP Adaptive Streaming (HAS) standards.

# Motivation

For live/"DVR" media, client media players will frequently have user interfaces that are specifically related to the "live edge". These include, for example:

- A "live indicator", which indicates to a viewer whether or not they are currently watching at "the live edge" of available media
- A "seek to live" button or similar, that allows a user to easily seek to "the live edge".

## The problem

Unfortunately, due to nature of HTTP Adaptive Streaming (HAS), the live edge cannot be represented as a simple point/moment in the media's timeline. This is for a few reasons:

1. At the manifest/playlist level, the available live segments are typically known by the client by periodically re-fetching these using in-specification polling rules. These files may also be updated by the server in a discontiguous manner as segments are ready for streaming. Together, since the client does not know when segments will be added by the server, the known advertised "true live edge" will "jump" discontiguously through this process, which needs to be accounted for as a plausible "range" or "window" for what counts as the live edge.
2. HAS provides segmented media via a client-side pull based model (most typically, e.g., a GET request), where each segment has a duration. This means that a client must first "see" the segment (via the process described above), then GET and buffer the segment, and then (eventually) play the segment, starting at its start time. Here again, this entails a discontiguous, per-segment update of the timeline, which again needs to be accounted for via a "range" or "window", rather than a discrete point.
3. In order to avoid live edge stalls, both MPEG-DASH and HLS have a concept of a "holdback" or "delay," which inform client players that they should not attempt to fetch/play some set of segments from the end of a playlist/manifest. Luckily, this can be treated as an independent offset calculation applied to e.g. the seekable.end(0) of a media element, which can then be used as a reference for any other live edge window computation.

### A concrete sub-optimal (not worst case) but in spec example - HLS:

Let's say a client player fetches a live HLS media playlist just before the server is about to update it with the following values:
```
# ...
# Unfortunately, EXT-X-TARGETDURATION is only an upper limit (>= any EXTINF duration) after rounding to the nearest integer
#EXT-X-TARGETDURATION: 5
# Client side "LIVE EDGE" will be 5.46 seconds into the segment below, aka 3 * 5 (target duration) = 15 seconds from the playlist end duration
# NOTE: Assume playback begins at the beginning of the segment below, since some client players choose to do this to avoid stalling/rebuffering, meaning playback starts -5.46 seconds from the "LIVE EDGE"
#EXTINF:5.49
#EXTINF:4.99
#EXTINF:4.99
#EXTINF:4.99
```
The server then ends up updating the playlist with two larger-duration segments (in spec and happens under sub-optimal but not unheard of conditions) before the client re-requests the playlist after 4.99 seconds (the **_minimum_** amount of time the player must wait) and continues re-fetching the available segments, with an updated playlist of:
```
# ...
#EXT-X-TARGETDURATION: 5
# NOTE: Current playhead will be 4.99 seconds into the segment below, assuming optimal buffering and playback conditions at 1x playback speed
#EXTINF:5.49
#EXTINF:4.99
#EXTINF:4.99
# New Client side "LIVE EDGE" will be 0.97 seconds into the segment below, aka 3 * 5 (target duration) = 15 seconds from the playlist end duration
#EXTINF:4.99
#EXTINF:5.49
#EXTINF:5.49
```
In this example, playback _**started**_ 5.46 seconds behind the computed "LIVE EDGE" and, after a single reload of the playlist, ended up 11.45 seconds behind the next computed "LIVE EDGE" without any stalls/rebuffering. Note that, even in this example, we do not account for round trip times (RTT) for fetches, time to parse playlists, times to buffer segments, initial seeking of the player's playhead/`currentTime`, and the like. Note also that, even without those considerations, the playhead still ends up > 2 * TARGETDURATION behind the "LIVE EDGE".

## Implications

This means two things:

1. For commonly used HAS specifications, there is a distinction between (a) "the latest advertised media content available from the live streaming media server" and (b) "the latest media content a client should try to play or seek to." an extended `HTMLMediaElement` should have an API that predictably models (b) for player UIs.
2. For commonly used HAS specifications, the "live edge" is better modeled as a "live edge window," and an extended `HTMLMediaElement` should have an API that somehow models this window.

# Guide-level Explanation

## Definitions

- __Advertised Live Edge__ - This is the latest available media time as represented in a manifest, playlist, or simiilar 
- __Seekable Live Edge__ - This is the latest available media time a player should attempt to seek to or play, typically some offset from the Advertised Live Edge.
- __Live Edge Window__ - This is the range of time between the Seekable Live Edge and some offset that should be treated as the live edge for visualization purposes, given the segmented nature of HAS media delivery and updates.

## Usage

The API will both constrain `seekable.end()` to mean Seekable Live Edge and add a new `liveEdgeOffset` property so UI implementors can rely on these to indicate live playback and/or seek to live functionality.

## Example:

```js
const mediaEl = document.querySelector('#extended-media');

// Indicating that playback is in the Live Edge Window
const liveIndicatorEl = document.querySelector('#live-indicator');
mediaEl.addEventListener('timeupdate', () => {
  const liveWindowStart = mediaEl.seekable.end(mediaEl.seekable.length -1) - mediaEl.liveEdgeOffset;
  const playingLive = mediaEl.currentTime >= liveWindowStart;
  liveIndicatorEl.innerHtml = playingLive ? 'LIVE' : 'NOT LIVE';
});

// Seeking to the Live Seekable Edge.
const seekToLiveEl = document.querySelector('#seek-to-live-button');
seekToLiveEl.addEventListener('click', () => {
  mediaEl.currentTime = mediaEl.seekable.end(mediaEl.seekable.length -1);
});
```

# Reference-level explanation

## Property: Constraint on `HTMLMediaElement::seekable.end()` for Seekable Live Edge

To guarantee reliable representation of the Seekable Live Edge, `HTMLMediaElement`s should ensure that `seekable.end(seekable.length - 1)` is set to this expected value. For HAS that relies on Media Source Extensions, this can be done using `MediaSource::setSeekableLiveRange()`. Below is the recommended methods of computation of this value for common HAS standards.

For more information, see:
- `seekable` ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTMLMediaElement/seekable)) ([WHATWG Spec](https://html.spec.whatwg.org/multipage/media.html#dom-media-seekable))
- `setSeekableLiveRange()` ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/MediaSource/setLiveSeekableRange)) ([W3C Spec](https://w3c.github.io/media-source/#dom-mediasource-setliveseekablerange))

### Recommended computation of Live Edge - RFC8216bis12 (aka "HLS")

1. __"Standard Latency" Live__

- Let Advertised Live Edge be the computed duration of the live stream, based on a sum of the `#EXTINF` duration values.
- Seekable Live Edge = Advertised Live Edge - (`#EXT-X-SERVER-CONTROL:HOLD-BACK` value ?? (`#EXT-X-TARGETDURATION` value * 3))

__Context__:
> HOLD-BACK
>
> The value is a decimal-floating-point number of seconds that indicates the server-recommended minimum distance from the end of the Playlist at which clients should begin to play or to which they should seek, unless PART-HOLD-BACK applies. Its value MUST be at least three times the Target Duration.
>
> This attribute is OPTIONAL. Its absence implies a value of three times the Target Duration. It MAY appear in any Media Playlist.

  \- From: https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis-12#section-4.4.3.8


2. __Low Latency Live__

- Let Advertised Live Edge be the computed duration of the live stream, based on a sum of the `#EXTINF` duration values / `#EXT-X-PART:DURATION` values.
- Seekable Live Edge = Advertised Live Edge - (`#EXT-X-SERVER-CONTROL:PART-HOLD-BACK` value)

__Context__:

> PART-HOLD-BACK
> 
> The value is a decimal-floating-point number of seconds that indicates the server-recommended minimum distance from the end of the Playlist at which clients should begin to play or to which they should seek when playing in Low-Latency Mode. Its value MUST be at least twice the Part Target Duration. Its value SHOULD be at least three times the Part Target Duration. If different Renditions have different Part Target Durations then PART-HOLD-BACK SHOULD be at least three times the maximum Part Target Duration.

  \- From: https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis-12#section-4.4.3.8

### Recommended computation of Live Edge - ISO/IEC 23009-1 (aka "MPEG-DASH")

1. __"Standard Latency" Live__

- Let Advertised Live Edge be the computed latest presentation time / duration available as derived from the `MPD` at time T, using `MPD@availabilityStartTime` for reference, and presuming an NTP time-synchronized client and server.
  - NOTE: Since there are many ways to represent `AdaptationSet`s and their segments, describing the various ways of computing the Advertised Live Edge will be treated as out of scope for this proposal.
- Seekable Live Edge = Advertised Live Edge - (`MPD@suggestedPresentationDelay` value ?? client player specific value ?? 0)

__Context__:

MPD@suggestedPresentationDelay: 
> it specifies a fixed delay offset in time from the presentation time of each access unit that is suggested to be used for presentation of each access unit... When not specified, then no value is provided and the client is expected to choose a suitable value.

  \- From: *§5.3.1.2 Table 3 - Semantics of MPD element*


2. __Low Latency Live__

- Let Reference Time be derived from the `ProducerReferenceTime` values which correlate to the presentation timeline and wallclock times 
  - NOTE: Since there are many ways to represent `AdaptationSet`s and their segments, describing the various ways of computing these values will be treated as out of scope for this proposal.
- Seekable Live Edge = Reference Time - (`ServiceDescription` -> `Latency@target` value)

__Context__:

> The service provider's preferred presentation latency in milliseconds compared to the producer reference time. Indicates a content provider's desire for the content to be presented as close to the indicated latency as is possible given the player's capabilities and observations.
>
> This attribute may express latency that is only achievable by low-latency players under favourable network conditions.

  \- From: *Annex K, §K.3.2 Table K.1 - Service Latency* (See Also: *Table K.6 — Semantics of Latency element* from the same section)

## Property: `liveEdgeOffset`

An offset or delta from the Seekable Live Edge (`seekable.end(seekable.length -1)`, appropriately constrained as defined by this specification). An extended media element is playing within the Live Edge Window iff: `currentTime >= (seekable.end(seekable.length - 1) - liveWindowOffset`).

### Possible values

- `undefined` - Unimplemented
- `NaN` - "unknown" or "inapplicable" (e.g. for `streamType = "on-demand"`)
- `0 <= x <= Number.MAX_SAFE_INTEGER` - known stable value for current stream

### Recommended computation for RFC8216bis12 (aka HLS)

1. **"Standard Latency" Live**

`liveEdgeOffset = 3 * #EXT-X-TARGETDURATION`

Note that this is a cautious computation. In many stream + playback scenarios, `2 * EXT-X-TARGETDURATION` will likely be sufficient. However, with this less cautious value, there may be edge cases where standard playback will "hop in and out of the live edge," so recommending the more cautious value here.

2. **Low Latency Live**

`liveEdgeOffset = 2 * #EXT-X-PART-INF:PART-TARGET`

Unlike "standard" segments (`#EXTINF`s), parts' durations _**must**_ be <= `#EXT-X-PART-INF:PART-TARGET` (without rounding). Also unlike "standard," HLS servers _**must**_ add new partial segments to playlists within 1 (instead of 1.5) Part Target  Duration after it added the previous Partial Segment. This means that, even under sub-optimal conditions, low latency HLS _**should**_ end up with a much smaller `liveEdgeOffset`.

### Recommended computation for ISO/IEC 23009-1 (aka "MPEG-DASH")

TBD

## Event: `liveedgeoffsetchange`

An event that fires whenever the `liveEdgeOffset` value changes. Note that this should only ever fire once per `src` load, and is expected to fire as soon after loading sufficient metadata to compute the value.

# Rationale and alternatives

This proposed solution provides a simple API with simple values for exposing information that is useful for live UI development but is only available by the code/entity responsible for parsing the media data. We could go with a more complex API (e.g. having a `TimeRange` to represent the live window, adding a new interface instead of constraining `seekable.end()`, etc.), but it's unclear that this would offer any advantage over the proposed simplified solution. UI implements could also "guess" at a reasonable value for the `liveEdgeOffset`, but this would likely lead to both false positives and false negatives for determining whether or not playback is in the Live Edge Window.
