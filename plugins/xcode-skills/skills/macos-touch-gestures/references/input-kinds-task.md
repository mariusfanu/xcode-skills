# Task: Configure gesture recognizers per input kind

Read this file first for any new recognizer: the other references assume these rules.
## Detection patterns

Recognizer configuration (AppKit):
- Recognizers created with no `allowedTouchTypes` or `buttonMask` assignment, including `[[NSClickGestureRecognizer alloc]`, `[[NSPressGestureRecognizer alloc]`, `[[NSPanGestureRecognizer alloc]`.
- `NSTouchTypeDirect` (the enum) where `NSTouchTypeMaskDirect` (the mask) is required.
- One recognizer whose timing (`minimumPressDuration`, `allowableMovement`) compromises between mouse and touch.
- `gestureRecognizer:shouldAttemptToRecognizeWithEvent:` reading `event.locationInWindow` with no `gestureRecognizer:shouldReceiveTouch:` beside it.
- `[NSApp currentEvent]` and `[self.window currentEvent]` anywhere (Pattern 6); `[event clickCount]` in recognizer actions or delegates.

Touch-origin guessing:
- `lastInputWasTouch`, `isTouching`, `lastTouchTime` set in `touchesBeganWithEvent:`, `mouseDown:` or a passive recognizer; time-window checks (`timeIntervalSinceDate:`, `CACurrentMediaTime() - lastTouchTimestamp`).
- `NSScreen.touchCapabilities` used as a gate.
- State armed in `mouseMoved:` or `mouseEntered:` and consumed in `mouseUp:` (`hover-task.md` Pitfall 1).

SwiftUI: gestures without `inputKinds:` that should differ per input; state such as `touchedID` next to `onHover`.

Catalyst: `UITouch.type`, `.indirectPointer`, `gestureRecognizer:shouldReceiveTouch:` with a `UITouch`; a caret or focus ring that moves after a scroll touch (Catalyst Pitfall 1).

A tap that does not hit at all (SwiftUI `.clear` areas, tiny views) is `hit-testing-task.md`.

## Patterns

### Pattern 1: Set the input kind explicitly: touch-only and mouse-only recognizers

**When to use:** a recognizer is added next to an existing mouse path, or an existing recognizer must stay mouse-only.

- `allowedTouchTypes` (`NSTouch.TouchTypeMask`, macOS 10.12.2+) selects direct touch. On macOS 27 a recognizer created in code accepts direct touch by default; earlier systems default to none.
- `buttonMask` selects mouse buttons. It exists on `NSClickGestureRecognizer`, `NSPressGestureRecognizer` and `NSPanGestureRecognizer`, not on the `NSGestureRecognizer` base class or on magnification and rotation. Default `0x1` (primary); `0x2` is secondary. A trackpad click is a mouse click here.

Touch-only: `allowedTouchTypes = .direct`, `buttonMask = 0x0`. Mouse-only: `allowedTouchTypes = []`, `buttonMask = 0x1`:

```swift
let click = NSClickGestureRecognizer(target: self, action: #selector(handleClick(_:)))
click.allowedTouchTypes = []
click.buttonMask = 0x1

let hold = NSPressGestureRecognizer(target: self, action: #selector(handleHold(_:)))
hold.buttonMask = 0x0                 // no mouse buttons
hold.allowedTouchTypes = .direct      // finger on a display only
```

In Objective-C, `allowedTouchTypes` takes the mask `NSTouchTypeMaskDirect`, not the enum value `NSTouchTypeDirect`; the enum silently selects the wrong set.

### Pattern 2: Two recognizers instead of one compromise

**When to use:** mouse and touch need different durations, movement tolerances or actions (a mouse drag starts on movement; a touch drag starts after a hold so a swipe can scroll).

```swift
let mouseDrag = NSPanGestureRecognizer(target: self, action: #selector(drag(_:)))
mouseDrag.allowedTouchTypes = []
mouseDrag.buttonMask = 0x1

let touchDrag = NSPressGestureRecognizer(target: self, action: #selector(drag(_:)))
touchDrag.buttonMask = 0x0
touchDrag.allowedTouchTypes = .direct
touchDrag.minimumPressDuration = Self.touchDragHold    // 0.325 s
touchDrag.allowableMovement = Self.touchHoldTolerance       // about 10 pt
```

The shared action reads `state` and the recognizer, not `NSEvent`. Set `minimumPressDuration` explicitly on each recognizer.

### Pattern 3: Implement both delegate entry points

**When to use:** a delegate decides whether a recognizer takes part (start only over a link or selected text; pick pan versus press by what lies under the finger).

