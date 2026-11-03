# Task: Pan, pinch and rotate on one canvas

Leave framework-provided `NSScrollView` magnification and `WKWebView` content alone.

## Detection patterns

- `magnifyWithEvent:`, `rotateWithEvent:`, `smartMagnifyWithEvent:` overrides; `NSScrollView` with `allowsMagnification` plus a custom pinch or pan.
- A pan without `minimumNumberOfTouches` / `maximumNumberOfTouches`, or with `numberOfTouchesRequired` (deprecated in macOS 27.0).
- App-defined three- or four-finger gestures built on `NSTouch` (`touchesBegan(with:)`, `touches(matching:in:)`). Direct touches reach only gesture recognizers, so track them in an `NSGestureRecognizer` subclass.
- `UIPinchGestureRecognizer`, `UIRotationGestureRecognizer` in UIKit files.

## Patterns

### Pattern 1: Pan, pinch and rotate on one canvas

Keep three recognizers (pan, `NSMagnificationGestureRecognizer`, `NSRotationGestureRecognizer`), allow simultaneous recognition, and decide per gesture which components apply:
- Cap the pan: `minimumNumberOfTouches` = `maximumNumberOfTouches` = 1 when one finger pans and two transform. An uncapped pan (maximum defaults to `NSIntegerMax`) can claim a pinch.
- Use dead zones so a twist does not zoom and a drag does not rotate (a twist moves the centroid; a drag produces some pinch). Rotation or zoom applies only after its component leaves the zone, measured from the zone edge so content does not jump on crossing. Use a wider zone while two-finger scroll is enabled; do not clamp rotation.
- Keep trackpad rotation and touch transform as separate recognizers when a tool can disable the touch path.
- If an item is selected, give it a two-finger gesture only when the touch centroid is over it; otherwise the canvas owns it.
- Values are cumulative within one gesture. Store the start value at `.began` and compute from it every callback; never add to a running total, which double counts when a finger lands or lifts mid-gesture.

```swift
@objc private func handleRotation(_ sender: NSRotationGestureRecognizer) {
    let deg = sender.rotationInDegrees              // cumulative from the start of the gesture
    switch sender.state {
    case .began:
        rotationStart = angle
        rotationApplies = false
    case .changed:
        rotationApplies = rotationApplies || abs(deg) > Self.rotationDeadZone   // start at 10 to 12 degrees; about 20 while two-finger scroll is enabled
        if rotationApplies {   // measured from the zone edge, no clamp
            angle = rotationStart + deg - (deg > 0 ? Self.rotationDeadZone : -Self.rotationDeadZone)
        }
    case .cancelled, .failed:
        if rotationApplies { angle = rotationStart }   // undo only what this gesture did
    default: break
    }
}
```

Magnification has the same shape: set `startScale` at `.began`, `scale = startScale * (1 + sender.magnification)` on `.changed`, restore `startScale` on `.cancelled` and `.failed`. A magnification or rotation recognizer reports the same cumulative value for touch and trackpad, so a gain error is in how the value is applied (units, base value, a non-linear zoom mapping). Recompute from the start value in the model's own unit; do not add a constant factor for touch.

### Pattern 2: Fail early instead of staying possible

A recognizer that cannot apply should fail as soon as it knows; anything waiting on it is starved while it stays `.possible`. A rotation recognizer stays `.possible` until it rotates or a finger lifts.

