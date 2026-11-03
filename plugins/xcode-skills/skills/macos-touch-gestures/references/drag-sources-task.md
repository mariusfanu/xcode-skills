# Task: Drag sources for touch

Drags started from `mouseDragged:` or the legacy write-to-pasteboard callbacks stay on the mouse event stream: under a finger the drag never lifts. Framework-provided table, outline and collection views lift and snapshot items when the data source uses pasteboard writers; do not hand-build touch previews for them. Custom views must start drags from a recognizer. Edit destination code only for a rule the drop must enforce, such as refusing a drop onto the dragged selection itself (return `[]` / `.none` from `draggingEntered(_:)`, which `draggingUpdated(_:)` often forwards to, and false from `performDragOperation(_:)` when `draggingSource` is the view and the point is over the selection). SwiftUI `.draggable`, `.onDrag` and `.dropDestination` are framework-driven (test them).

## Detection patterns

- Legacy callbacks: `-tableView:writeRowsWithIndexes:toPasteboard:`, `outlineView(_:writeItems:to:)` / `-outlineView:writeItems:toPasteboard:`, `collectionView(_:writeItemsAt:to:)` / `-collectionView:writeItemsAtIndexPaths:toPasteboard:`.
- A dragging data source without `-tableView:pasteboardWriterForRow:`, `-outlineView:pasteboardWriterForItem:` or `-collectionView:pasteboardWriterForItemAtIndexPath:`.
- `-beginDraggingSessionWithItems:event:source:` called from `mouseDragged:` or `mouseDown:`.
- Deprecated `dragImage(_:at:offset:event:pasteboard:source:slideBack:)`, `dragFile(_:from:slideBack:event:)`, `dragPromisedFiles(ofTypes:from:source:slideBack:event:)`.
- `mouseDown:` overrides that cache the selection "because the table decides later whether this is a drag".
- Hand-built lift, shadow or scale code in `draggingSession(_:willBeginAt:forRowIndexes:)` or `NSDraggingItem.imageComponentsProvider`; frames computed from `event.locationInWindow`.
- Custom pasteboard types that are not reverse-DNS exported identifiers (for example `"_MyItemType"`).

## Patterns

### Pattern 1: Pasteboard writers for table, outline and collection drags

Pasteboard writers give touch lift, snapshots and finger-sized images.

```swift
// Before:
func tableView(_ tableView: NSTableView, writeRowsWith rows: IndexSet, to pasteboard: NSPasteboard) -> Bool {
    draggedItems = rows.map { items[$0] }                 // cached, may go stale
    pasteboard.declareTypes([.myItem], owner: self)
    return true
}

// After:
func tableView(_ tableView: NSTableView, pasteboardWriterForRow row: Int) -> NSPasteboardWriting? {
    let item = items[row]
    return item.isDraggable ? ItemPasteboardWriter(item) : nil   // nil refuses the drag
}
```

Equivalents: `outlineView(_:pasteboardWriterForItem:)`, `collectionView(_:pasteboardWriterForItemAt:)`. `NSBrowser` has `browser(_:pasteboardWriterForRow:column:)` (macOS 27.0+) with `browser(_:draggingSession:willBeginAt:forRowsWith:column:)`; its older write-rows method is deprecated.

- The callback is per row. When the whole selection must drag together, validate it in `draggingSession(_:willBeginAt:forRowIndexes:)` or in each row's writer, and return `nil` consistently.
- Build every writer from the row index and model object (no ivar filled by an old callback; rows that stop dragging after the switch returned `nil` here), and read dragged items from `session.draggingPasteboard`. Remove the `mouseDown:` code that cached the selection.
- Use a reverse-DNS type identifier declared as an exported type (for example `com.example.myapp.item`); test that `NSPasteboardItem` accepts it.
- If a failed drag leaves source items hidden in a collection view, restore them in `collectionView(_:draggingSession:endedAt:dragOperation:)`.

