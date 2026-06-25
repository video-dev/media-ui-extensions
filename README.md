# media-ui-extensions
Extending the HTMLVideoElement API (`<video>`) to support advanced video player user-interface features

With the addition of [Web Components](https://developer.mozilla.org/en-US/docs/Web/Web_Components) to the browser, video player developers can create custom elements that mimic the video tag API, with goals of creating stand-in compatibility with the video tag and compatibility across players. The video tag API is however lacking some important functions to support modern player UIs, including playback quality/resolution selection and awareness of ads. This repo is intended to capture requests and proposals for those functions.

## Media Extensions Interface

The following IDL summarizes the additions to `HTMLMediaElement` defined by the proposals accepted so far. See each linked proposal for full definitions, possible values, and recommended computations.

```ts
partial interface HTMLMediaElement {
  // Stream Type (proposals/0010-stream-type.md)
  // "unknown" | "live" | "on-demand"
  readonly attribute DOMString streamType;
  // dispatches "streamtypechange" when the stream type is determined

  // Live Edge (proposals/0007-live-edge.md)
  // Media presentation time at which the Live Edge Window begins.
  // The element is at the live edge when `currentTime >= liveEdgeStart`.
  // NaN when unknown/inapplicable (e.g. on-demand streams).
  readonly attribute double liveEdgeStart;
  // `seekable.end(seekable.length - 1)` is constrained to model the Seekable Live Edge.

  // Renditions List (proposals/0011-renditions-list.md)
  readonly attribute VideoRenditionList videoRenditions;
  readonly attribute AudioRenditionList audioRenditions;
};

// Renditions List (proposals/0011-renditions-list.md)
interface VideoRenditionList : EventTarget {
  readonly attribute unsigned long length;
  getter VideoRendition (unsigned long index);
  VideoRendition? getRenditionById(DOMString id);
  attribute long selectedIndex;

  attribute EventHandler onchange;
  attribute EventHandler onaddrendition;
  attribute EventHandler onremoverendition;
};

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

## Goals for Proposals
We want these proposals to be something that we can eventually propose to the Media WG or WhatWG as additional features to the video element and anything related. In the meantime, proposals accepted to media-ui-extensions can be used as a specification to keep things interoperable between implementers.

## Before submitting a proposal
If you have a new idea, it might be worth creating an issue to discuss it a bit first before submitting a proposal to make sure that it's something could be a good fit and won't be less likely to be rejected later on.

## Submitting a proposal
- Fork the repo
- Copy `0000-template.md` to `proposals/0000-your-feature.md`. Don't change the number yet as that will be updated when the PR is created.
- Fill out your proposal with as much detail as possible.
- Submit a pull request. Once the PR is created, a new commit can be made to update the filename `0000-` prefix and the header of the file to point to the new PR number.
- Folks should now review the proposal and suggest changes. While in the PR, it's expected that many changes may be made. Ideally, please don't rebase commits in the branch as it'll make reviewing only new changes easier.
- At some point, a committer should issue a Call for Consensus (CfC). This CfC will include whether this proposal should be accepted, rejected, or postponed.
- The CfC will last a minimum of 2 weeks to make sure that all collaborators gets a chance to review. If all relevant parties have explicitly finished reviewing before 2 weeks, the CfC may end early.
- If there is enough consensus, the CfC will close successfully. If there is no consensus, the proposal could be rejected or go back to be re-edited.

## Implementing a proposal
Once a proposal has been accepted, it can start to be implemented. If during implementation, major issues are found, the existing proposal shouldn't be updated but a new proposal should be issued. The old proposal should be updated to link to the new proposal.

## Resources
- [MDN video element docs](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/video)
- [WHATWG spec](https://html.spec.whatwg.org/multipage/media.html#the-video-element)
- [Media Working Group](https://github.com/w3c/media-wg/)

## Inspiration
- [rust-lang/rfcs](https://github.com/rust-lang/rfcs) via [video-dev/hlsjs-rfcs](https://github.com/video-dev/hlsjs-rfcs).
- [WebKit Explainers](https://github.com/WebKit/explainers)

