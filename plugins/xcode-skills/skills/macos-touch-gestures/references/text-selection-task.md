# Task: Text selection and links

A subclass of a framework-provided `NSTextView` that overrides a mouse method, or a custom view that tracks selection in mouse handlers, loses the framework's touch text handling (`SKILL.md`, How AppKit Routes Touch). Field editors and Mac Catalyst views (UIKit text interaction) need no change.

## Detection patterns

- `NSTextView` subclasses that override any mouse method, including empty or `super`-only ones.
- Custom views that draw their own text and track selection in mouse handlers; `NSTextInputClient` implementations that draw their own caret and selection.
- `clickedOnLink:atIndex:` called only from `mouseUp`; `textView(_:clickedOnLink:at:)` never reached on touch.
- `NSTextSelectionDataSource`, `textSelectionMode`, `gesturesForFailureRequirements` (macOS 27.0+).

## Patterns

### Pattern 1: NSTextView subclass with mouse overrides

1. Find why the override exists (delegate notification on click, focus change, clicks on embedded items, a legacy workaround).
2. Replace the reason, not the override:
   - Focus or "was clicked" notification: an observer click recognizer (below). It runs on click end, not mouse-down.
   - Clicks on embedded items: a recognizer that begins only over the item (`gestureRecognizerShouldBegin(_:)` hit-tests the location).
   - Menu on tap-and-hold of selected text: do not clear the selection on press. Provide the menu through `menu(for:)` and let AppKit raise it.
3. Delete the override, including an empty or `super`-only `mouseDown`: a leftover override keeps touch on mouse emulation and can clear the selection before a menu or drag starts. In custom surfaces, clear selection only when the drag session is requested or the press ends.
4. Test with touch and mouse: tap to place the caret, double tap to select a word, long press, tap on a link, tap-and-hold on selected text.

The observer sets its properties in both initializers:

```swift
final class ClickObserver: NSClickGestureRecognizer, NSGestureRecognizerDelegate {
    override init(target: Any?, action: Selector?) {
        super.init(target: target, action: action)
        configure()
    }

    required init?(coder: NSCoder) {
        super.init(coder: coder)
        configure()
    }

    private func configure() {
        allowedTouchTypes = .direct              // mouse uses the default buttonMask
        delaysPrimaryMouseButtonEvents = false
        delegate = self
    }

    func gestureRecognizer(_ me: NSGestureRecognizer,
                           shouldRecognizeSimultaneouslyWith other: NSGestureRecognizer) -> Bool {
        true
    }
}
```

### Pattern 2: Custom text surface (macOS 27.0+)

**When to use.** A custom view draws its own text and tracks selection in mouse handlers (a custom text surface). `NSTextSelectionManager` (macOS 27.0+; the whole snippet needs it) manages selection with recognizers for mouse, trackpad and touch. Attach one and remove the mouse-tracking selection code.

```swift
final class TextCanvasView: NSView, NSTextSelectionManager.Delegate {
    var textSelection: NSTextSelection?                      // required by the delegate protocol
    func installSelection(dataSource: NSTextSelectionDataSource) {
        let manager = NSTextSelectionManager()
        manager.delegate = self
        manager.textSelectionDataSource = dataSource
        manager.textSelectionMode = .selectable               // .editable, .selectable or .nonInteractive
        textSelectionManager = manager                        // installs the recognizers on this view
    }
}
```

- Points from the manager are in the view's coordinate system. If the view is not flipped and the layout is, convert once at the boundary and test hit points near both ends of the text.
- The manager's recognizers are exposed in `gesturesForFailureRequirements`. To make your own tap or press yield to selection, return `true` from `gestureRecognizer(_:shouldRequireFailureOf:)` for any recognizer in that array.
- Provide the drag session through `selectionManager(_:makeDraggingSession:)` with `beginDraggingSession(items:gesture:source:)`. The selection may be cleared before the framework asks for a session; do not rely on it staying set between press and drag.
- Default `textSelectionMode` is `.editable`, which makes a drag of selected text a move. Set `.selectable` on read-only surfaces and refuse drops onto the surface's own selection.

### Pattern 3: Links in text

- `NSTextView`: remove the `mouseUp` override and implement `textView(_:clickedOnLink:at:)`; it fires on click and tap.
- Custom surface: an `NSClickGestureRecognizer` that begins only over a link (`gestureRecognizerShouldBegin(_:)` hit-tests) so ordinary taps go to selection; resolve the character index at `r.location(in: self)` and read `.link`.
- Do not install a link recognizer on a field editor: a single click looks like two.
- A tap on a link in a scrolling surface must stop a decelerating scroll first, not activate the link under the finger.

## Pitfalls

### Pitfall 1: Text selection fires during a pinch or drops text on itself

**Cause:** a press recognizer you added accepts a two-finger press, and the default `.editable` mode turns a drag into a move.
**Fix:** set `textSelectionMode` and refuse self-drops as in Pattern 2. A press that must stay on selected text or an embedded item tolerates about 12 pt of movement (tuning start).
