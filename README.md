# APZ Notes

This is a place I write down notes about APZ. Mostly for myself, but published in case it's useful for others.

This is not documentation. Anything here may well be out of date a minute after I've written it.

## 2020-01-17

** APZ Architecture Topics **

This is a rough outline of the topics I cover in my APZ architecture peering sessions when new engineers join the Graphics team.

* the main processes of relevance to APZ (parent, compositor/GPU, content)
* the rendering pipeline in the content process
  * differences between WebRender and non-WebRender
  * mention that there is an instance of this pipeline in the parent process as well for browser chrome
  * mention that this happens on the main thread and therefore can be blocked on JS
* the rendering pipeline in the compositor process
  * differences between WebRender and non-WebRender
  * the APZ component in that process
* transactions from the content process to the compositor process
* input events arriving at the parent process
  * sending them to the compositor process to undergo compositor hit testing
  * dispatching them to the correct content process (or to the content in the parent process)
    * implied requirement on compositor hit testing: identify the correct content process to handle the event
* display-list based hit testing in content
* the APZ scrolling and zooming fast-path
  * implied requirement on compositor hit testing: identify the correct scrollable element to scroll
* displayports
  * checkerboarding
  * displayport heuristics
* the dispatch-to-content mechanism, and the 3 problems it solves
  * inactive scrollframes
  * irregular shapes
  * preventDefault()
    * passive event listeners
    * content response timeout, correctness vs. responsiveness tradeoffs
* scroll offset synchronization
* visual vs. layout viewports, mobile viewport sizing

## 2019-09-30

**APZ has two different codepaths where it computes a transform for hit-testing purposes**

Three, actually, if you count WebRender:

1. During APZ hit testing, to decide which hit testing tree node to target
   * For non-WebRender, this walks the hit testing tree and unapplies each node's `APZCTreeManager::ComputeTransformForNode()`.
   * For WebRender, this presumably does something similar inside WebRender.
1. When converting the screen coordinates into APZC coordinates, it uses `GetScreenToApzcTransform()`. (These then get converted further into Gecko coordinates using `GetApzcToGeckoTransform()`.)

In principle, `GetScreenToApzcTranform()` should return the same transform as multiplying together the `ComputeTransformForNode()` for every node from the APZC to the root, and implementing it that way should have no effect on correctness. We do not implement it that way, in part for historical reasons (`GetScreenToApzcTranform()` predates `ComputeTransformForNode()`) and in part for performance reasons (`GetScreenToApzcTranform()` uses the "ancestor transforms" which are segments of the multiplication that only change during hit testing tree updates and are pre-computed and cached during such updates).

We should be careful to make sure these functions remain in sync. Already today, there are some subtle differences:

* The ancestor transforms used in `GetScreenToApzcTranform()` have special handling for perspective transforms, which `ComputeTransformForNode()` lacks.
* `ComputeTransformForNode()` handles async transforms on scrollbar layers.

The second one is fine (a scrollbar layer is never on a path from the APZC to the root), but the first is a subtle bug that may have a user-noticeable effect on some pages with perspective transforms.

There may be existing inconsistencies with the WebRender codepath as well, which I haven't investigated.

## 2019-09-16

**Under what circumstances does `APZCTreeManager::ReceiveInputEvent()` return its various return values?**

* `eIgnore` is returned when:
  * WebReplay is active
  * the event has no target APZC
  * the event has a non-positive W-coordinate after untransforming it from screen coordinates into Gecko coordinates
  * for touch events:
    * we receive a non-touchstart touch event when no touch blocks are active (this shouldn't happen)
    * the event's target APZC cannot consume pointer events (due to e.g. `touch-action`)
  * for pan gesture events:
    * if the pan gesture is horizontal, didn't cause scrolling, and may trigger a swipe gesture
    * the event (considering its modifiers) is configured (via default or modified prefs) to perform an action other than scroll, horizontalized scroll, or pinch-zoom
  * for mouse events, if the mouse event is not part of a drag
  * for wheel events:
    * the event has an unrecognized delta mode
    * the event (considering its modifiers) is configured (via default or modified prefs) to perform an action other than scroll, horizontalized scroll, or pinch-zoom
  * for keyboard events:
    * if keyboard APZ is disabled
    * if caret browsing is enabled
    * if the key is not mapped to a scrolling action
    * if the key event needs to be dispatched to content (e.g. because it can change focus)
    * if the main thread hasn't given us a valid scroll target for keyboard events via the focus state
  * we don't recognize the type of the event (i.e. it's not a touch, mouse, wheel, pan gesture, or keyboard event)
* `eConsumeNoDefault` is returned when:
  * on Fennec, if the entire scroll amount caused by the event went into moving the dynamic toolbar
  * for touch events:
    * if we are panned into overscroll and a second finger goes down (and subsequent movements of that second finger)
    * the touch event is dropped due to being in a fast fling
    * the touch event is dropped due to it being a touch-move within the "slop" radius of the touch-start
  * for keyboard events, if we don't allow keyboard APZ with passive listeners
* otherwise `eConsumeDoDefault` is returned

Additionally, we may return `eIgnore` or `eConsumeNoDefault` in some cases when `APZCTreeManager` is given a pinch gesture event or tap gesture event directly, but this only happens in test code.

**What do the callers of `APZCTreeManager::ReceiveInputEvent()` do with its return value?**

* if it returns `eConsumeNoDefault`:
  * for most platforms / event types, the event is not dispatched to Gecko
  * sometimes, the treatment is the same as if the event was dispatched to Gecko and it prevent-defaulted it, which can have a variety of effects
    * e.g. on some platforms, some events can trigger window manager actions like minimization if they're not consumed by Gecko
* if it returns `eIgnore`:
  * for Mac pan gestures, we track the event as potentially triggering a swipe
* if it returns `eConsumeNoDefault` or `eIgnore`:
  * we avoid sending a `pointercancel` event in cases where we otherwise might
