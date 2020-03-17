## CLS Shifted Element Surfacing

* [WICG tracking bug](https://github.com/WICG/layout-instability/issues/11)
* [Spec PR](https://github.com/WICG/layout-instability/pull/32/files)
* [Chromium tracking bug](http://crbug.com/1053510)

### Overview

Today it is difficult for web developers to understand the cause
of a high [CLS score](README.md) using the
[Layout Instability API](https://wicg.github.io/layout-instability/),
because nothing in the score or the `PerformanceEntry` connects
back to the specific DOM elements that were affected by the layout shift.

CLS Shifted Element Surfacing (SES) is an effort to surface
a subset of the shifted DOM elements in the
[`LayoutShift` interface](https://wicg.github.io/layout-instability/#sec-layout-shift).
This will improve the "actionability" of CLS for developers.
