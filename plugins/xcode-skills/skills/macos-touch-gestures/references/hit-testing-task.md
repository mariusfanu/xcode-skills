# Task: Hit testing and touch target size

If a target is not easily tappable, first check that touches can reach the full view and that it is not partially overlapped by another view (Patterns 1–3). Then, if there are still issues with tappability, address the size of the touch target, even if the view visually remains the same size (Patterns 4 and 5).

AppKit already does some of this work for touch, and none of it changes the existing mouse behavior:

- A touch is hit-tested once, when it begins, with a forgiveness radius of about 16pt around the touch point. `NSControl` subclasses and views with gesture recognizers installed are prioritized over views that are neither.
- `NSView` passes a touch hit-test on to descendants within a forgiveness radius, even when the point is outside the descendant, or outside the parent’s bounds if the parent does not clip. This gives a subview the chance to extend its hit area for touch, but a subview only takes the touch if it claims the point. `NSView` by default only returns itself as a hit-test result strictly within its bounds, even if it ends up descending its subtree.
- Some `NSControl`s claim points outside their bounds: the control must be enabled and interactive (text fields only when editable or selectable, image views and level indicators only when editable), and not a button that requires a very deliberate press, such as a destructive action. In each dimension shorter than 24pt, its touch target grows around its center to 24pt. A dimension already 24pt or longer is not grown automatically. No control is grown if an ancestor cuts into it, or if it is hidden or rotated. AppKit-provided controls, such as the disclosure button in a view-based `NSOutlineView`, already work this way.
- A plain `NSView` claims a touch only within its bounds, even if it has a gesture recognizer, and there is no API besides its frame and `hitTest(_:)` for it to opt into a grown target. A touch just outside it goes to a nearby control that claims it, or to whichever view contains the point, usually its superview.
- The touch is offered to the recognizers on the hit view and on each of its ancestors. This is why a recognizer on a container sees touches on the views inside it (Pattern 5). `gestureRecognizer(_:shouldReceive:)` can see a touch whose location is outside the recognizer’s view.

## Detection patterns

- Small interactive geometry: dividers of 1–3 pt, handles of 9–14pt, disclosure controls near 12pt, rows of 19–28pt; views that draw nothing but must be tappable.
- A custom `NSView` subclass (not an `NSControl`) that acts as a small button, toggle, or handle.
- Interactive subviews outside their parent’s `bounds` reached by a custom `hitTest(_:)`; an overlaid sibling in front of interactive content.
- `gestureRecognizer(_:shouldReceive:)` or a recognizer action that rejects a touch with `bounds.contains(_:)` or `NSPointInRect`.
- A `hitTest(_:)` call passed a point from `location(in:)` without converting it.
- SwiftUI `Spacer()` or outer padding as the tappable area; tiny `.frame(width:height:)` on interactive views.

## Patterns

### Pattern 1: Make interactive views reachable

**When to use.** A touch near or on a control never reaches it, and no other view covers it.

- Keep interactive subviews inside their ancestors’ bounds. An ancestor that clips to its bounds filters out a hit-test before it gets to that subview, and AppKit grows a control’s touch target only when the control lies wholly inside every ancestor. A custom `hitTest(_:)` override can reach a subview outside its parent, but that only applies to the views that subclass and override that method.
- Do not hide an interactive control inside a view that claims every touch itself. If the hit view is a container that handles the click, a nested button that should keep its own action will not receive it; let the button be the hit view, or have the container’s handler check what is under the point and forward to the button.

### Pattern 2: Covering siblings and overlapping views

**When to use.** The target is reachable and large, but touches never arrive or reach an ancestor instead. The view that hit testing resolves decides which recognizers see the touch.

**Overlapping views.** An overlay sibling in front of interactive content takes the touch even if it draws nothing, and returning `super.hitTest` does not remove it. Return `nil` for the overlay itself and let its subviews through. This also removes anything the overlay does with the touch itself, such as its own recognizers or mouse handling, so only do this for an overlay that has none. If the overlay exists for accessibility, test touch and VoiceOver.

```swift
final class PassthroughOverlay: NSView {
    override func hitTest(_ point: NSPoint) -> NSView? {
        let hit = super.hitTest(point)
        return hit === self ? nil : hit      // the overlay itself is transparent to hit testing
    }
}
```

### Pattern 3: Use consistent coordinate spaces in every hit-test call

**When to use.** Hit testing returns the wrong view for some positions, mostly in nested, scrolled, or zoomed content.