### Pattern 2: Start a custom-view drag from a gesture (macOS 27.0+)

`NSView.beginDraggingSession(items:gesture:source:)` (macOS 27.0+; ObjC `-beginDraggingSessionWithItems:gesture:source:`) takes the driving recognizer and returns an optional `NSDraggingSession` (`nil` when no drag begins). Start it from the recognizer's `.began`.

```swift
@objc private func handleDrag(_ sender: NSPressGestureRecognizer) {
    guard sender.state == .began else { return }   // once a session exists, do not drive it from this recognizer
    let item = NSDraggingItem(pasteboardWriter: writer(at: sender.location(in: self)))
    item.setDraggingFrame(dragFrame(at: sender.location(in: self)), contents: snapshotImage())
    session = beginDraggingSession(items: [item], gesture: sender, source: self)  // macOS 27.0+; nil: no drag began
}

func draggingSession(_ dragSession: NSDraggingSession, endedAt location: NSPoint, operation: NSDragOperation) {
    session = nil                             // drag state ends here, not when the recognizer ends
}
```

- `NSDraggingSession` has no public method that takes a recognizer or forwards its state. Once a session is returned, keep drag state until `draggingSession(_:endedAt:operation:)`; do not clear it on the recognizer's `.ended` or `.cancelled`. If the call returns `nil`, clear state and undo anything changed at `.began`.
- Test on touch and mouse that the image follows the finger or pointer, the drop arrives, lifting over no destination calls `draggingSession(_:endedAt:operation:)`, and a recognizer cancelled before the drag starts leaves no state. If the session does not follow the recognizer on mouse, keep `beginDraggingSession(items:event:source:)` for the mouse path and use the gesture overload for touch only.
- Touch-only recognizer: leave the `mouseDragged:` path for mouse and trackpad. One recognizer for both: remove the `mouseDragged:` path and test modifier-key copy and move. Never leave both drag starts active for the same input.

### Pattern 3: Finger-sized drag images

When a custom image is needed:

- Use a snapshot of the dragged row or item, not a small icon. Render with `bitmapImageRepForCachingDisplay(in:)` and `cacheDisplay(in:to:)`, grid lines and hover overlays off, with an opaque background that differs from the selected background.
- Set the item frame to the snapshot size in the view's coordinate space, keeping the dragged item's origin; do not center the frame on the pointer.
- For multiple items, set `session.draggingFormation`. Leave it at `.none` (the default) when the drag rearranges items within the same view (for example an icon view), so items can be placed precisely. Use `.pile` for lists and collection views.

## Pitfalls

### Pitfall 1: A multi-selection drag is badged "1"

One pasteboard item stands for many app objects: write one pasteboard item per object, or accept the limit.

### Pitfall 2: Drag images are tiny, drift from the finger, or items land offset

The dragging frame is centered instead of at the item's origin, or image component origins include a row indent: use the item's frame in view coordinates; test with indented outline rows.

## SwiftUI / Mac Catalyst

`.draggable(_:preview:)` and `.onDrag(_:preview:)` supply the lift preview. Follow the session with `onDragSessionUpdated(_:)`.

### Catalyst Pattern 1: Gate `UIDragInteraction` when only one side supports drag and drop

When nothing accepts the drop, a long press starts a drag that cannot complete.

```swift
let drag = UIDragInteraction(delegate: self)
drag.isEnabled = canCompleteDrag          // false when only one side supports drag and drop
view.addInteraction(drag)
```

Return `[]` from `dragInteraction(_:itemsForBeginning:)` for items that must not drag.

### Catalyst Pattern 2: Pointer-only selection drags

When one recognizer serves both pointer selection and touch, restrict it to the pointer and let touch use the framework-provided text interaction.

```swift
selectDrag.allowedTouchTypes = [NSNumber(value: UITouch.TouchType.indirectPointer.rawValue)]
```