`gestureRecognizer(_:shouldAttemptToRecognizeWith:)` takes an `NSEvent` and is not called for direct touches. A gate that must apply to touch uses `gestureRecognizer(_:shouldReceive:)`, which takes an `NSTouch` (macOS 10.12.2+). Keep `shouldAttemptToRecognizeWith` for mouse and trackpad.

Before (the gate never runs for a finger):

```swift
func gestureRecognizer(_ me: NSGestureRecognizer,
                       shouldAttemptToRecognizeWith event: NSEvent) -> Bool {
    isInteractive(at: convert(event.locationInWindow, from: nil))
}
```

After (one decision function, two entry points, touch-down point recorded):

```swift
private var downPoint: NSPoint?

func gestureRecognizer(_ me: NSGestureRecognizer,
                       shouldAttemptToRecognizeWith event: NSEvent) -> Bool {
    let location = convert(event.locationInWindow, from: nil)    // mouse and trackpad
    guard isInteractive(at: location) else { return false }
    downPoint = location
    return true
}
func gestureRecognizer(_ me: NSGestureRecognizer, shouldReceive touch: NSTouch) -> Bool {
    let location = touch.location(in: self)                      // direct touch
    guard isInteractive(at: location) else { return false }
    downPoint = location
    return true
}
```

- Record the starting point in these two methods. A pan reports `.began` only after movement, so `location(in:) - translation(in:)` is not the touch-down point. An unmoved press reports the touch-down point at `.began`.
- To keep a recognizer without `buttonMask` (magnification, rotation) off the mouse path, return false for mouse events in `shouldAttemptToRecognizeWith`; touches are not events, so touch is unaffected.

### Pattern 4: Read counts, modifiers and positions from the recognizer

Click counts: `click-counting-task.md`. Positions: the recognizer's `location(in:)`, not `NSApp.currentEvent`. Modifiers: the recognizer's `modifierFlags` (macOS 26.0+) in the action; in `gestureRecognizer(_:shouldReceive:)` use `NSEvent.modifierFlags`. Replace each read of `NSApp.currentEvent` with the recognizer, event or touch you were handed (Pattern 6).

### Pattern 5: Adapt to touch without a "touch active" flag

**When to use:** code branches on "is a touch happening" (a `lastTouchTime` window, an `isTouch` flag) to pick sizes, delays or affordances (Principle 6). The recognizer that fired is the signal; pass it on:

```swift
enum InputKind { case touch, pointer }

@objc private func tapped(_ sender: NSClickGestureRecognizer) {
    guard sender.state == .ended else { return }
    // touchTap is a touch-only recognizer; the mouse path has its own recognizer
    select(via: sender === touchTap ? .touch : .pointer)
}
```

Public signals, most to least reliable:
- Which recognizer fired. A touch-only recognizer (Pattern 1) is itself the signal.
- `NSEvent.type == .directTouch` and `NSTouch.type == .direct` on an event or touch you were handed.

**Hardware hint.** `NSScreen.touchCapabilities` (macOS 27.0+) returns `NSScreen.TouchCapabilities`, an option set with `.multiTouch`; the empty set is `[]`. Use it only as a coarse UI hint (whether a settings pane shows touch-oriented affordances), never to decide whether to install recognizers or handle input: hardware can change. Do not restyle the UI because a window moved to a touch display.


### Pattern 6: Replace `NSApp.currentEvent` reads

**When to use.** Code reads `NSApp.currentEvent` or `window.currentEvent`. Replace each read with a value you were handed:

- Click counts, positions and modifiers: Pattern 4.
- Touch versus pointer: Pattern 5.
- A menu target: `clickedRow` (`context-menus-task.md`, Pattern 2).
- Code with no event or recognizer in scope (a timer, a notification handler): record the value in the recognizer action or event handler that starts the work, and pass it along.

## Pitfalls

### Pitfall 1: A touch-only recognizer never fires

**Cause:** one of: the enum `NSTouchTypeDirect` assigned instead of `NSTouchTypeMaskDirect`; the view cleared `NSView.allowedTouchTypes` (defaults to direct touch for apps linked on or after macOS 10.12.2); a delegate implementing only `shouldAttemptToRecognizeWith`, which is not called for touches.
**Fix:** check the causes in that order, then Pattern 3.

### Pitfall 2: One tap runs the action twice

Two recognizers reach the action. Fix: disjoint input sets (Pattern 1) or a failure requirement.

### Pitfall 3: The pan origin is a few points off

A drag on a thin handle misses it, or a marquee starts away from the finger, because the origin was computed as `location - translation`. Record the touch-down point in `shouldReceive` or `shouldAttemptToRecognizeWith` (Pattern 3).

