# Task: Replace mouse tracking with gesture recognizers

`mouseDown(with:)`, `mouseDragged(with:)`, `mouseUp(with:)` and `nextEvent(matching:)` loops have no cancel path unless they handle `mouseCancelled(with:)` (event type `.mouseCancelled`, macOS 26.0+), and under mouse emulation they compete with scroll, drag and long press. `NSClickGestureRecognizer`, `NSPressGestureRecognizer` and `NSPanGestureRecognizer` give mouse, trackpad and touch one code path plus `.cancelled` and `.failed`. Read `input-kinds-task.md` first if you create any recognizer.

## Detection patterns

Legacy mouse entry points on views, controls, view controllers and other responders in the chain (include empty and `super`-only overrides):

- Swift: `override func mouseDown(with`, `mouseDragged(with`, `mouseUp(with`, `rightMouseDown(with`, `rightMouseUp(with`, `otherMouseDown(with`, `mouseMoved(with`
- Objective-C: `- (void)mouseDown:`, `- (void)mouseDragged:`, `- (void)mouseUp:`, `- (void)rightMouseDown:`, `- (void)otherMouseDown:`
- Scroller and cell tracking: `trackKnob:`, `trackMouse:inRect:ofView:untilMouseUp:`, `startTrackingAt:`, `continueTracking`, `stopTracking`
- Nested loops and timers: `window?.nextEvent(matching:`, `NSApp.nextEvent(matching:`, `NSEvent.startPeriodicEvents`, `nextEventMatchingMask:`, `NSEventMaskLeftMouseDragged | NSEventMaskLeftMouseUp`, `startPeriodicEventsAfterDelay:`, `sendActionOn:`.
- Highlight-then-fire: subclasses that set a highlighted flag in `mouseDown` and call `sendAction` in `mouseUp` after a `bounds.contains` check.

`rightMouseDown` and `menu(for:)` also match menu and drag work: migrate the mouse tracking here first, then apply `context-menus-task.md` or `drag-sources-task.md`. Mouse handling that presents a context menu goes to `context-menus-task.md` instead. A window that moves while the user scrolls is `scroll-and-drag-task.md` Pitfall 2.

## Choose the migration path

Three paths replace mouse handling with recognizers. They differ in mouse behavior, back-deployment and how much code changes, so **review the choice with the user and agree on one before editing**: recommend a path, say why, and ask about anything you cannot see in the code.

| Path | Use when | What you do |
|---|---|---|
| A. Touch-only recognizers beside the responder methods | The app deploys before macOS 27, or the code is in a framework; mouse behavior must stay exactly as it is; or a recognizer cannot reproduce all of the mouse behavior | Keep the mouse overrides. Add recognizers with `buttonMask = 0x0` and `allowedTouchTypes = .direct` (Pattern 5) |
| B. One recognizer for every input | Mouse and touch behave the same: nothing depends on hover (`mouseMoved`, `mouseEntered`, `mouseExited`), pressure or modifier keys, no `acceptsFirstMouse` click-through needs reproducing, and no subclass or caller overrides your mouse methods | Delete the overrides. One recognizer serves mouse, trackpad and touch (Patterns 1 to 4) |
| C. Separate recognizers for mouse and touch | Mouse and touch need different behavior (for example a mouse drag that starts on movement and a touch drag that starts after a hold) and the overrides are going away | A mouse and trackpad recognizer and a touch recognizer that share one action (`input-kinds-task.md`, Pattern 2). |

Ask the user before choosing:
- If unknown: what is the deployment target, and is this code an app or a framework others link?
- Does any mouse behavior (modifier keys, hover, pressure, click-through on the first click) have to stay as it is?
- Should mouse and touch differ anywhere (touch-only long press, a larger touch hit area)?

If the answers are unknown, take path A: it leaves mouse behavior untouched and is the easiest to undo.

Path C example: a mouse drag starts on movement, a touch drag starts after a hold so a swipe can still scroll. The two inputs need different recognizer types, so one recognizer cannot serve both (`input-kinds-task.md`, Pattern 2, has the full version).

```swift
// Path C: different recognizers, one action
let mouseDrag = NSPanGestureRecognizer(target: self, action: #selector(drag(_:)))
mouseDrag.allowedTouchTypes = []              // mouse and trackpad: starts on movement
let touchDrag = NSPressGestureRecognizer(target: self, action: #selector(drag(_:)))
touchDrag.buttonMask = 0x0                      // touch only: starts after a hold
touchDrag.allowedTouchTypes = .direct
touchDrag.minimumPressDuration = 0.325        // seconds
```

## Before removing an override

