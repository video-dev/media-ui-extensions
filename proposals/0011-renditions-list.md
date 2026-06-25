- Feature Name: Renditions List
- Start Date: 2023-04-06
- PR: [video-dev/media-ui-extensions#11](https://github.com/video-dev/media-ui-extensions/pull/11)

# Summary
[summary]: #summary

This proposal introduces an addition to existing APIs to be able to see and control the quality level (rendition) for currently playing HTTP Adaptive Streaming (HAS) video.

# Motivation
[motivation]: #motivation

With the prevalence of segmented media formats like HLS and DASH, playback uses Adaptive Bitrate (ABR) algorithms to choose a reasonable quality for the user based on things like network conditions. However, users often want more control over the quality of the stream that they are watching. For example, to force a high quality rendition for playback when watching a movie or a screen recording.

This proposal provides the mechanism to be able to see what renditions are available for the media that is being played and to change the current selection.

## Goals
- Introduce an API that lists the available renditions for the currently selected video track and allows selecting which rendition is used.
- Take inspiration from existing `TrackList`s so the shape is familiar and a candidate for eventual standardization.
- Keep the consumer-facing API as simple as possible so that it is easy for UI implementations to consume and easy for playback engines to populate.
- Support streams that can be interoperable via CTA-WAVE's [CTA-5005][] specification.

## Non-Goals
- Provide comprehensive support for all potential valid segmented streams with multi-period and discontinuity support.
- Expose all potentially available information on the renditions. Only the minimum set useful for building a UI is included.
- Impose a configuration interface (e.g. min/max resolution caps) on playback engines. How a player restricts which renditions are available is left up to the player.

# Guide-level Explanation
[guide-level-explanation]: #guide-level-explanation

The main addition in this proposal is a `VideoRenditionList`, exposed directly on the media element as `video.videoRenditions`. The list represents the renditions of the **currently selected video track**, and each entry is a `VideoRendition` that includes the minimum set of information requested by HAS formats (resolution, bitrate, frame rate, codec).

Surfacing the list at the media-element level (rather than hanging it off of each `VideoTrack`) keeps the common case simple: a quality-selection UI only needs to know which renditions the viewer can choose from for what is currently playing, and which one is selected. It does not need to reason about the renditions of inactive video tracks or stack rendition listeners on top of track listeners.

For the symmetric audio case, an `AudioRenditionList` is exposed as `video.audioRenditions`, representing the renditions of the currently enabled audio track.

## Selecting a specific rendition

To switch to a specific rendition, set `selectedIndex` on the list. Setting it to `-1` returns control to the ABR algorithm ("auto").

```js
// assuming a video is already loaded
const video = document.querySelector('video');

// switch to the 3rd rendition
video.videoRenditions.selectedIndex = 2;

// hand control back to the ABR algorithm
video.videoRenditions.selectedIndex = -1;
```

## Constraining which renditions the ABR algorithm may use

Each `VideoRendition` also has a writable `selected` boolean. Unlike `selectedIndex` (which selects exactly one rendition), setting `selected` on individual renditions lets you express the set of renditions the engine is allowed to play. For example, to limit playback to SD content only, keep just the renditions whose height is `720` or less selected:

```js
// assuming a video is already loaded
const video = document.querySelector('video');

for (const rendition of video.videoRenditions) {
  rendition.selected = rendition.height <= 720;
}
```

If the currently active rendition becomes unselected, the engine should switch to one of the still-selected renditions.

## Building a quality selector menu

The list dispatches `addrendition`, `removerendition`, and `change` events, so a UI can keep a menu in sync as renditions are added (e.g. once the manifest is parsed), removed (e.g. when switching the active video track), or selected (either manually or by the ABR algorithm).

```js
const video = document.querySelector('video');
const renditionList = video.videoRenditions;

function renderMenu() {
  const options = Array.from(renditionList, (rendition, index) => ({
    index,
    height: rendition.height,
  }));

  options.sort((a, b) => b.height - a.height);

  // ...build an "Auto" option plus one <option> per rendition...
  // The currently selected option is renditionList.selectedIndex,
  // or "Auto" when selectedIndex === -1.
}

renditionList.addEventListener('addrendition', renderMenu);
renditionList.addEventListener('removerendition', renderMenu);
renditionList.addEventListener('change', renderMenu);
```

Changing `selectedIndex` (or a rendition's `selected`) manually will trigger a `change` event on the list. The ABR algorithm changing the active rendition should also trigger a `change` event, so a single handler keeps the UI consistent regardless of who initiated the change.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Below are WebIDL definitions for the changes and some explanation around them. These mirror the API that has been implemented and validated in the [`media-tracks`](https://github.com/muxinc/media-elements/tree/main/packages/media-tracks) polyfill.

## Media element additions

The media element is augmented with two read-only rendition lists:

```ts
partial interface HTMLMediaElement {
  readonly attribute VideoRenditionList videoRenditions;
  readonly attribute AudioRenditionList audioRenditions;
};
```

`videoRenditions` always reflects the renditions of the currently selected `VideoTrack`; `audioRenditions` reflects the renditions of the currently enabled `AudioTrack`. When the selected track changes, the contents of the corresponding list change accordingly (dispatching `removerendition`/`addrendition` events as appropriate).

## `VideoRenditionList`

`VideoRenditionList` is similar to [`VideoTrackList`](https://html.spec.whatwg.org/multipage/media.html#videotracklist). The main difference, aside from naming, is the addition of a read-write `selectedIndex` property.

`selectedIndex` defaults to `-1` when no rendition is explicitly selected (for example before metadata has loaded, or when playback is left to the ABR algorithm). Otherwise it reflects the index of the currently selected rendition. Setting it to a number requests a rendition change to that rendition; setting it to `-1` returns control to the ABR algorithm.

```ts
interface VideoRenditionList : EventTarget {
  readonly attribute unsigned long length;
  getter VideoRendition (unsigned long index);
  VideoRendition? getRenditionById(DOMString id);
  attribute long selectedIndex;

  attribute EventHandler onchange;
  attribute EventHandler onaddrendition;
  attribute EventHandler onremoverendition;
};
```

When the active rendition changes, either via manually setting `selectedIndex`/`selected` or automatically via the ABR algorithm, a `change` event is dispatched. Implementations should coalesce multiple selection changes within a single tick into a single `change` event.

## `VideoRendition`

Each `VideoRendition` includes some metadata about the rendition, such as `src`, `width`, `height`, `bitrate`, `frameRate`, and `codec`. It also includes a writable `selected` boolean that determines whether the ABR algorithm is allowed to switch to this rendition.

```ts
interface VideoRendition {
  readonly attribute DOMString src;
  readonly attribute DOMString id;
  readonly attribute unsigned long width;
  readonly attribute unsigned long height;
  readonly attribute unsigned long bitrate;
  readonly attribute double frameRate;
  readonly attribute DOMString codec;
  attribute boolean selected;
};
```

Notes:
- `codec` is a `DOMString` (e.g. `"avc1.640028"`), not a numeric value.
- Multiple renditions may be `selected` at once to constrain the set the ABR algorithm may choose from. Setting `selectedIndex` is a convenience that selects exactly one rendition and unselects the rest.
- If the currently active rendition is unselected, the engine should request a rendition change to a still-selected rendition.

## `AudioRenditionList` and `AudioRendition`

Audio renditions follow the same shape as video, omitting the video-only resolution and frame-rate fields:

```ts
interface AudioRenditionList : EventTarget {
  readonly attribute unsigned long length;
  getter AudioRendition (unsigned long index);
  AudioRendition? getRenditionById(DOMString id);
  attribute long selectedIndex;

  attribute EventHandler onchange;
  attribute EventHandler onaddrendition;
  attribute EventHandler onremoverendition;
};

interface AudioRendition {
  readonly attribute DOMString src;
  readonly attribute DOMString id;
  readonly attribute unsigned long bitrate;
  readonly attribute DOMString codec;
  attribute boolean selected;
};
```

## Populating renditions (playback engine side)

Renditions are organized under the track they belong to, and a playback engine (e.g. an MSE-based engine like hls.js or shaka) populates them as it parses the manifest. In the `media-tracks` polyfill this is done by adding renditions to the relevant track:

```ts
partial interface VideoTrack {
  VideoRendition addRendition(
    DOMString src,
    optional unsigned long width,
    optional unsigned long height,
    optional DOMString codec,
    optional unsigned long bitrate,
    optional double frameRate
  );
  undefined removeRendition(VideoRendition rendition);
};

partial interface AudioTrack {
  AudioRendition addRendition(
    DOMString src,
    optional DOMString codec,
    optional unsigned long bitrate
  );
  undefined removeRendition(AudioRendition rendition);
};
```

Only renditions belonging to the currently selected video track (or enabled audio track) appear in `video.videoRenditions` / `video.audioRenditions`, and `addrendition` / `removerendition` events are only dispatched for the active track.

### Recommended mapping for RFC8216 (aka HLS)

Each `#EXT-X-STREAM-INF` variant maps to a `VideoRendition`:
- `src` ← the variant playlist URI
- `width` / `height` ← `RESOLUTION`
- `bitrate` ← `BANDWIDTH` (or `AVERAGE-BANDWIDTH` where present)
- `frameRate` ← `FRAME-RATE`
- `codec` ← the video portion of `CODECS`

Variants that reference the same `AUDIO` group share audio renditions defined by the corresponding `#EXT-X-MEDIA:TYPE=AUDIO` entries, which map to `AudioRendition`s on the matching `AudioTrack`.

### Recommended mapping for ISO/IEC 23009-1 (aka "MPEG-DASH")

Each video `Representation` within a video `AdaptationSet` maps to a `VideoRendition`:
- `src` ← the `Representation`'s `BaseURL` / resolved media URL
- `width` / `height` ← `@width` / `@height`
- `bitrate` ← `@bandwidth`
- `frameRate` ← `@frameRate`
- `codec` ← `@codecs`

Audio `Representation`s within audio `AdaptationSet`s map to `AudioRendition`s on the corresponding `AudioTrack`.

# Drawbacks
[drawbacks]: #drawbacks

The main drawback to the above proposal is that implementing this in userland is harder, since we are extending built-in pieces of HTML. However, since one of the goals of these proposals is to eventually land in the HTML spec, this is worth doing because the API would be more likely to stay the same if that happens.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

An earlier version of this proposal hung the rendition list off of each `VideoTrack` (`videoTrack.renditions`) and used an `enabled` flag per rendition. While intuitive from a data-modeling perspective, it required consumers to stack rendition listeners on top of track listeners, which made the common quality-selection UI more complex to build than necessary.

This version makes the following changes based on implementation experience in the `media-tracks` polyfill and feedback on the proposal:

- **Surface the list at `video.videoRenditions`.** A quality-selection UI only cares about the renditions of what is currently playing and which one is selected. Surfacing a flat list at the media-element level keeps that common case simple while still allowing the underlying engine to organize renditions per track.
- **Keep the list read-only with respect to membership.** Externally adding/removing renditions, or imposing min/max caps, is left to playback engines; this avoids dictating a configuration interface that not every engine can support.
- **Use `selected` rather than `enabled`.** This aligns with track terminology and keeps the door open for both "select exactly one rendition" (via `selectedIndex`) and "constrain the set the ABR algorithm may use" (via per-rendition `selected`).
- **Type `codec` as a `DOMString`.** The earlier draft typed it as a number, which could not represent codec strings such as `"avc1.640028"`.

Some of the prior-art APIs are either overly complex or too simple to allow the flexibility above. For example, we want to be able to constrain the set of renditions the underlying engine switches between, while still allowing fully automatic ABR.

# Prior art
[prior-art]: #prior-art

Most web-based players that play back HAS have some kind of API to control renditions.

## Existing APIs in Web Players

### Video.js's videojs-contrib-quality-levels

Video.js has [videojs-contrib-quality-levels][]. It's a simple API that was designed to be augmented into a proposal like this. Here, each quality level is represented like so:
```
Representation {
  id: string,
  width: number,
  height: number,
  bitrate: number,
  frameRate: number,
  enabled: boolean
}
```
`frameRate` is a recent addition.

In this API, each level can be separately enabled or disabled via the enabled getter and setter.

### Hls.js's Quality switch Control API

Hls.js has their [Quality switch Control API](https://github.com/video-dev/hls.js/blob/master/docs/API.md#quality-switch-control-api).

The level object has all the details available in the manifest. It's possible to control which level is used via the `currentLevel` setter.
It is only possible to set a level cap to restrict which levels are allowed to be used. If a user wishes to have a more complex level selection, they'll need to create their own ABR Controller.

### Shaka Player's Variant Tracks

Shaka Player has a [`getVariantTracks()`](https://shaka-player-demo.appspot.com/docs/api/shaka.Player.html#getVariantTracks) method that returns a list of [Tracks](https://shaka-player-demo.appspot.com/docs/api/shaka.extern.html#.Track). Each Track object includes a lot of the information available from the manifests.
It is possible to select a particular variant via [`selectVariantTrack`](https://shaka-player-demo.appspot.com/docs/api/shaka.Player.html#selectVariantTrack).
It's also possible to configure the player with [ABR Restrictions](https://shaka-player-demo.appspot.com/docs/api/shaka.extern.html#.Restrictions) to limit the minimum and maximum resolutions or bitrates that can be used.

It's also possible to provide your own ABR Manager for more complex configurations.

### Other players

Dash.js also has some APIs around this.

## Reference implementation

The [`media-tracks`](https://github.com/muxinc/media-elements/tree/main/packages/media-tracks) package polyfills the media elements with audio and video tracks (as [specced](https://html.spec.whatwg.org/multipage/media.html#media-resources-with-multiple-media-tracks)) plus the renditions described in this proposal, and is consumed by UI implementations such as [media-chrome](https://github.com/muxinc/media-chrome).

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should there be a separate read-only signal for the *active* rendition (currently playing) distinct from the *selected* rendition(s) the ABR algorithm may choose from? When all renditions are selected, a UI may still want to display which one the ABR algorithm has activated.
- How should `codec` handle in-mux audio? When the audio track is multiplexed with the video track, should the rendition's `codec` include both video and audio codecs (or should this be a full mime `type` string to match `MediaSource.addSourceBuffer` and the `<source>` element's `type` attribute)?
- Is there anything here that would need to change once differing audio renditions are tied to a selected video rendition (e.g. different audio renditions depending on the selected video rendition)?

[CTA-5005]: https://cdn.cta.tech/cta/media/media/resources/standards/pdfs/cta-5005-final.pdf
[videojs-contrib-quality-levels]: https://github.com/videojs/videojs-contrib-quality-levels