## SwiftUI

### SwiftUI Pattern 1: Split a gesture by input kind (macOS 27.0+)

`GestureInputKinds` (macOS 27.0+) is an `OptionSet` with `.directTouch`, `.indirectTouch` (the trackpad), `.pointer`, `.pencil` and `.all`. Default `.all`, so existing code keeps its behavior.

Applicable API (all macOS 27.0+): `TapGesture(count:inputKinds:)`, `SpatialTapGesture(count:coordinateSpace:inputKinds:)`, `LongPressGesture(minimumDuration:maximumDistance:inputKinds:)`, `DragGesture(minimumDistance:coordinateSpace:inputKinds:)`, `MagnifyGesture(minimumScaleDelta:inputKinds:)`, `RotateGesture(minimumAngleDelta:inputKinds:)`, `onTapGesture(count:coordinateSpace:inputKinds:perform:)` (the closure takes a `CGPoint`) and `onLongPressGesture(minimumDuration:maximumDistance:inputKinds:perform:onPressingChanged:)`.

```swift
extension View {
    func rowTap(select: @escaping () -> Void,
                details: @escaping () -> Void) -> some View {
        self.onTapGesture(count: 1, inputKinds: [.pointer, .indirectTouch]) { _ in select() }
            .onTapGesture(count: 1, inputKinds: .directTouch) { _ in details() }
    }
}
```

### SwiftUI Pattern 2: Restrict a gesture to one input kind beside another

Gestures with disjoint `inputKinds` do not compete; add them as separate `.gesture` / `.simultaneousGesture` modifiers. For gestures accepting the same kind: `recognizer-relationships-task.md` SwiftUI Pattern 1.

```swift
struct Compose: View {
    var body: some View {
        Text("Row")
            .simultaneousGesture(DragGesture(minimumDistance: 4, inputKinds: .pointer))
    }
}
```

The drag is limited to the pointer so a finger swipe can scroll; give touch its own drag or hold gesture if it needs one.

### SwiftUI Pattern 3: Design the affordance around state

When the interaction should not depend on input kind, reveal actions from state (a selected row shows its actions) alongside the `onHover` path (`hover-task.md` SwiftUI Pattern 1). This is the only way to tell touch from pointer before macOS 27.0.

### SwiftUI Pattern 4: Keep a `Button`; read its pressed state

A `Button` already responds to touch, and its `ButtonStyle` configuration exposes `isPressed`. Drive the highlight from `isPressed`. Do not replace the `Button` with `onTapGesture` or `DragGesture`: that risks firing the action twice.

### SwiftUI Pitfall 1: A `.pencil` gesture does not fire for touch or trackpad

`.pencil` applies to pencil input only; Mac touch displays and trackpads report `.directTouch` and `.indirectTouch`.

### SwiftUI: no equivalent

- No public "touch screen present" signal; `NSScreen.touchCapabilities` is a hint only (Pattern 5). `allowedTouchTypes`, `buttonMask`, `shouldReceive` have no counterpart; the only per-input filter is `inputKinds`.

## Mac Catalyst and UIKit

### Catalyst Pattern 1: Set `allowedTouchTypes` per input

A default UIKit recognizer accepts every touch type the system delivers. `UIGestureRecognizer.allowedTouchTypes` is an array of `NSNumber` wrapping `UITouch.TouchType` raw values; a pointer-only recognizer uses `.indirectPointer` the same way (pointer-only selection drag: `drag-sources-task.md`, Catalyst Pattern 2).

```swift
tap.allowedTouchTypes = [NSNumber(value: UITouch.TouchType.direct.rawValue)]
```

### Catalyst Pattern 2: Branch on `UITouch.type` in the delegate

Replace a global "last input" flag with the touch handed to the delegate: in `gestureRecognizer(_:shouldReceive:)`, switch on `touch.type` (`.indirectPointer` versus `.direct`). Use the delegate only to decide whether the recognizer receives the touch. State that must change on a tap belongs in the recognizer's action, and focus-driven state in `becomeFirstResponder`.

### Catalyst Pitfall 1: A caret appears on a scroll touch

**Cause:** `becomeFirstResponder` in a UIKit text host runs on the touch.
**Fix:** defer `becomeFirstResponder` until the `UITapGestureRecognizer` action runs; test that a scroll does not place the caret.

### Catalyst: no equivalent

- `NSScreen.touchCapabilities`, `.directTouch` and the `shouldAttemptToRecognizeWith` split are AppKit only; UIKit has no "touch screen present" signal, so do not gate recognizers on hardware. For UIKit single/double tap use `recognizer-relationships-task.md` Catalyst Pattern 1.