- Removing an empty or `super`-only override inside a table or outline view, or under an ancestor click recognizer, can change click behavior. Delete it, then test single click, double click and a click on a disclosure control in the surrounding container (Pitfall 2).
- A view controller or other responder above the view that overrides the same mouse methods still receives the touch as mouse events after you remove the view's own. Migrate it too. When a touch is stolen, cancelled or mis-routed and the recognizers in the hierarchy do not explain it, search the whole target for `mouseDown:` and `mouseUp:` overrides, including view controllers of overlay and hosting views: each makes AppKit add emulation recognizers to its view that compete with yours and can cancel them.
- You cannot attach a recognizer to a view controller: add it to the controller's view, with the controller as target and delegate.
- Use one recognizer for all inputs only if there is no `mouseMoved`/`mouseEntered`/`mouseExited` dependence, no pressure or modifier-dependent click-through, no `acceptsFirstMouse` click-through you cannot reproduce, and no subclass or caller that overrides your mouse methods. Otherwise use Pattern 5.
- A nested event loop has no cancel path: convert it to a pan or press with explicit `.cancelled` handling. Inside a scroll view, apply `scroll-and-drag-task.md` after this migration.

## Patterns

Handle `.cancelled` and `.failed` as well as `.ended` in every action: a competing gesture can cancel yours at any time.

Snippets set `allowedTouchTypes = .direct` so the recognizer serves touch as well as mouse and trackpad; use `[]` to keep one mouse-only.

### Pattern 1: Discrete click

**When to use.** The handler reacts once, on release, with no pressed state. Use `NSClickGestureRecognizer`, act on `.ended`, and read `location(in:)` and `modifierFlags` (macOS 26.0+) from the recognizer instead of the `NSEvent`.

If descendants still use `mouseDown` (a nested button, a table inside a container), set `delaysPrimaryMouseButtonEvents = false` explicitly (`recognizer-relationships-task.md` Pattern 3).

### Pattern 2: Press and release (highlight on down, fire on up inside)

**When to use.** A custom view highlights while pressed and fires only if the pointer is inside on release. Use `NSPressGestureRecognizer` with `minimumPressDuration = 0`: `.began` is mouse down, `.ended` is mouse up. A click recognizer has no `.began`, so no pressed highlight.

```swift
// Before:
override func mouseDown(with event: NSEvent) {
    isHighlighted = true
}

override func mouseUp(with event: NSEvent) {
    isHighlighted = false
    if bounds.contains(convert(event.locationInWindow, from: nil)) {
        sendAction(action, to: target)
    }
}

// After (in init)
let press = NSPressGestureRecognizer(target: self, action: #selector(handlePress(_:)))
press.minimumPressDuration = 0
press.allowedTouchTypes = .direct
addGestureRecognizer(press)

@objc private func handlePress(_ sender: NSPressGestureRecognizer) {
    switch sender.state {
    case .began, .changed: isHighlighted = bounds.contains(sender.location(in: self))
    case .ended:
        isHighlighted = false
        if bounds.contains(sender.location(in: self)) {
            sendAction(action, to: target)
        }
    case .cancelled, .failed: isHighlighted = false
    default: break
    }
}
```

Re-test the location at `.ended`; a fast release can skip `.changed`. A zero-duration press can report `.began` only at release (`recognizer-relationships-task.md` Pattern 2). The control must be the hit-test leaf for its whole target (`hit-testing-task.md`).

### Pattern 3: Tracking loop or drag to a pan

**When to use.** The handler loops on drag and up events, or implements `mouseDragged`. Show the preview in `.began` and `.changed`, commit in `.ended`, restore in `.cancelled` and `.failed`.

```swift
// Before:
override func mouseDown(with event: NSEvent) {
    while let nextEvent = window?.nextEvent(matching: [.leftMouseDragged, .leftMouseUp]), nextEvent.type != .leftMouseUp {
        updatePreview(at: convert(nextEvent.locationInWindow, from: nil))
    }
}

// After:
let pan = NSPanGestureRecognizer(target: self, action: #selector(handlePan(_:)))
pan.minimumNumberOfTouches = 1   // macOS 27.0+
pan.maximumNumberOfTouches = 1   // macOS 27.0+
pan.allowedTouchTypes = .direct
addGestureRecognizer(pan)

@objc private func handlePan(_ sender: NSPanGestureRecognizer) {
    switch sender.state {
    case .began, .changed:
        updatePreview(at: sender.location(in: self))
    case .ended:
        commit(at: sender.location(in: self))
    case .cancelled, .failed:
        restorePreview()
    default:
        break
    }
}
```

