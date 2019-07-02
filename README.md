## Explainer: Layout Instability Metric

### Overview

Many websites suffer from "layout instability" - DOM elements shifting around
due to content loading asynchronously.

We propose a way for the user agent to measure layout instability during a
browsing session to compute "layout shift scores", which would be exposed by a
new interface in the
[Performance API](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API).

### Layout Shift Score

Each animation frame (a.k.a.
"[rendering update](https://html.spec.whatwg.org/#update-the-rendering)")
computes a *layout shift* (LS) score approximating the severity of visible
layout instability in the document during that frame.  An animation frame with
no layout instability has an LS score of 0.  Higher LS scores correspond to
greater instability.

### Shifting Elements

A "shifting element" is one whose visual representation starts in a
significantly different location than it did in the previous animation frame.
"Starts" refers here to the element's
[flow-relative](https://www.w3.org/TR/css-writing-modes-4/#flow-relative) offset
in the document.

The visual representation of a block-level element is its
[border box](https://www.w3.org/TR/css-box-3/#border-box).  The visual
representation of an inline element is the geometric union of its
[box fragments](https://www.w3.org/TR/css-break-3/#box-fragment), the first of
which determines its starting location.

Note that:

* An element that changes in size (for example, by having children appended),
  but starts at the same offset, is not a shifting element.

* An element whose start location changes two or more times during the same
  animation frame (for example, from
  [forced synchronous layouts](https://developers.google.com/web/fundamentals/performance/rendering/avoid-large-complex-layouts-and-layout-thrashing#avoid_forced_synchronous_layouts)),
  but is ultimately painted at the same location as the previous frame, is not
  a shifting element.

* An element whose start location changes by less than 3 CSS pixels is not a
  shifting element.

### Impact Region

The "impact region" of an animation frame is the geometric union of the
previous-frame and current-frame visual representations, intersected with the
viewport, of all shifting elements in that frame.

The "impact fraction" of an animation frame is the fraction of the viewport that
is occupied by the impact region.

![Illustration of a shifting element on a device, with the impact region
highlighted](https://i.imgur.com/XN7xdKF.png)

*Example: An element which occupies half the viewport shifts by a distance equal
to half its height.  The impact fraction for this animation frame is 0.75.*

In general, the layout shift (LS) score is equal to the impact fraction.

TODO(skobes): describe distance fraction proposal

However, if the user has generated certain
[UI events](https://www.w3.org/TR/uievents/) within the past 500 ms, the LS
score is 0.  This allows the page to modify its layout in response to the
event.  The event types that trigger this exception include taps, key presses,
and mouse clicks, but not mousemove or events that cause scrolling.

The user agent may trade off precision for efficiency in the computation of
LS scores.  It is intended that the LS score have a correspondence to the
perceptual severity of the instability, but not that all user agents produce
exactly the same LS scores for a given page.

### Cumulative Scores

The user agent can compute a *document cumulative layout shift* (DCLS) score as
the sum of the document's LS scores for each animation frame that has occurred
during the browsing session.  The DCLS score is 0 when the document begins
loading, and grows whenever layout instability occurs.  The DCLS score does not
account for layout instability inside descendant browsing contexts, such as
those created by `<iframe>` elements.

The user agent can compute a *cumulative layout shift* (CLS) score for a
[top-level browsing context](https://html.spec.whatwg.org/multipage/browsers.html#top-level-browsing-context)
by summing the LS scores of the top-level browsing context to the weighted LS
scores of its descendant browsing contexts.  In performing this aggregation,
the LS score of a layout shift in an `<iframe>` should be weighted by the
fraction of the top-level viewport the `<iframe>` occupies at the time the
layout shift occurs.

### Performance API

Animation frames with non-zero layout shift (LS) scores will notify a registered
[PerformanceObserver](https://w3c.github.io/performance-timeline/#the-performanceobserver-interface).
The observer's callback receives one or more `LayoutShift` entries:

```idl
interface LayoutShift : PerformanceEntry {
    readonly attribute double value;
};
```

The entry's `value` attribute is the LS score.  Its
[entryType](https://w3c.github.io/performance-timeline/#dom-performanceentry-entrytype)
attribute is `"layout-shift"`.

The developer can compute the DCLS score by summing the LS scores:

```javascript
addEventListener("load", () => {
  new PerformanceObserver((list) => {
      list.getEntries().forEach((entry) => { DCLS += entry.value; });
  }).observe({type: "layout-shift", buffered: true});
});
```

The observer is only notified of layout instability that occurs after it is
registered.  The `getEntriesByType` method retrieves buffered entries for layout
shifts that has already occurred.  These entries are only buffered until the
`load` event fires.  Layout instability that occurs after the `load` event can
only be seen by the `PerformanceObserver`.

A "final" DCLS score for the user's session can be reported by listening to the
[visibilitychange event](https://developers.google.com/web/updates/2018/07/page-lifecycle-api#event-visibilitychange),
and using the value of `DCLS` at that time.

A [demo page](https://output.jsbin.com/gemufev) illustrating the use of this
code can be viewed in Chrome 76+ with the command-line flag
`--enable-blink-features=LayoutInstabilityAPI`, or in Chrome 73-75 with the
command-line flag `--enable-blink-features=LayoutJankAPI`.

### Privacy and Security

Layout instability bears an indirect relationship to resource timing, as slow
resources could cause intermediate layouts that would not otherwise be
performed.  Resource timing information can be used by malicious websites for
statistical fingerprinting.

The layout instability API only reports layout shifts in the current browsing
context (frame).  It does not directly provide the CLS score incorporating
subframes.  Developers can implement such aggregation manually, but browsing
contexts with different
[origins](https://html.spec.whatwg.org/multipage/origin.html#concept-origin)
would need to cooperate to share LS scores.

### Terminology

The "layout instability metric" was previously called the "layout stability
metric".

"Layout instability" and "layout shift" were previously referred to as
"layout jank".  The impact region was previously referred to as the "jank
region".  The LS score was previously referred to as the "jank fraction".

The DCLS score and CLS score were previously referred to as
"(aggregate) jank score".

The LayoutShift interface was previously implemented as PerformanceLayoutJank.
Its "value" attribute was previously named "fraction", and its entryType was
previously "layoutJank".

The layout instability API is an extension of the web performance API, but it is
not related to the speed or timing of layout computation.

### Links

* [Chrome Feature Dashboard Entry](https://www.chromestatus.com/feature/5110682739539968)
* [Blink Intent to Implement](https://groups.google.com/a/chromium.org/d/msg/blink-dev/jF1-M8KWAMU/ubGV4Fx2BgAJ)
* [Chromium Tracking Bug](https://crbug.com/581518)