The point given to `NSView.hitTest(_:)`/`-[NSView hitTest:]` is in the receiver’s superview’s coordinate system (the same coordinate system that the view’s `frame` is in). And the point from `NSGestureRecognizer.location(in:)`/`-[NSGestureRecognizer locationInView:]` is in the coordinate space of the view given as the parameter. So you may need to convert between coordinate spaces to work between these two APIs.

```swift
let gestureLocation = gesture.location(in: self)
let before = self.hitTest(gestureLocation)                          // self’s space (wrong)
let after = self.hitTest(convert(gestureLocation, to: superview))   // superview’s space (correct)
```

For magnified or scrolled content (a zoomed canvas, a zoomed image view) convert between coordinate spaces with `convert(_:from:)` and `convert(_:to:)` instead of applying scroll offset, magnification or backing scale by hand, or subtracting across flipped views.

### Pattern 4: Accept touches that land outside a small control

**When to use.** The small target is an enabled `NSControl` subclass whose recognizers or actions reject touches near it. AppKit already grows its touch target to at least 24pt, so the fix is not to add forgiveness but to stop rejecting the touches AppKit routed to the control.

- A touch in the grown area is delivered to the control’s recognizers with a location outside `bounds`. Accept it: in `gestureRecognizer(_:shouldReceive:)` and in a re-test at `.ended`, compare against the touch target. Use the touch target only for touch; a mouse keeps testing `bounds`.
- Test points against rects with `isMousePoint(_:in:)` (ObjC `-mouse:inRect:`, `NSMouseInRect`), which accounts for flipped coordinates.
- Keep the control wholly inside its ancestors (Pattern 1); a clipped control is not grown.
- A target that should be larger than 24pt, or a custom view that is not an `NSControl`, needs Pattern 5.

```swift
// Before: drops the touches AppKit routed to the control through its grown target
let isInside = bounds.contains(recognizer.location(in: self))

// After: in a touch-only recognizer (`allowedTouchTypes = .direct`, `buttonMask = 0x0`), test against at least the 24pt minimum; the mouse path still tests `bounds`
let isInside = isMousePoint(recognizer.location(in: self), in: touchTarget())

func touchTarget() -> NSRect {
    let dx = max(0, (Self.minimumTouchTarget - bounds.width) / 2)
    let dy = max(0, (Self.minimumTouchTarget - bounds.height) / 2)
    return bounds.insetBy(dx: -dx, dy: -dy)
}
static let minimumTouchTarget: CGFloat = 24
```

### Pattern 5: Touch-only target on a container view

**When to use.** A custom view (not an `NSControl`) has a small hit area, or a target needs more than 24pt. Leave the small view’s own hit area (the mouse path) alone, and give touch a larger area through a container view that you own.

Wrap the small view in a container whose frame is the touch target, and put a touch-only recognizer on the container. A touch in the margin hits the container; a touch on the small view still reaches the container’s recognizer, because recognizers on ancestors of the hit view receive the touch too. Do not widen `hitTest(_:)` itself; it runs for every input type, so it changes the mouse as well.

```swift
final class DisclosureContainer: NSView {
    let disclosure = DisclosureView()         // 12 x 12pt; handles the mouse with a mouse-only recognizer
    static let touchOutset: CGFloat = 6       // 12 + 2 x 6 = 24pt touch target

    override init(frame: NSRect) {
        super.init(frame: frame)
        disclosure.translatesAutoresizingMaskIntoConstraints = false
        addSubview(disclosure)
        NSLayoutConstraint.activate([
            disclosure.centerXAnchor.constraint(equalTo: centerXAnchor),
            disclosure.centerYAnchor.constraint(equalTo: centerYAnchor),
            widthAnchor.constraint(equalTo: disclosure.widthAnchor, constant: 2 * Self.touchOutset),
            heightAnchor.constraint(equalTo: disclosure.heightAnchor, constant: 2 * Self.touchOutset),
        ])

        let tap = NSClickGestureRecognizer(target: self, action: #selector(handleTouchTap(_:)))
        tap.buttonMask = 0x0                    // touch only; the mouse path stays on `disclosure`
        tap.allowedTouchTypes = .direct
        tap.delaysPrimaryMouseButtonEvents = false
        addGestureRecognizer(tap)
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    @objc private func handleTouchTap(_ tap: NSClickGestureRecognizer) {
        guard tap.state == .ended else { return }
        disclosure.toggle()
    }
}
```