- `minimumNumberOfTouches` and `maximumNumberOfTouches` are macOS 27.0+ (a maximum of 0 means exactly the minimum); `numberOfTouchesRequired` on pans is deprecated in 27.0. Both at 1 keeps a pinch or two-finger gesture from being claimed by this pan.
- A helper that reads `NSEvent.mouseLocation` or polls periodic events: while a recognizer drives it, convert `sender.location(in: nil)` with `window.convertPoint(toScreen:)` and read `r.modifierFlags` (macOS 26.0+); drop periodic-event timers. Global mouse state can be stale or emulated under touch.
- A pan begins only after the input moves, so a plain click never starts it. If the old code acted on a plain click, also add a click recognizer (Pattern 1) and let both coexist (`recognizer-relationships-task.md` Pattern 1).

### Pattern 4: Drag that must react at touch-down

**When to use.** Visual state must change at touch-down (a custom picker's marker jumps to the touch point, a slider handle grabs) and a pan's movement threshold delays the visual change.

```swift
let drag = NSPressGestureRecognizer(target: self, action: #selector(handleDrag(_:)))
drag.minimumPressDuration = 0
drag.allowableMovement = .greatestFiniteMagnitude
drag.allowedTouchTypes = .direct
addGestureRecognizer(drag)
// .began: pick up at sender.location(in: self); .changed: follow; .ended: commit; .cancelled/.failed: restore
```

A press fails when movement exceeds `allowableMovement`; an unbounded value keeps it alive for the whole drag. Inside a scroll view a handle that must grab at touch-down keeps this press and yields with `scroll-and-drag-task.md` Pattern 4; a drag that should yield to scrolling uses a press with a real duration (`scroll-and-drag-task.md` Pattern 1).

### Pattern 5: Touch-only recognizer beside the legacy mouse path

**When to use.** The recognizer cannot reproduce all mouse behavior (see "Before removing an override"), or a large mouse-driven view is stable and only needs touch.

```swift
let touchPan = NSPanGestureRecognizer(target: self, action: #selector(handleTouchPan(_:)))
touchPan.buttonMask = 0x0                 // ignore mouse and trackpad clicks
touchPan.allowedTouchTypes = .direct    // NSTouch.TouchTypeMask
addGestureRecognizer(touchPan)
```

Test that one touch runs one action: if both paths fire, move the action into the recognizer and delete the override rather than filtering events. A view that keeps its mouse override does not get touch-specific behavior (long-press menu, touch text selection); do not read `NSApp.currentEvent` there (`input-kinds-task.md` Pattern 5).

### Pattern 6: Click-and-hold buttons (`sendActionOn:` and periodic events)

**When to use.** A custom button sends its action on mouse down, mouse up and a periodic event (`sendActionOn:` with `NSEventMaskPeriodic`, `setPeriodicDelay:interval:`) and switches on `NSApp.currentEvent.type` to tell click from hold. Replace with one press recognizer for mouse and touch on a custom `NSView` button. For an `NSButton` or `NSControl` subclass do not add a recognizer (Principle 3): fire on tracking end or the primary action. The example shows a panel on press and a fuller panel on hold.

```swift
let press = NSPressGestureRecognizer(target: self, action: #selector(pressed(_:)))
press.minimumPressDuration = 0
press.allowableMovement = .greatestFiniteMagnitude   // drift must not fail the press
press.allowedTouchTypes = .direct                    // default buttonMask: the mouse uses it too
button.addGestureRecognizer(press)

@objc private func pressed(_ sender: NSPressGestureRecognizer) {
    switch sender.state {
    case .began:
        wasShowing = isShowing
        didHold = false
        if !wasShowing { setPanelShown(true) }            // show on down
        holdTimer?.invalidate()
        let timer = Timer(timeInterval: 1.0, repeats: false) { [weak self] _ in
            self?.didHold = true
            self?.setFullPanelShown(true)
        }
        RunLoop.current.add(timer, forMode: .common)          // fires while a menu or panel is tracking the press
        holdTimer = timer
    case .ended:
        holdTimer?.invalidate()
        if wasShowing && !didHold { setPanelShown(false) } // hide on release unless the hold fired
    case .cancelled, .failed: holdTimer?.invalidate()
    default: break
    }
}
```

The 1 s hold is guidance. If the button shows a window that dismisses on focus loss, the opening press can report focus loss on press-down (`dismissal-task.md`, Pitfall 3).

## Pitfalls

### Pitfall 1: First click is swallowed by a recognizer on a parent

After a double-click recognizer is added to a container, the enclosing table does not select on the first click, or a child `NSButton` stops acting. Fix: `delaysPrimaryMouseButtonEvents = false` (`recognizer-relationships-task.md` Pattern 3); test the child, the selection and the double click.

### Pitfall 2: Removing an empty `mouseDown` changes double-click behavior

**Symptom:** after deleting a seemingly dead `mouseDown(with:)`, a double action fires on the second click of a disclosure control, or a row stops responding.
**Cause:** the override took part in mouse event delivery for the surrounding container.
**Fix:** restore it and migrate the owning view first; delete it only after a recognizer or a test covers the surrounding click behavior.
