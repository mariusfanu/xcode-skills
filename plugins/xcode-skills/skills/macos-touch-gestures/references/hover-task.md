# Task: Hover-only UI

Controls that appear on `mouseEntered:` or `onHover` never appear for a touch. Add a touch path next to the hover path; a revealed control also needs a touch-sized hit area (`hit-testing-task.md`).

## Detection patterns

- `NSTrackingArea(`, `addTrackingArea(`, `updateTrackingAreas`, `- (void)mouseEntered:`, `- (void)mouseExited:`, `- (void)mouseMoved:`, `acceptsMouseMovedEvents`.
- Enter and exit handlers that toggle `isHidden`, `alphaValue` or `isEnabled`, or flags (`isHovered`, `isRolledOver`, `mouseInside`) that gate drawing; hover that is the only route to an action (close, delete, more-actions).
- SwiftUI `.onContinuousHover`, `hoveredID`; Catalyst hover flags that gate visibility.

Leave alone hover that only decorates (highlight, cursor, tooltip) and mouse-only utility surfaces.

If an item is selected only by hover (a `mouseEntered:` that sets the selection), set the selection in the action path as well: a touch does not reliably send `mouseEntered:` before the tap.

## Patterns

### Pattern 1: Give a hover-only control a touch path

Approaches A to E: pick one per control, in this order of preference; give every hover-only action a touch route. The choice between approaches is a design decision: present the options to the user and agree on one before applying it.

#### Approach A: also show it when selected

```swift
var showsRollover: Bool { isHovered }                 // before
var showsRollover: Bool { isHovered || isSelected }   // after
```

#### Approach B: always show it

When the control is cheap and not noisy, remove the hover gate.

#### Approach C: add it to a menu

When the action is secondary, add it to the row view's `menu` (`context-menus-task.md`); SwiftUI: SwiftUI Pattern 1.

#### Approach D: first touch reveals, second touch activates

**When to use.** The control must stay hidden until needed. The first touch only reveals it; it does not select the row, toggle disclosure or activate anything. Hide after a short idle (start near 3 s), when another touch lands elsewhere, or when the pointer is used again. Keep touch-revealed state separate from hover state. SwiftUI: SwiftUI Pattern 2.

```swift
let reveal = NSClickGestureRecognizer(target: self, action: #selector(revealTapped(_:)))
reveal.buttonMask = 0x0                 // touch only; mouse hover path is unchanged
reveal.allowedTouchTypes = .direct
reveal.delaysPrimaryMouseButtonEvents = false
addGestureRecognizer(reveal)
```

A touch-revealed control has no tracking area to hide it. Hide it at the next mouse-moved event anywhere (a local and a global monitor, installed only while revealed and removed by `hideReveal` with `NSEvent.removeMonitor(_:)`), at the next touch elsewhere, or after the idle time.

```swift
moveMonitors = [
    NSEvent.addLocalMonitorForEvents(matching: .mouseMoved) { [weak self] event in
        self?.hideReveal()
        return event
    },
    NSEvent.addGlobalMonitorForEvents(matching: .mouseMoved) { [weak self] _ in
        self?.hideReveal()
    },
].compactMap { $0 }
```

- If the tap landed on the already revealed control, let it handle the tap instead of toggling the reveal off.
- If the reveal tap also selects the row or is part of a double-tap, make the reveal wait for the double-click recognizer with `gestureRecognizer(_:shouldRequireFailureOf:)`, because report order is not guaranteed (`click-counting-task.md`, Pattern 1).

#### Approach E: press-and-drag highlighting

**When to use.** Lists that highlight or select under the pointer (suggestions, pickers) should do the same under a finger. Use a touch-only press with `minimumPressDuration = 0`.

```swift
@objc private func handleTouchSelect(_ sender: NSPressGestureRecognizer) {
    let row = sender.location(in: tableView)
    switch sender.state {
    case .began, .changed: highlight(rowAt: row)
    case .ended: commit(rowAt: row)              // re-hit-test: a fast release can skip .changed
    case .cancelled, .failed: clearHighlight()
    default: break
    }
}
```

Do not clear the selection in a `default:` branch: `.changed` fires on small finger jitter. Read `recognizer-relationships-task.md` (Pattern 2) before adding a zero-duration press next to a drag or context-menu press.

### Pattern 2: Do not rely on `mouseExited:` timing

**When to use.** A hover control hides in `mouseExited:` and must stay usable for a touch. Show the control for the selected row (Approach A), always (Approach B) or reveal it on a recognized touch (Approach D).

- After a touch there is no pointer to leave: also clear exit-only state (including a hover preview) on the next touch elsewhere, on selection change or on a timer.

## Pitfalls

### Pitfall 1: State armed only by mouse movement

**Symptom:** a custom pop-up or segmented control needs a second tap with touch, or a segmented-style control reselects the old segment after a release outside every segment.
**Cause:** the control sets tracking state in `mouseMoved:` or `mouseEntered:` and acts in `mouseUp:`; a touch has no move before the up.
**Fix:** arm the state at press `.began` (or `mouseDown:`), and on a release outside every target choose the nearest one.

## SwiftUI

### SwiftUI Pattern 1: Hover-revealed action also reachable by menu or selection

Keep `onHover` for the pointer and reach the action through `.contextMenu` or keep the control visible while the row is selected, or insert it with a conditional `if`.

### SwiftUI Pattern 2: Touch reveal with separate state

Keep one state per input: hover for the pointer, a touch-revealed flag for the finger.

```swift
struct Row: View {
    @State private var hovered = false
    @State private var touchRevealed = false
    var body: some View {
        HStack {
            Text(title)
            Spacer()
            if hovered || touchRevealed { Button("Remove", action: remove) }
        }
        .contentShape(.rect)
        .onHover { hovered = $0 }
        .modifier(TouchReveal(revealed: $touchRevealed))
    }
}

struct TouchReveal: ViewModifier {
    @Binding var revealed: Bool
    func body(content: Content) -> some View {
        content.simultaneousGesture(TapGesture(count: 1, inputKinds: .directTouch).onEnded { revealed = true })
    }
}
```

`TapGesture(count:inputKinds:)` and `.directTouch` are macOS 27.0+; before 27 SwiftUI cannot tell a touch tap from a click, so use SwiftUI Pattern 1. For a list, keep a `touchedID` beside `hoveredID`. Hide the control after a short idle or on a touch elsewhere (Approach D). A tap on the revealed control also reaches the row beneath, and both gestures fire in no guaranteed order: defer the row action when the reveal tap is on the control.

## Mac Catalyst and UIKit

### Catalyst Pattern 1: Hover-revealed control with a touch path

Beside the `UIHoverGestureRecognizer`, add a `UITapGestureRecognizer` restricted to direct touch (`allowedTouchTypes = [NSNumber(value: UITouch.TouchType.direct.rawValue)]`) that toggles a separate `touchRevealed` property on `.ended`. Set `hovered` from `.began` and `.changed` only, so `.ended`, `.cancelled` and `.failed` clear it; show the control when either is true.
