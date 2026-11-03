# Task: Scrolling versus pressing and dragging

One finger scrolls, presses and drags: a pan that starts a drag on movement steals the scroll swipe, and a touch-down action inside a scroll view fires when the user meant to scroll. Make the drag or press out-wait the scroll gesture, or choose per touch which recognizer applies. Framework-provided scrolling views already handle touch scrolling, momentum catch and nested-scroller handoff.

## Detection patterns

- `NSPanGestureRecognizer` accepting `.direct` that starts a drag, reorder or marquee inside a scrollable view.
- Controls inside an `NSScrollView` that act on touch-down: `minimumPressDuration = 0`, `sendAction(on: .leftMouseDown)` / `sendActionOn:NSEventMaskLeftMouseDown`.
- `numberOfTouchesRequired` on a pan (deprecated in macOS 27.0), or hard-coded 1 or 2 on a pan that shares a view with scrolling or zooming.
- `scrollWheel(_:)` / `-scrollWheel:` as the only path for page swipes, row swipe actions or custom scrolling; momentum timers started from `scrollWheel:`; a `scrollWheel(_:)` override that drops events or skips `super` to disable scrolling (Pattern 8).
- `mouseDownCanMoveWindow` returning true / `YES`, or `isMovableByWindowBackground` (`movableByWindowBackground`), on a view whose content scrolls.
- `touchesBegan(with:)` tracking drags in a view that scrolls.

## Patterns

### Pattern 1: Drag after a hold, scroll on movement

**When to use:** a touch drag, reorder or lift starts inside scrolling content. A press with a deliberate duration and small `allowableMovement` fails if the finger moves first, so the scroll runs; a hold starts the drag.

```swift
// Before: touch drag starts on movement, so a swipe picks up the item
let drag = NSPanGestureRecognizer(target: self, action: #selector(handleDrag(_:)))
drag.allowedTouchTypes = .direct

// After: dwell decides. Movement before the hold means "scroll".
let drag = NSPressGestureRecognizer(target: self, action: #selector(handleDrag(_:)))
drag.minimumPressDuration = Self.touchDragHold          // 0.325 s
drag.allowableMovement = Self.touchHoldTolerance       // about 10 pt; about 12 pt for selected text
drag.allowedTouchTypes = .direct
drag.buttonMask = 0x0                    // touch only; the mouse keeps its existing drag path
addGestureRecognizer(drag)
```

`.began` is "drag picked up"; on `.cancelled` and `.failed` restore the pre-drag state. To find the picked-up item, hit-test the touch-down point recorded in `shouldReceive` (`input-kinds-task.md`, Pitfall 3).

A pan plus a press also replaces a hand-written "move or hold?" timer state machine: the pan starts the move; the press, with small `allowableMovement`, starts the drag after a hold and fails by itself when the pointer moves. Start the drag session at the press `.began` (`drag-sources-task.md`, Pattern 2); set `allowedTouchTypes = []` on the pan if touch must not move the surface.

### Pattern 2: Choose pan or press per touch

**When to use:** part of the surface scrolls and part does not (collection with a non-scrolling gutter, canvas that scrolls only when zoomed). Install a touch pan and a touch press; the delegate picks one per touch.

```swift
func gestureRecognizer(_ me: NSGestureRecognizer, shouldReceive touch: NSTouch) -> Bool {
    let location = touch.location(in: self)                 // valid here; the recognizer's own location is not
    if touchDownPoint == nil { touchDownPoint = location }  // the touch-down point; clear it in terminal states
    let canScroll = canScrollAncestor(at: location)         // your own check
    if me === touchDragPan   { return !canScroll }    // nothing to scroll: drag immediately
    if me === touchDragPress { return canScroll }     // scrollable: dwell first
    return true
}
```

`canScrollAncestor` is yours: per enclosing `NSScrollView`, is the document larger than the clip view on any axis? Also consider views that scroll without an `NSScrollView`.

### Pattern 3: Controls inside a scroll view yield to scrolling

