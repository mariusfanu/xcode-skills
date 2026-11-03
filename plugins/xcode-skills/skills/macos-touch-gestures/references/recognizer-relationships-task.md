# Task: Recognizer relationships: failure, simultaneity and ownership

Every recognizer you add competes with the recognizers gathered for the touch (see "How AppKit Routes Touch" in SKILL.md), including those of framework-provided views you do not own. A recognizer that waits on one that never fails never fires. Apply after `recognizers-task.md` and `input-kinds-task.md`.

## Detection patterns

- Objective-C delegate spellings: `-gestureRecognizer:shouldRequireFailureOfGestureRecognizer:`, `-gestureRecognizer:shouldBeRequiredToFailByGestureRecognizer:`, `-gestureRecognizer:shouldRecognizeSimultaneouslyWithGestureRecognizer:`, especially ones that `return YES` unconditionally.
- Subclass overrides of `canPrevent(_:)`, `canBePrevented(by:)`, `shouldRequireFailure(of:)`, `shouldBeRequiredToFail(by:)`. UIKit's `require(toFail:)` / `requireGestureRecognizerToFail:` do not exist on `NSGestureRecognizer`.
- `delaysSecondaryMouseButtonEvents`, `delaysOtherMouseButtonEvents`, `delaysMagnificationEvents`, `delaysRotationEvents`, `delaysKeyEvents`.
- Several `NSPressGestureRecognizer` instances with different `minimumPressDuration` on one view chain.
- Popovers closed or recognizers removed inside `gestureRecognizer(_:shouldReceive:)` or `gestureRecognizerShouldBegin(_:)` (Pitfall 5).

## Building the graph

- Keep each recognizer you own in a stored property and compare with `===`, never against framework classes. Do not disable or reorder recognizers that framework-provided views install; make yours yield or coexist through the delegate callbacks, using `NSScrollView.scrollGestureForRelationships` (macOS 27.0+) to relate to scrolling.
- Draw the graph: node per recognizer, edge X to Y if X waits for Y to fail. Any cycle is a bug; any edge into a pinch or rotation recognizer is a bug (`multitouch-task.md` Pitfall 1).
- If a recognizer is cancelled or never recognizes and the file does not show the competitor, check the enclosing scroll view's pan first (`scrollGestureForRelationships`, `scroll-and-drag-task.md` Pattern 3). If still unknown, state the assumed competitor in your report.
- Return true for a specific pair, never as a catch-all; do not make every recognizer simultaneous to "fix" a conflict, or rely on event monitors to resolve conflicts.

## Patterns

### Pattern 1: Describe ownership with the delegate, not with timing

Delegate methods, in order of preference:

1. `shouldBeRequiredToFailBy`: "I take precedence."
2. `shouldRequireFailureOf`: "I yield." Click-count relationships: `click-counting-task.md`.
3. `shouldRecognizeSimultaneouslyWith`: true for passive observers (tap-outside watchers, hover observers) and for gestures that should recognize together (rotate and pinch).
4. Subclass `canPrevent(_:)` and `canBePrevented(by:)` returning false for observers that must never block anyone (for example `final class ClickObservingGesture: NSClickGestureRecognizer`).
5. `gestureRecognizerShouldBegin(_:)` for decisions on state known later (hold versus drag, a point recorded in `shouldReceive`). Failure requirements are determined when the touch lands; `shouldBegin` runs when the recognizer tries to leave `.possible`.

### Pattern 2: Press recognizers that do not wait on each other

**Why a zero-duration press waits on a longer press.** A press with a shorter `minimumPressDuration` does not recognize while a longer press on the same view is still possible, and a longer press only fails on lift or movement. So a zero-duration press (highlight at touch-down) waits on your drag and menu presses. The reverse also bites: a medium drag press can hold back a longer menu press. Symptoms: a highlight never appears on screen before the action runs, action and dismissal happen in the same frame at lift, a press-based text selection starts late behind a short preflight press, drag and drop "sometimes does not work".

A zero-duration press on an ancestor also recognizes at touch-down, before a descendant click (which needs a lift) or pan (which needs movement) can, so the descendant does nothing for touch. Stop the ancestor press from beginning where it is not needed (`gestureRecognizerShouldBegin(_:)`) instead of adding recognizers to the descendant. This is AppKit behavior; SwiftUI gestures follow descendant-wins rules.

Break the dependency for presses that must react at once:

```swift
final class HighlightPress: NSPressGestureRecognizer {
    override func shouldRequireFailure(of other: NSGestureRecognizer) -> Bool {
        if other is NSPressGestureRecognizer { return false }   // never wait on other presses
        return super.shouldRequireFailure(of: other)
    }
}
```

