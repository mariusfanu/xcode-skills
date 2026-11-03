# Task: Click counting, double-click and rename

Use `NSClickGestureRecognizer.numberOfClicksRequired` instead of `NSEvent.clickCount` or a hand-rolled double-click timer in code you migrate to recognizers. It counts clicks for mouse, trackpad and touch alike. Leave `clickCount` and timer logic alone in mouse handlers you are not migrating.

## Detection patterns

- `NSEvent.doubleClickInterval` combined with a timer or timestamp comparison; `NSApp.currentEvent` read to decide open or rename.
- Objective-C: `[event clickCount]`, `[NSEvent doubleClickInterval]`, `setDoubleAction:`.
- Click-to-rename: a timer started when a selected row is clicked again, ending in an edit action or a field-editor start.
- An "open" or "activate" action triggered from `mouseUp` after a click-count check.

## Patterns

### Pattern 1: Single and double click on one view

Two click recognizers; the single waits for the double to fail.

```swift
let single = NSClickGestureRecognizer(target: self, action: #selector(handleSingle(_:)))
single.numberOfClicksRequired = 1
single.allowedTouchTypes = .direct
let double = NSClickGestureRecognizer(target: self, action: #selector(handleDouble(_:)))
double.numberOfClicksRequired = 2
double.allowedTouchTypes = .direct
single.delegate = self
addGestureRecognizer(single)
addGestureRecognizer(double)

func gestureRecognizer(_ me: NSGestureRecognizer,
                       shouldRequireFailureOf other: NSGestureRecognizer) -> Bool {
    guard let me = me as? NSClickGestureRecognizer,
          let other = other as? NSClickGestureRecognizer else { return false }
    return other.numberOfClicksRequired > me.numberOfClicksRequired && other.buttonMask == me.buttonMask
}
```

Match `buttonMask` as well as click count, so a primary double-click does not make a secondary single click wait. For an item in a selectable grid, where a single click selects and a double click opens it, show the pressed highlight on touch-down and change the selection on release.

### Pattern 2: Container double-click and child single-click

**When to use.** A custom item in a selectable grid has a double-click action, and a button-like child inside it (a favorite badge) has its own single click. A click on the badge must toggle the favorite and must not count toward the item's double-click. On the lower-count recognizer's delegate:

```swift
func gestureRecognizer(_ me: NSGestureRecognizer,
                       shouldBeRequiredToFailBy other: NSGestureRecognizer) -> Bool {
    guard let me = me as? NSClickGestureRecognizer,
          let other = other as? NSClickGestureRecognizer else { return false }
    return other.numberOfClicksRequired > me.numberOfClicksRequired && other.buttonMask == me.buttonMask
}
```

If a click anywhere on the item, including the badge, should select it, put the single and double recognizers on the item (Pattern 1) and give the badge no recognizer.

### Pattern 3: App-owned timing

For timing that stays app-owned outside the recognizer (such as a click-to-rename timer), use `NSEvent.doubleClickInterval` as the window for mouse and as the starting point for touch; shorten it only if consecutive taps merge into one in testing. Recognizer click counts need no interval of your own.

## Pitfalls

### Pitfall 1: A click opens an item as a double click

**Symptom:** an item opens or activates as a double click, and a long press followed by moving away activates it.
**Cause:** app-owned double-click timing ignores which button produced the click.
**Fix:** `numberOfClicksRequired` with no timer of your own; handle `.cancelled` so a drag-away does not activate.

### Pitfall 2: Reading `clickCount` from a touch event

**Symptom:** a double-tap handler raises an exception, because it reads `clickCount` from a direct-touch event.
**Cause:** `NSEvent.clickCount` is valid only for mouse events and raises for other event types.
**Fix:** use `numberOfClicksRequired` (Pattern 1). To keep a mouse `clickCount` path unchanged, add a touch-only `NSClickGestureRecognizer` (`numberOfClicksRequired = 2`, `buttonMask = 0x0`, `allowedTouchTypes = .direct`), read the position from `location(in:)`, and make sure one touch runs one action (`input-kinds-task.md` Pitfall 2).
