- Feature Name: renditions_list
- Start Date: 2023-04-06
- PR: (leave this empty)

# Summary
[summary]: #summary

This proposal introduces an addition to existing APIs to be able to control the quality level for currently playing HTTP Adaptive Streaming (HAS) video.

# Motivation
[motivation]: #motivation

With the prevalence of segment media formats like HLS and DASH, playback uses Adaptive bitrate (ABR) algorithms to choose the reasonable quality for the user based on things like network conditions. However, users often want more control over the quality that of the stream that they are watching. For example, to force a high quality rendition for playback when watching a movie or a screen recording.

This proposal provides the mechanism to be able to see what are the available renditions associated with the media that is being played and change the current selection.

## Goals
- Introduce an API that returns available renditions and allows selecting which renditions are used.
- Take inspiration from existing `TrackList`s.
- Support streams that can be interoperable via CTA-WAVE's [CTA-5005][]'s specification.

## Non-Goals
- Provide a comprehensive support for all potential valid segmented streams with multi-period and discontinuity support.
- The API isn't intended to provide all potentially available information on the renditions.
- Provide a renditions list for AudioTracks

# Guide-level Explanation
[guide-level-explanation]: #guide-level-explanation

The main addition in this proposal is a `VideoRenditionList` which augments the [`VideoTrack`](https://html.spec.whatwg.org/multipage/media.html#videotrack) object API. This `VideoRenditionList` is a list of `VideoRendition`s which include the minimum set of information that's request by HAS formats. In addition, it includes a way to turn each individual rendition on and off.

For example, if I wanted to limit a video to only play back SD content, I can loop through the `VideoRenditionList`, and enable only the renditions that have a height less than 720.

```js
// assuming a video is already loaded
const video = document.querySelector('video');

const videoTrack = video.videoTracks[0];
const videoRenditions = videoTrack.renditions;

// only keep renditions enabled that are of height less than or requal to 720
Array.from(videoRenditions).forEach(rendition => {
    rendition.enabled = rendition.height <= 720;
});
```

To change the selected rendition, you can set the `selectedIndex` on the renditions list.
```js
// assuming a video is already loaded
const video = document.querySelector('video');

const videoTrack = video.videoTracks[0];
const videoRenditions = videoTrack.renditions;

// switch to the 3rd rendition
videoRenditions.selectedIndex = 2;
```

Changing the `selectedIndex` manually will trigger a `change` event on the renditions list. The ABR algorithm changing the selected rendition should also trigger a `change` event on the list object.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Below is WebIDL definitions for the changes and some explanation around it.

```ts
partial interface VideoTrack {
  readonly attribute VideoRenditionList renditions;
};
```

`VideoRenditionList` is similar to [`VideoTrackList`](https://html.spec.whatwg.org/multipage/media.html#videotracklist).
The main difference aside from naming is the addition of a read-write `selectedIndex` property.
This value will default to `-1` when no rendition is selected, such as before metadata has loaded.
Otherwise, it will reflect the index of the currently selected rendition.
Setting the value to a number will request a rendition change to that rendition.

When the rendition changes, either via manually setting the `selectedIndex`, or automatically via the ABR algorithm, a `change` event should be triggered.

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

Then each `VideoRendition` is includes some metadata on the rendition, like width, height, bitrate, and codec.
It also includes a boolean to determine whether this rendition is enabled or not.
This allowed the ABR algorithms to know whether they should be able to switch to this rendition or not.

If the currently active rendition is disabled, it should request a rendition change.

```ts
interface VideoRendition {
  readonly attribute DOMString id;
  readonly attribute unsigned long width;
  readonly attribute unsigned long height;
  readonly attribute unsigned long bitrate;
  readonly attribute unsigned long frameRate;
  readonly attribute unsigned long codec;
  attribute boolean enabled;
};
```

# Drawbacks
[drawbacks]: #drawbacks

The main drawbacks to the above proposal is that to implement this in userland makes it a lot harder since we're extending built-in pieces of HTML. However, since one of the goals of these proposals is to eventually land in the HTML spec, this is worth doing because the API would be more likely to stay the same if that happens.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Some of the APIs for this mentioned in the Prior Art section are either overly complex or too simple to allow the flexibility mentioned above.
For example, we want to be able to turn off individual renditions and have the underlying engine then only switch between these renditions.

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

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should VideoRendition and VideoRenditionList be generic Rendition and RenditionList to allow for usage with AudioTracks in the future?
- Is there anything here that would be affected and need to be changed once AudioRenditions are added? Specifically, it's possible to have different AudioRenditions depending on the selected VideoRendition.

[CTA-5005]: https://cdn.cta.tech/cta/media/media/resources/standards/pdfs/cta-5005-final.pdf
[videojs-contrib-quality-levels]: https://github.com/videojs/videojs-contrib-quality-levels