**When to use:** a button or pop-up inside a scroll view fires on touch-down and scrolling the page triggers it. A continuously dragged control (slider or knob) keeps its touch-down response: use Pattern 4.

```swift
// Before: press.minimumPressDuration = 0 fires on touch-down, even when the user meant to scroll
// After: the press waits for the enclosing scroll gesture to fail (macOS 27.0+)
func gestureRecognizer(_ me: NSGestureRecognizer,
                       shouldRequireFailureOf other: NSGestureRecognizer) -> Bool {
    guard let scrollView = enclosingScrollView else { return false }
    return other === scrollView.scrollGestureForRelationships
}
```

Objective-C: `other == self.enclosingScrollView.scrollGestureForRelationships` inside `gestureRecognizer:shouldRequireFailureOfGestureRecognizer:`.

(macOS 27.0+) `NSScrollView.scrollGestureForRelationships` is the supported handle for failure and exclusion relationships against scrolling; do not search `gestureRecognizers` or match a class to find the scroll gesture. Before macOS 27, use a non-zero press duration only.

### Pattern 4: Let scrolling cancel a recognizer that already began

(macOS 27.0+) Set `isCancellableByScrollGesture = true` (Objective-C: property `cancellableByScrollGesture`, getter `isCancellableByScrollGesture`; default false) on a recognizer that begins at touch-down but must yield if the enclosing scroll view starts scrolling. It receives `.cancelled`; the cancel branch must undo what `.began` did (highlight, preview, selection).

```swift
highlightPress.isCancellableByScrollGesture = true
```

### Pattern 5: Scroll-view touch counts

(macOS 27.0+) `NSScrollView.minimumNumberOfTouchesForScrolling` (default 1) and `maximumNumberOfTouchesForScrolling` (default `NSIntegerMax`; 0 means exactly the minimum) set how many fingers scroll. `NSScrollView.isTouchScrollingEnabled` (default true) turns touch scrolling off for that scroll view.

Use when a canvas tool owns one finger and the document must still scroll:

```swift
savedMinTouches = scrollView.minimumNumberOfTouchesForScrolling   // remember the host's value
scrollView.minimumNumberOfTouchesForScrolling = toolIsActive ? 2 : 1
// On teardown or tool change, restore savedMinTouches.
```

### Pattern 6: Nested scrollers and page navigation

For a horizontal strip inside a vertical scroll view, a swipe that starts mostly off the strip's axis belongs to the outer scroller. Measure direction from the recorded touch-down point, not `translation(in:)` or `velocity(in:)`; a pan's `location(in:)` is valid in `gestureRecognizerShouldBegin(_:)`:

```swift
func gestureRecognizerShouldBegin(_ me: NSGestureRecognizer) -> Bool {
    guard me === stripPan, let start = touchDownPoint, outerCanScrollVertically else { return true }
    let now = me.location(in: self)
    let angle = atan2(abs(now.y - start.y), abs(now.x - start.x)) * 180 / .pi
    return angle < Self.axisDeferralAngle      // 60 degrees; steeper movement goes to the outer scroller
}
```

Returning false fails the strip's pan and the outer scroll continues. Alternative: the strip's pan requires the outer scroll gesture to fail (Pattern 3) only while the outer scroller can scroll orthogonally and is not pinned. Hand off at the strip's edge only once it reaches it, counting elastic overscroll as "not at the edge".

Page-style swipe navigation driven only by `scrollWheel(_:)` phases never reaches direct touch: add a touch-only pan beside it.

### Pattern 7: Catching a coasting scroll

A touch on moving content should stop the motion, not activate what is underneath. `NSScrollView` does this. For a custom scroller:
- Stop momentum in `gestureRecognizer(_:shouldReceive:)` (touch-down). A pan reports `.began` only after movement, so waiting for `.began` lets content drift under the finger.
- Make content recognizers (tap, select) wait on the scroll pan only while the touch caught deceleration, not unconditionally.
- Run momentum timers in `.common` run-loop modes so they keep firing while a finger is down.
- Ignore stale velocity: if the last movement is more than about 0.1 s old (tuning), treat release velocity as zero.

