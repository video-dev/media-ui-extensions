# media-ui-extensions
Extending the HTMLVideoElement API (`<video>`) to support advanced video player user-interface features

With the addition of [web components](https://developer.mozilla.org/en-US/docs/Web/Web_Components) to the browser, video player developers can create custom elements that mimic the video tag API, with goals of creating stand-in compatiblity with the video tag and compatibility across players. The video tag API is however lacking some important functions to support modern player UIs, including playback quality/resolution selection and awareness of ads. This repo is intended to capture requests and proposals for those functions.

## Submitting a proposal
Use `template.md` to frame your proposal and make a pull-request to add your proposal document to the `proposals` directory.

Inspired by [rust-lang/rfcs](https://github.com/rust-lang/rfcs) via [video-dev/hlsjs-rfcs](https://github.com/video-dev/hlsjs-rfcs).

## Submitting a request
Use the github issues to request and disucss a desired feature/function at a high level.

## Resources
* [MDN video element docs](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/video)
* [WHATWG spec](https://html.spec.whatwg.org/multipage/media.html#the-video-element)