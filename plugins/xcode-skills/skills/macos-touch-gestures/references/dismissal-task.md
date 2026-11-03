# Task: Outside-tap dismissal

A custom pop-up that dismisses on an outside click must dismiss on an outside tap too. An event monitor may not see touch-originated presses, so keep the mouse monitor and add a touch path (Pattern 2). A `.transient` `NSPopover`, a SwiftUI `.popover`, and a UIKit popover with default dismissal already dismiss on touch, so this applies only to overlays built without them.

## Detection patterns

- `addLocalMonitorForEvents(matching:` / `addGlobalMonitorForEvents(matching:` with `.leftMouseDown`, `.rightMouseDown`, `.otherMouseDown` or `.leftMouseUp`; Objective-C `+addLocalMonitorForEventsMatchingMask:handler:`, `+addGlobalMonitorForEventsMatchingMask:handler:` with `NSEventMaskLeftMouseDown`.
- A scrim that dismisses in `mouseDown(with:)`; custom pop-up windows (`NSWindow` or `NSPanel` with `orderOut` on outside click); `resignKey` dismissal.
- `NSPopover` left at the default or `.applicationDefined` `behavior`.

## Patterns

### Pattern 1: Use `NSPopover` when it fits

Replace the monitor with an `NSPopover` and `popover.behavior = .transient`; the default, `.applicationDefined`, never dismisses on its own. If the anchor also toggles the popover, see Pitfall 1. `NSPopover` is not always viable: when the visuals, animation, or behavior must differ, keep the custom overlay and use Pattern 2.

### Pattern 2: Keep the monitor, add an observe-only touch recognizer

**When to use.** A custom overlay must stay (a dismissable panel, a scrim, an inline editor, a custom `NSWindow` pop-up). Never use this for an `NSPopover`: it dismisses itself according to its `behavior`. Add a touch-only recognizer on the container that borders the overlay (the window’s content view or a full-size backing view). It observes; it never blocks or delays the touch reaching whatever is tapped.

```swift
final class OutsideTapObserver: NSPressGestureRecognizer {
    override init(target: Any?, action: Selector?) {
        super.init(target: target, action: action)
        minimumPressDuration = 0
        buttonMask = 0x0                      // mouse keeps its monitor
        allowedTouchTypes = .direct
        delaysPrimaryMouseButtonEvents = false
    }
    required init?(coder: NSCoder) {
        super.init(coder: coder)
    }

    override func canPrevent(_ other: NSGestureRecognizer) -> Bool {
        false
    }

    override func canBePrevented(by other: NSGestureRecognizer) -> Bool {
        false
    }
}
```

- The two overrides keep the observer from blocking other recognizers or being blocked by them. If a view beneath still behaves differently, allow simultaneity for this recognizer only.
- The press reports the touch-down point at `.began`, so record it there; decide on `.ended`.
- Dismiss the overlay from the recognizer action, not from the callback that decides recognition (`recognizer-relationships-task.md`, Pitfall 5).

```swift
@objc private func outsideTap(_ sender: NSPressGestureRecognizer) {
    let location = sender.location(in: containerView)
    switch sender.state {
    case .began:
        startPoint = location
        startTime = ProcessInfo.processInfo.systemUptime
        exceededForgiveness = false
    case .changed where hypot(location.x - startPoint.x, location.y - startPoint.y) > Self.tapForgiveness:
        exceededForgiveness = true                // stays set if the finger moves back
    // tapDuration: the longest press that still counts as a tap
    case .ended where !exceededForgiveness && ProcessInfo.processInfo.systemUptime - startTime <= Self.tapDuration:
        dismissIfOutside(windowPoint: containerView.convert(location, to: nil))
    default: break                                // a scroll or long press must not dismiss; .cancelled, .failed: do nothing
    }
}
```

Share the decision between mouse and touch by passing a window-coordinate point into one function, not an `NSEvent`; the monitor passes `event.locationInWindow`. Tap forgiveness: about 10 pt of movement. Rebuild the recognizer when the container is reparented (for example a view moving to another window).

Install the touch observer and any event monitor after the overlay is on screen, and remove them on dismiss and in `deinit`.

## Pitfalls

### Pitfall 1: Dismiss then immediately reopen

**Symptom:** tapping the control that opens a popover closes it, then it reopens.
**Cause:** a `.transient` popover swallows the click or tap that lands on its positioning view, but a control outside that view still receives the one that closes the popover and calls `show` again. A `.semitransient` popover lets the click or tap through to the control, so the control’s action runs while the popover is open.
**Fix:** in the opener’s action, check `popover.isShown` and call `performClose(_:)`; do not call `show` unconditionally. Test repeated taps.

### Pitfall 2: A new recognizer holds back the first mouse event of its descendant views

A click on a button inside the observer's view needs two attempts with a mouse after adding the observer: set `delaysPrimaryMouseButtonEvents = false` explicitly (defaults differ by class; `recognizer-relationships-task.md`, Pattern 3).

### Pitfall 3: Dismissal bookkeeping that outlives its event

**Symptom:** the window an item opens closes at once, or the next presentation is ignored or closed.
**Cause:** the opening press can report a focus loss that dismisses the window it opened (for example a press on a button that opens a panel); a flag or timer set for the “next” event outlives it.
**Fix:** clear it as in `recognizer-relationships-task.md` (Pattern 5, last bullet); dismiss on press end rather than press down.

## SwiftUI

Custom overlay dismissal on outside tap has no SwiftUI recipe: use `.popover`, which dismisses for every input, and leave existing overlay code. A `Color.clear` scrim whose tap never fires: `hit-testing-task.md`, SwiftUI Pattern 1.

## Mac Catalyst and UIKit

### Catalyst Pattern 1: Keep a popover open while content is dragged out

A `UIPopoverPresentationController` closes when a drag leaves its content (an item dragged out of a picker) because the drag reads as an outside tap.

```swift
func popoverPresentationControllerShouldDismissPopover(_ popoverPresentationController: UIPopoverPresentationController) -> Bool {
    !isDraggingOutOfPopover     // set in the drag interaction’s lift and end callbacks
}
```

Set the delegate, and clear the flag on every drag end, cancel and failure.
