- Feature Name: Stream Type
- Start Date: 2023-03-10
- PR: [video-dev/media-ui-extensions#10](https://github.com/video-dev/media-ui-extensions/pull/10)

# Summary

This is a proposal to provide well defined behaviors and an API for distinguishing between "live" content vs. "on demand" (or pre-recorded).

# Motivation

The idea of different “stream types” has been around for a long time in various HTTP Adaptive Streaming (HAS) standards and its precursors in some manner - minimally distinguishing between “live” content and “video on demand” / "on demand" content. However, these categories aren’t consistently named or distinguished in the same way across the various specifications. Moreover, there is no corresponding API in the browser. Yet these categories directly inform how one expects users to consume and interact with the media, including what sort of UI or “chrome” should be made available for the user. By way of example, the built in controls/UI in Safari that show up for a live source are different than those that show up for a VOD playlist. This proposal aims to normalize the names and definitions of stream types (in a way that is extensible and evolvable over time) by way of how they are expected to be consumed and interacted with by a viewer/user. It also provides a concise and easy to understand differentiator for anyone implementing different UIs/controls/"chromes" for the various stream types.

An additional goal of this proposal is to recommend for MSE-based players or “playback engines” to try to normalize their use of existing APIs to be as consistent as possible with the proposed inferred stream type Algorithm.

# Guide-level Explanation

## Definitions

- __Live Media Stream__ - Media content that is being made available in real-time at the time of playback, and the end of the media (if there is an end), will only be available sometime in the future.
  - __NOTE__ - This definition also applies to "DVR" or "sliding window" content, where the content is being published/provided in real-time but earlier content is still also available.
- __On Demand Media Stream__ - Media content that is entirely available, from beginning to end, at the time of playback. Also sometimes called "Video on Demand" or "VOD."

## Usage

The API provides a single property, `streamType`, that will identify whether the media content is `"live"` or `"on-demand"`. If no source is loaded or the stream type for the source has yet to be determined, the `streamType` will be `"unknown"` (to disambiguate from elements that have not implemented the Stream Type API).

When the stream type is determined for a given media source, the extended media element will dispatch a `"streamtypechange` event.

## Example:

```js
const mediaEl = document.querySelector('#extended-media');

// A "big play button" to show on initial player load/render
const bigPlayButtonEl = document.querySelector('#big-play-button');
// Parent element containing the player's standard UI for both "live" and "on-demand"
const playerUIEl = document.querySelector('#player-ui');
// Element in the UI for indicating playback of live content at the "live edge"
const liveIndicatorEl = playerUIEl.querySelector('#live-indicator');
// Seek bar for the UI
const seekBarEl = playerUIEl.querySelector('#seek-bar');

const updateUI = () => {
  // If the stream type is "unknown" (because no source has
  // been loaded, the previous source has been unloaded, or 
  // the stream type is not yet derived from the source), only 
  // show a "big play button". This also accounts for cases
  // where `preload="none"`.
  if (mediaEl.streamType === 'unknown') {
    playerUIEl.classList.add('hide');
    bigPlayButtonEl.classList.remove('hide');
  }
  // Otherwise, we know the stream type for the current source,
  // so show the appropropriate UI based on the stream type.
  else {
    // For "live", show the live indicator and hide the seek bar
    // NOTE: For "DVR" content ("live" content that is still intended
    // to be seekable), you may want to further differentiate here.
    if (mediaEl.streamType === 'live') {
      liveIndicatorEl.classList.remove('hide');
      seekBarEl.classList.add('hide');
    }
    // For "on-demand", do the opposite.
    // NOTE: In this example, if `mediaEl` has not implemented `streamType`,
    // it will show the same UI as "on-demand".
    else {
      seekBarEl.classList.remove('hide');
      liveIndicatorEl.classList.add('hide');
    }
    // Show the standard UI and hide the "big play button"
    bigPlayButtonEl.classList.add('hide');
    playerUIEl.classList.remove('hide');
  }
};

// Update the UI whenever "streamtypechange" is dispatched.
mediaEl.addEventListener('streamtypechange', updateUI);
// Also update the UI whenever "emptied" is dispatched to account for unloading the media.
mediaEl.addEventListener('emptied', updateUI);
```

# Reference-level explanation

## Property: `streamType`

A string value that represents the type of media stream that *__MUST__* be one of the values listed below.

__NOTE__: Any computation of the `streamType` value that requires fetching of playlists, manifests, or the like, *__SHOULD__* respect the load and `preload` state specified on the extended `HTMLMediaElement` instance. For example, if `preload="none"`, implementors should wait until e.g. a `"loadstart"` event is dispatched from the instance.

### Possible values

- `undefined` - Unimplemented, in which case UI implementors may rely on a "best guess" solution for modeling the stream type (See *§Recommended inferred value from native `HTMLMediaElement`*, below).
- `"on-demand"` - The current loaded media source is an On Demand Stream.
- `"live"` - The current loaded media source is a Live Stream (including potentially "DVR" or "sliding window" live streams).
- `"unknown"` - No media source is set, the source is not yet loaded (e.g. because `preload="none"`), or the stream type has yet to be determined.

### Recommended computation for RFC8216 (aka HLS)

`streamType = (#EXT-X-PLAYLIST-TYPE === "VOD") ? "on-demand" : "live"`

__Context__:

> A Media Playlist has further constraints on its updates if it contains an EXT-X-PLAYLIST-TYPE tag.  An EXT-X-PLAYLIST-TYPE tag with a value of VOD indicates that the Playlist file MUST NOT change.  An EXT-X-PLAYLIST-TYPE tag with a value of EVENT indicates that the Server MUST NOT change or remove any part of the Playlist file, with the exception of EXT-X-PART tags and Media Metadata tags as described above; the Server MAY append lines to the Playlist.

  \- From: https://datatracker.ietf.org/doc/html/draft-pantos-hls-rfc8216bis-12#section-6.2.1

#### NOTE: Why not `#EXT-X-ENDLIST` detection?

The `#EXT-X-ENDLIST` tag only indicates that a particular playlist will no longer be updated with additional Media Segments or media segment parts (`#EXTINF` / `#EXT-X-PART-INF`). Using this value therefore conflates, e.g., an ended live stream with an on demand stream, and thus should not be relied upon for stream type detection.

### Recommended computation for ISO/IEC 23009-1 (aka "MPEG-DASH")

`streamType = (MPD@type === "dynamic") ? "live" : "on-demand`

__Context__:

MPD@type: 
> default: `static`
> specifies the type of the Media Presentation. For static Media Presentations (`@type="static"`), all Segments are available between the `@availabilityStartTime` and the `@availabilityEndTime`. For dynamic Media Presentations (`@type="dynamic"`), Segments typically have different availability times.

\- From: *§5.3.1.2 Table 3 - Semantics of MPD element*

### Recommended inferred value from native `HTMLMediaElement`

```js
streamType = Number.isFinite(mediaEl.duration)
  ? "on-demand"
  : mediaEl.duration === Number.POSITIVE_INFINITY
  ? "live"
  : "unknown"
```

__Context__:

`media.duration`: 
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Returns the length of the media resource, in seconds, assuming that the start of the media resource is at time zero. <br/>
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Returns NaN if the duration isn't available. <br/>
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Returns Infinity for unbounded streams.

\- From: https://html.spec.whatwg.org/multipage/media.html#dom-media-duration-dev


## Event: `"streamtypechange"`

This event should be dispatched from an extended `HTMLMediaElement` instance whenever the stream type has been determined for a loaded media source.

__NOTE__: For media unload cases, implementors *__SHOULD__* ensure the `streamType` is (re)set to `"unknown"` after an unload begins but prior to any `"emptied"` external event handlers being invoked. This ensures that consumers of the API may reliably use the `"emptied"` event to monitor potential changes in `"streamType"`.

# Rationale and alternatives

This proposed solution provides a simple API with simple values for exposing information that is useful for UI development that distinguishes between live and on demand content, but is only guaranteed to be available by the code/entity responsible for parsing the media data. Based on the scope of currently proposed stream types, an alternative implementation could potentially rely on `duration` values (corresponding to the values used in *§Recommended inferred value from native `HTMLMediaElement`*, above). Below is a discussion on why this is not recommended.

## Why not simply rely on `duration`?

While there are elements of various specifications that suggest live content *should*, under most circumstances, have a `duration=Infinity` (See, e.g. https://html.spec.whatwg.org/multipage/media.html#dom-media-duration-dev, https://www.w3.org/TR/media-source-2/#dom-mediasource-setliveseekablerange, https://www.w3.org/TR/media-source-2/#dfn-duration-change), there are a few reasons to avoid this.

1. Although https://html.spec.whatwg.org/multipage/media.html#dom-media-duration-dev strongly suggests that `duration` *__SHOULD__* be `Infinity` for live streams, there is at least some ambiguity between "live" vs. "unbounded" (referencing the language from the specification), so conflating the two may be inappropriate.
2. At least some "playback engines" *__MAY__* set a finite duration for live streams, at least by default. While I do not know of any cases, this may also be true for some native playback environments.
3. As with the case of `#EXT-X-ENDLIST` in RFC8216 (aka "HLS"), the `duration` of live streams that have ended will typically be set to a finite value, leading to the same problems as discussed above at end of stream. This is arguably the most compelling reason to avoid conflating the two.
4. By avoiding unnecessary constraints on `duration`, we leave room for alternative media ui extension proposals that may want to augment this value for other, duration-specific reasons.
5. By using a custom `streamType` property with well-defined enumerated values, we increase legibility for develoeprs and leave room for extending the set of stream types in subsequent proposals, if merited.
5. By using a custom `streamType` property, it is possible to polyfill the `HTMLMediaElement` without *requiring* the use of e.g. Web Components (though this is still recommended & encouraged).