- Install the recognizer only on a view managed by the same code as the small view. Do not add it to a superview you do not own, such as a framework-provided view or a view that another component lays out: you cannot rely on when it exists, when it changes, or what else it does with touches. If no suitable view exists, add the container.
- The container owns touch for the small view. If the small view overrides `mouseDown:`, AppKit can also emulate a click on it for a touch, and the action runs twice. Move its mouse handling to a mouse-only recognizer (`allowedTouchTypes = []`, see `input-kinds-task.md`) before adding the container.
- Do not override mouse methods on the container. Mouse clicks in its margin then pass up the responder chain as before.
- The container wins touches inside its frame over a neighbor’s grown target, so grow toward empty space, not into a neighbor. If the growth is asymmetric, flip it in right-to-left layouts.
- Grow only dimensions that are too small to tap, and only up to the floor. Do not extend a target that is already large enough: the empty space between controls is where a touch clears the focused view, brings a window to the front, or reaches a gesture on the container, and every extension takes some of it away.
- A divider or handle that its parent draws has no subview to wrap. Put a touch-only recognizer (`allowedTouchTypes = .direct`, `buttonMask = 0x0` as above, so the mouse path is unchanged) on the parent you own and gate it in `gestureRecognizer(_:shouldReceive:)` with a band around the divider, as `NSSplitView` does:

```swift
func gestureRecognizer(_ recognizer: NSGestureRecognizer, shouldReceive touch: NSTouch) -> Bool {
    guard recognizer === dividerPan else { return true }
    let band = dividerRect.insetBy(dx: -Self.dividerTouchOutset, dy: 0)   // a vertical divider grows sideways
    return isMousePoint(touch.location(in: self), in: band)
}
static let dividerTouchOutset: CGFloat = 10
```

## SwiftUI

### SwiftUI Pattern 1: Enlarge the content shape, not the visuals

SwiftUI hit-tests a view by its content shape. Without one, a shape is hit only inside its path, `Color.clear`, fully clear fills and the empty space between views are not hit, and `Text` and `Image` are hit across their whole frame, including transparent pixels. Set the shape with `contentShape(_:eoFill:)`, or `contentShape(_:_:eoFill:)` with a kind.

```swift
// Before: only the 12 pt glyph is tappable
Button {
    isExpanded.toggle()
} label: {
    Image(systemName: "chevron.right")
}
.buttonStyle(.plain)

// After: padding is inside the label, so the padded rect is the hit shape
Button {
    isExpanded.toggle()
} label: {
    Image(systemName: "chevron.right")
        .padding(6)
        .contentShape(.rect)
}
.buttonStyle(.plain)
```

- Use `.contentShape(.rect)` on views whose tappable area is a transparent fill, `Color.clear` or `Spacer`.
- A content shape applies to every input, including the pointer; SwiftUI has no touch-only hit area, so confirm with the user that the larger area is acceptable for the pointer before growing it. To grow the shape without changing layout, use `.contentShape(Rectangle().inset(by: -4))`, only where neighbors have room, and keep the 24pt floor.
- For a covering overlay, `allowsHitTesting(false)` is the nearest control to Pattern 2; test it.

## Mac Catalyst and UIKit

### Catalyst Pattern 1: Hit-test points are in the view's own coordinate space

`UIView.hitTest(_:with:)` and `point(inside:with:)` receive the point in the receiver's own coordinate system, not its superview's as `NSView.hitTest(_:)` does (Pattern 3). Do not copy the AppKit conversion to the superview's space into UIKit code: test `bounds.contains(point)` directly, and convert with `convert(_:from:)` or `convert(_:to:)` when the point comes from another view.

## Guidance numbers

Conventions, not requirements: follow the app’s design guidelines and the Human Interface Guidelines. Do not apply one number everywhere.

| Value | What it controls |
|---|---|
| 24 x 24 pt | Floor for the touch target of any interactive control. AppKit grows small, enabled controls to this size for touch, per dimension, and a control already at or above it in a dimension is not grown in that dimension. |
| 16 pt | Forgiveness radius AppKit and SwiftUI use when picking among nearby views for a touch. It is not a target size; use it to understand why a nearby touch lands on a view, not to size one. |
| 44 x 44 pt | Human Interface Guidelines touch-target convention. Touch-only areas should bring a smaller drawn size up to 24 pt, and toward 44 pt only where neighbors have room and the empty space around them is not needed. |
| 10-20 pt each side | Hit band for thin dividers, column borders and splitters, touch only. |