For a drag press and a menu press on one view, choose one outcome per item. Where hold-still on a draggable item must start the drag, invert the rule: return false when this press is shorter than the other, so it recognizes first and the menu press fails through ordinary exclusion.

```swift
final class DragPress: NSPressGestureRecognizer {
    override func shouldRequireFailure(of other: NSGestureRecognizer) -> Bool {
        if let press = other as? NSPressGestureRecognizer,
           minimumPressDuration < press.minimumPressDuration { return false }   // never wait on the menu press
        return super.shouldRequireFailure(of: other)
    }
}
```

### Pattern 3: Do not hold back the mouse path: `delaysPrimaryMouseButtonEvents`

Delaying is useful when a responder would act on a click before the recognizer can recognize it, but it can make a descendant's `mouseDown:` arrive late or never. Check each class's actual default and set the property explicitly: false on any recognizer above a view that needs `mouseDown:` (a header double-click recognizer must not swallow the first `mouseDown:` the enclosing table needs), including touch-only recognizers. Test: click the underlying control with the mouse and check the first click arrives.

```swift
doubleClick.delaysPrimaryMouseButtonEvents = false   // the table still needs the first mouseDown:
```

### Pattern 4: Exclusive gesture behavior

(macOS 27.0+) Set `NSView.exclusiveGestureBehavior` to `.notExclusive` on a container hosting independent controls (panel or popover with several controls) so touching a second control while a first is active works. Use `.exclusive` on a view whose event-tracking loop became a pan, to keep unrelated views from recognizing at the same time.

The app-wide switch is the `NSViewGestureRecognizerIsExclusive` Info.plist key. Absent means YES, and the template for a new macOS app project sets it to NO, so the same code can behave differently across projects: check the key before diagnosing a second touch that is ignored.

```swift
container.exclusiveGestureBehavior = .notExclusive
```

### Pattern 5: Reconcile touch state with current touches

**When to use.** A custom recognizer or view tracks touches with a state machine (drag, multi-finger gesture, freehand drag) closed by one specific end event. A second finger, window change, appearing menu or cancelled touch can remove a touch without that event (so `touchesEnded` never finishes it), wedging the state machine: a drag, freehand drag or window move never ends, or the next one joins it. Reconcile against the touches currently down, in a direct `NSGestureRecognizer` subclass so it may set `state`:

```swift
final class DragTracker: NSGestureRecognizer {
    private var trackedID: NSObject?

    override func touchesBegan(with event: NSEvent) {
        let began = event.touches(matching: .began, in: view)
        guard trackedID == nil, began.count == 1, let touch = began.first else { return }   // one new finger
        trackedID = touch.identity as? NSObject
        state = .began
    }
    override func touchesMoved(with event: NSEvent) {
        guard reconcile(event) else { return }
        state = .changed
    }
    override func touchesEnded(with event: NSEvent) {
        _ = reconcile(event)
    }

    override func touchesCancelled(with event: NSEvent) {
        state = .cancelled
    }

    override func reset() {
        trackedID = nil
    }

    private func reconcile(_ event: NSEvent) -> Bool {      // false after the touch is gone
        let down = event.touches(matching: .touching, in: view).compactMap { $0.identity as? NSObject }
        if let id = trackedID, down.contains(where: { $0 == id }) {
            return true
        }
        state = state == .possible ? .failed : .ended        // the tracked touch vanished
        return false
    }
}
```

- Track touches by `identity`; treat a vanished touch as ended, or cancelled if completing would be wrong (a drag whose drop should not apply). Clear all per-gesture state in `reset()`.
- Before showing a menu or sheet, cancel the gesture up front. Re-entrant callbacks during a cancelled gesture must not start new work.
- Install touch recognizers only in the modes that need them, keep them installed whether or not a touch display is attached, re-sync `isEnabled` when mode, selection or tool changes, and detach them while restoring any host state you changed.
- A flag or timer set for "the next event" (ignore the next focus loss, pending dismissal timer, long-press timer) goes stale the same way: set it when you cause the event, clear it on every end, cancel, fail and abandon path, and cancel a pending timer when you present again.

## Pitfalls

### Pitfall 1: Failure-graph cycles

**Symptom:** a tap on a coasting or just-stopped scroll view is delayed until lift, or a recognizer "sometimes never fires".
**Cause:** example cycles: "everything inside a scroll view waits on the scroll pan" when one of those is also something the scroll pan waits on; a menu recognizer on an ancestor that a drag-lift recognizer waits on, while the menu recognizer waits on the drag.
**Fix:** the set waiting on the scroll pan must exclude recognizers that already wait on the pan, and pinch and rotation. Where a recognizer must not be a failure requirement for anyone, return false from `shouldBeRequiredToFailBy`.