- Delegate: return false from `gestureRecognizerShouldBegin(_:)`, which transitions the recognizer to `.failed`. It is consulted only when the recognizer tries to leave `.possible`, so a rotation that has not tried to begin cannot be failed this way. `location(in:)` is valid there.
- Subclass: to fail while still `.possible`, subclass `NSGestureRecognizer` directly and set `state = .failed` from `touchesBegan(with:)` and `touchesMoved(with:)`. Compute centroid and finger spread (a spread change of about 10% means pinch, not rotation) from `event.touches(matching: .touching, in: view)` with `touch.location(in: view)`, because the recognizer's own `location(in:)` is not set yet. Setting `state` from a subclass of a concrete recognizer such as `NSRotationGestureRecognizer` is outside the documented contract. Event ordering differs between trackpad (no touches delivered) and direct touch; test both. A custom recognizer must reach a terminal state on every path: override the whole family (`touchesBegan` also needs `touchesMoved`, `touchesEnded` and `touchesCancelled`), and go to `.cancelled`, not `.failed`, when the system takes the touches for its own gesture. To detect a missing override, recommend that the user run with `-NSGestureRecognizerCrashOnMissingOverrides YES`; it does not catch an action that ignores `.cancelled`. Prefer the standard recognizers over a custom one where they fit.

```swift
func gestureRecognizerShouldBegin(_ me: NSGestureRecognizer) -> Bool {
    guard me === canvasRotation else { return true }
    return item(at: me.location(in: self)) != nil     // rotation only applies over an item
}
```

Pinch and rotation can keep running on the remaining finger when one lifts, and a pan with `maximumNumberOfTouches` keeps going when extra touches land. Do not assume `.ended` on the first lift or `.failed` on an extra touch.

### Pattern 3: Claiming three-finger and four-finger gestures

(macOS 27.3+) Set `NSWindow.disablesDirectTouchSystemGestures = true` while a window needs three or more fingers for its own gestures. Default false; applies to that window while visible. Set it only on windows that need it, reset when the mode ends, never app-wide; set it from AppKit code, not on a hosted view. Document in the app that system gestures are unavailable there.

```swift
// Before: a recognizer's touchesBegan(with:) tracks three fingers and the system gesture competes
// After: the window claims the fingers while the canvas is active (macOS 27.3+)
window.disablesDirectTouchSystemGestures = canvasIsActive
```

## Pitfalls

### Pitfall 1: Pinch and rotation behind failure requirements

**Symptom:** after adding a "wait for the other gesture to fail" relationship, trackpad pinch or rotate does nothing, or only sometimes.
**Cause:** input without touches (a trackpad pinch) never lets a waiting recognizer fail, and a rotation stays `.possible` until it rotates or a finger lifts.
**Fix:** never put pinch or rotation behind a failure requirement or make another recognizer wait on them. Let them compete with `shouldRecognizeSimultaneouslyWith`, and fail the rotation early when it cannot apply (Pattern 2). Test with a trackpad.

## SwiftUI

### SwiftUI Pattern 1: Pinch and rotate together

Attach both with the same `inputKinds` (macOS 27.0+), the second with `.simultaneousGesture`. Do not chain them with `exclusively(before:)` or `sequenced(before:)` 

```swift
content
    .gesture(MagnifyGesture(inputKinds: .directTouch).onChanged { scale = $0.magnification })
    .simultaneousGesture(RotateGesture(inputKinds: .directTouch).onChanged { angle = $0.rotation })
```

### SwiftUI Pattern 2: Claim three- and four-finger gestures

(macOS 27.3+) `View.directTouchSystemGesturesDisabled()` is the SwiftUI counterpart of `NSWindow.disablesDirectTouchSystemGestures` (Pattern 3). Apply it to the view that needs three or more fingers, only while that mode is active.

### SwiftUI Pattern 3: Replace `MagnificationGesture` and `RotationGesture`

Both are deprecated (renamed to `MagnifyGesture` and `RotateGesture`, available since macOS 14), and only the new types take `inputKinds:`. The old closure value moves to `MagnifyGesture.Value.magnification` and `RotateGesture.Value.rotation` (`scale = $0` becomes `scale = $0.magnification`); `minimumScaleDelta` and `minimumAngleDelta` keep their names.

## Mac Catalyst and UIKit

Return true from `gestureRecognizer(_:shouldRecognizeSimultaneouslyWith:)` for the pinch and rotation pair in both argument orders, and give no other recognizer a failure requirement on them. Pan touch counts: `minimumNumberOfTouches` and `maximumNumberOfTouches` on `UIPanGestureRecognizer`.
