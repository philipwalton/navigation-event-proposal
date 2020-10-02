# Navigation Event Proposal

## Overview

Measuring SPA navigation timings is a challenging problem that the Chrome Speed Metrics team has been trying to solve for years. In addition, many people/teams at Google have raised issues with the current History API, e.g [here](https://groups.google.com/a/google.com/d/msg/web-standards-team/-MwEpEWnCIQ/hNRvKvAMCwAJ), and [here](https://groups.google.com/a/google.com/d/msg/web-sdk/dsldZ9_hJpM/H59whMnZBwAJ) (internal links, sorry), and there is exploratory work being done looking into how best to improve/replace that API.

In a perfect world, any improvements to the History API would also address (at least partially) the difficulties of SPA measurement.

This proposal tries to address some of the difficulties involved in SPA measurement, and could potentially be incorporated into a larger improvement to the History API.

## Problem

There are two primary challenges around measuring SPA navigation timing in Chrome today:

* It's not always clear what an SPA navigation is.
* It's not always clear when an SPA navigation begins and ends.

### Goals

* Spec a new API that will give browsers a clear indication as to when apps are performing SPA navigations (vs. just loading new content or just changing the URL for other reasons).
* Ensure this API has clear incentives for developers to use it for SPA navigation, as well as clear disincentives for using it for non-SPA navigations (to [prevent gaming](#can-this-api-be-gamed)).
* Ensure the API can be easily polyfilled so routers/frameworks can adopt it without requiring all browsers to implement.

### Non-goals

* Replace or supercede [existing efforts](https://docs.google.com/document/d/1ilfh6fpDwxrdrHOHRYllKaCcyeXrlM-0dHWVSnQoNpY/edit?usp=sharing) to get User Timing data into current frameworks for research purposes.
* Replace the existing History API or solve all of the longstanding issues it has.

## Proposal

The simplest way to address both of the problems outlined above is to scope the solution to just cases where a traditional navigation was initiated by a user:

* Clicks on `<a href>` elements
* Form submissions

In both of the above cases, a `navigation` event will be dispatched (or possibly a set of `navigationstart`, `navigationend`, and `navigationcancel` events) on the document, which can be intercepted by a developer to prevent a full navigation from occurring.

The `navigation` event will have a `respondWith()` method that takes a promise, which (when invoked) will trigger the usual loading indicator of the browser (e.g. a spinner in the tab) and once resolved will update the URL in the address bar.

The timing of the SPA navigation will be defined as the delta from the `event.timeStamp` to the first paint after the promise resolves (we may need to refine this to handle async rendering, common in React apps).

Here's some example code:

```js
addEventListener('navigation', (event) => {
  const destinationURL = new URL(event.url);
 
  if (destinationURL.origin === location.origin) {
    event.respondWith(asyncFunctionToLoadNewPageContent(event.url));
  }
});
```

### Why this solution?

* **Progressive enhancement:** by only allowing SPA navigations on otherwise real, user-initiated navigations, if a browser doesn't support the `navigation` event (or the developer chooses not to include a polyfill, or JavaScript is disabled) a regular navigation will take place.
* **Simpler for developers:** avoids the problem we have today where developers need to handle all possible OS-specific _open in a new tab_ actions (e.g. right/middle/CMD/CTRL clicks on links; such cases would not trigger a `navigation` event).
* **Hard to game: **A `navigation` event can only be triggered by user input via known navigation-initiating elements. This will make it hard/impossible for developers to programmatically initiate new SPA navigations just to inflate their LCP/FID stats.
* **Built-in timing:** `event.respondWith()` must be called sync (just like with fetch events) so you'll always have access to `event.timeStamp` + the duration of the promise.

### What are the incentives to use this API?

Currently most applications that implement SPAs will likely score **worse** on the Core Web Vitals metrics than if they'd built an MPA (traditional, multi-page application).

This is due to the caching benefits most sites get on all page loads in a session after the first one. In SPAs, however, all page loads after the initial page load are handled by the app, and are thus not tracked by Chrome as separate navigations.

The result is that SPAs will generally have a much higher percentage of cold-cache loads than MPAs. That combined with the fact that SPAs generally load more JavaScript in order to facilitate SPA navigation means their loads often take even longer than traditional MPAs.

If SPAs could score better on Core Web Vitals load metric by just implementing the `navigation` event, they'd have a strong incentive to do so.

### Can this API be gamed?

In theory, a developer could replace lots of buttons on their pages with `<a href>` as a means to artificially increase the percentage of "good" navigations that Google tracks.

While this is a real concern and worth worrying about, there are a few things we could do to discourage this practice that I believe would work:

- **Encourage analytics vendors to automatically track <code>navigation</code> events as pageviews.**
  If this happens (which I believe it would) then apps using most analytics vendors would have an incentive to keep their pageview stats meaningful, and thus gaming the metric wouldn't be worth it. \
_ **The <code>navigation</code> event comes with default browser behavior.**
  For example, the navigation event would require a URL change in the address bar as well as a loading indicator. This alone would likely make it undesirable for developers to use for anything that's clearly not a real navigation. \
- **Browsers could use heuristics to detect malicious behavior and turn SPA navigations into real navigations.**
  The <code>navigation</code> event is designed as a progressive enhancement—which means when used properly, any <code>navigation</code> event the browser chooses not to run should still work as a full navigation. If app developers know that Chrome may choose to fallback to full navigations in some cases, they'd be less likely to try to game the metric, since it could lead to a worse user experience.

### Alternative designs

In this proposal, the `navigation` event is only dispatched for a user-initiate navigation (as opposed to a script-initiated navigation).

An alternative design could be to dispatch the event in all cases, but add some sort of flag to the event object indicating whether or not it was user-initiated (possibly reusing the `isTrusted` flag).

That would solve most of the use cases in the proposal while still allowing navigations initiated by APIs like `location.assign` to work. It could even be the case that calls to `location.assign` inside of a user-initiated event handler could continue to be treated as user-initiated navigation, though I think that does increase the potential for abuse.

### Open Questions

* Should `navigation` events be dispatched for cross-origin navigations?
* Does a NavigationEvent need an `isBackForward` property?
* How will history state play with this API? (big question, must be resolved) 
* Should the navigation event be fired in cases where the user did not initiate the transition (e.g. location.assign, history.pushState, etc.)
* What about swipe (or other gestures) used to navigate that don't currently work with `<a href>` elements

### How to polyfill

The polyfill for this API is fairly straight-forward, and it would consist of the following pieces:

1. A [delegated](https://learn.jquery.com/events/event-delegation/) `click` and `submit` event listener for all `a[href]` and `form` elements.
1. A `popstate` event listener.
1. Both 1 and 2 above would synchronously dispatch a `navigation` event unless:
   1. The event's [default was prevented](https://developer.mozilla.org/en-US/docs/Web/API/Event/defaultPrevented).
   1. The event's `url` property matched the current URL.
   1. The event's target was a new tab/window
   1. The event was a `click` event and `event.which > 1` or the event's `metaKey`, `altKey`, `shiftKey`, `ctrlKey` property is set.    
1. The `navigation` event dispatched by the polyfill would be a regular event (of type `navigation`) that contains a `url` property and a `respondWith()` method. If the `respondWith()` method is called and the underlying event is a `click` or `submit` event:
   1. The event's `preventDefault()` method is called.
   1. `history.pushState()` is called with the new URL (and potentially a `state` object passed by the developer).