### Pitfall 2: Returning false from simultaneous does not force exclusion

**Cause:** returning true guarantees simultaneity; returning false does not prevent it, because the other recognizer's delegate may return true.
**Fix:** return true from your own recognizer's delegate for each recognizer that must coexist with it; returning true from one side is enough. Never set a delegate on a recognizer you do not own. To enforce exclusion, use a failure requirement.

### Pitfall 3: A framework-provided view's own handling or a blanket delegate blocks yours

**Symptom:** your click works only sometimes on a table or collection item, a control-click menu never appears, or a passive observer stops firing.
**Cause:** the table or collection view's own click or menu handling wins; separately, a delegate that only allows click-with-click simultaneity silently blocks observers.
**Fix:** run alongside (simultaneous, both orders) or yield, deciding from the callback's `other` argument. When your own `NSView` implements the recognizer delegate, publish the full conformance or none. In a shared delegate prefer an allowlist: a visual-feedback-only press coexists with recognizers on its own view; every foreign pair keeps default exclusion. A widened touch hit zone (divider, column border) overlaps neighbors: re-test which recognizer wins there.

### Pitfall 4: Press recognizers that never report failure

**Symptom:** after a drag-and-drop or menu interaction, taps stop working until the next interaction.
**Cause:** delayed state held by a recognizer that never reached `.failed` or `.cancelled`. Toggling `isEnabled` to cancel also drops state for the next touch and can re-trigger delegate evaluation.
**Fix:** reset pending flags in `.failed` and `.cancelled`; use the `isEnabled` toggle only when a menu or modal takes over, and re-enable afterwards.

### Pitfall 5: Side-effects in delegate methods

Delegate methods decide whether a recognizer takes part; they do not mean the gesture was recognized. Do the work, such as closing or removing views, in the recognizer action; use `gestureRecognizer(_:shouldReceive:)` and `gestureRecognizerShouldBegin(_:)` only to decide whether the recognizer takes part.

### Pitfall 6: Press stuck, or state lost when the window changes

**Cause:** recognizers are cancelled when their view's window changes, which is not a user action.
**Fix:** reset press and drag state in `.cancelled`, `.failed` and when the view's window becomes `nil`. Do not reparent a view during an active gesture (a drag that moves a view into another window).

A recognizer that has not reached a terminal state (`.ended`, `.failed` or `.cancelled`) a couple of seconds after input stops is stuck. The system cancels it, but until then it holds its exclusive view and the rest of the app can look unresponsive. To find the cause, recommend that the user run with `-NSCrashOnStuckGestureTimeout 2` (or the user default of the same name): the app then crashes with a stack trace instead of silently cancelling. Tell them to remove it before shipping.

## SwiftUI

### SwiftUI Pattern 1: Composition decides the winner

| Goal | SwiftUI spelling |
|---|---|
| Runs beside others (simultaneity) | `.simultaneousGesture(_:)` |
| "I take precedence" | `.highPriorityGesture(_:)` |
| Double tap before single tap (failure requirement) | `TapGesture(count: 2).exclusively(before: TapGesture(count: 1))` |
| Hold, then drag | `LongPressGesture().sequenced(before: DragGesture())` |

Two `onTapGesture` handlers (count 1 and 2) on one view compete; compose them with `exclusively(before:)` instead. The single tap is then delayed until the double tap fails: test single-tap latency.

### SwiftUI: no equivalent

- No failure graph beyond composition, no `canPrevent`. `exclusiveGestureBehavior`, `delaysPrimaryMouseButtonEvents`, `scrollGestureForRelationships` and `isCancellableByScrollGesture` have no counterpart; `disablesDirectTouchSystemGestures` is set on the `NSWindow` from AppKit code (`multitouch-task.md`).
- Consider using `NSGestureRecognizerRepresentable` only when necessary to achieve the gesture behavior goals.

## Mac Catalyst and UIKit

### Catalyst Pattern 1: Decide the relationship in the delegate

A double-tap recognizer on a cell with no delegate fights the cell's scroll pan, and a single tap fires before the double tap can. Decide each pair, and return true from simultaneity in both argument orders. An editable text view in the same cell should be simultaneous with the double tap, not something the double tap waits on.

```swift
func gestureRecognizer(_ me: UIGestureRecognizer,
                       shouldRecognizeSimultaneouslyWith other: UIGestureRecognizer) -> Bool {
    (me === doubleTap && other === contentScroll) || (me === contentScroll && other === doubleTap)
}
func gestureRecognizer(_ me: UIGestureRecognizer,
                       shouldRequireFailureOf other: UIGestureRecognizer) -> Bool {
    me === singleTap && other === doubleTap          // the single tap yields
}
```