```swift
// in shouldReceive
if me === scrollPan {
    caughtDeceleration = isDecelerating
    momentumTimer?.invalidate()
}
let timer = Timer(timeInterval: 1.0 / 60, repeats: true) { [weak self] _ in self?.stepMomentum() }
RunLoop.main.add(timer, forMode: .common)
momentumTimer = timer
// lastMoveTime = event.timestamp of the last movement (system uptime)
let velocity = ProcessInfo.processInfo.systemUptime - lastMoveTime > 0.1 ? 0 : sender.velocity(in: self).y
```

### Pattern 8: `scrollWheel(_:)` overrides do not see touch

Under direct touch, `NSScrollView` scrolls with its own pan recognizer, not by receiving scroll-wheel events. Code that depends on `scrollWheel(_:)` therefore never runs for a touch scroll: an override on an `NSScrollView` subclass or an ancestor, a scroll-wheel event monitor, momentum started from `scrollWheel(_:)`. Overriding `scrollWheel(_:)` on a view that is not a scroll view does not give it touch scrolling either, because mouse emulation (`SKILL.md`) covers mouse, menu, magnify and rotate overrides, not scroll wheel.

- Keep the `scrollWheel(_:)` path for trackpad and mouse. For touch, add a touch-only pan beside it (`allowedTouchTypes = .direct`, `buttonMask = 0x0`), and share the code that moves the content.
- To react to touch scrolling inside an `NSScrollView`, observe the clip view's bounds changes rather than `scrollWheel(_:)`; to order your recognizers against its scrolling, use `scrollGestureForRelationships` (Pattern 3).
- A `scrollWheel(_:)` override that does not call `super` to disable or lock scrolling (while a modal tool is active, over a zoomed canvas, for a locked page) stops trackpad and mouse scrolling but not touch scrolling. Keep the override for those inputs and also set `isTouchScrollingEnabled = false` (macOS 27.0+, default true) for the same period, restoring it when scrolling is allowed again. To limit touch scrolling by finger count instead, see Pattern 5.

## Pitfalls

### Pitfall 1: Hard-coded or uncapped touch counts

**Symptom:** pinch becomes a pan; a zoomed-in document cannot be panned with one finger; two- and three-finger gestures drag an item.
**Cause:** an uncapped pan accepts any touch count at or above its minimum, so it can claim a pinch. Forcing two touches in every state makes zoomed-in content unpannable. A deprecated `numberOfTouchesRequired` left at 1 or 2 drags with the wrong fingers.
**Fix:** set both `minimumNumberOfTouches` and `maximumNumberOfTouches` on pans (`multitouch-task.md`, Pattern 1), set scroll-view counts per tool state (Pattern 5), restore the host's values on teardown.

### Pitfall 2: Window drag competes with scrolling

Touching near the window edge or in an inset scroll view moves the window, because of `isMovableByWindowBackground` or `mouseDownCanMoveWindow` returning true over scrolling content. Return false from `mouseDownCanMoveWindow` for views whose content can scroll, stretch or be dragged.

## Guidance numbers

Starting points; retune on a touch display.

| Constant | Starting value | What it controls |
|---|---|---|
| Touch drag hold | 0.325 s | Press duration before a drag starts, on a surface that scrolls |
| Hold movement tolerance (press `allowableMovement`) | about 10 pt | Movement before the hold completes. Past this the hold fails and scroll wins. Use about 12 pt for selected text or other small selectable items |
| Drag threshold after hold | about 10 pt | Movement after the hold completes before the drag visibly starts |
| Menu long press | 0.725 s | Press duration before the menu opens |
| Menu press movement tolerance | about 10 pt | Movement before the menu press fails |
| Axis deferral angle | 60 degrees | Initial movement more than this far off a nested scroller's axis is handed to the outer scroller |
| Stale velocity | ignore samples older than about 0.1 s | Release velocity for custom momentum |
