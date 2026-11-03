# Task: Context menus for touch

Framework-provided views map press-and-hold to the secondary click. Code that builds a menu must be free of side effects: `menu(for:)` and `-menuForEvent:` can be asked for a menu that is never shown, so they should not change selection or other state. Menus built from the current mouse event, or raised from `mouseDown:`, fail under a finger: wrong item or no menu.

## Detection patterns

- `menu(for:)` / `-menuForEvent:` overrides that read `NSApp.currentEvent` or `window.currentEvent`, compute the row with `rowAtPoint:` / `row(at:)` from an event location, or call `selectRowIndexes(_:byExtendingSelection:)` or otherwise change state.
- An override of the `menu` getter (`override var menu: NSMenu?`, `- (NSMenu *)menu`) that does real work; it can run without the menu being shown (for example on touch-down).
- `rightMouseDown:` overrides, and `mouseDown:` overrides that test `modifierFlags.contains(.control)`. Migrate the mouse handling first (see Pattern 1 and `recognizers-task.md`).
- `+popUpContextMenu:withEvent:forView:` called from `mouseDown:` or `rightMouseDown:`.
- Menu actions that read `selectedRow` or `selectedRowIndexes` and ignore `clickedRow`.
- A custom `NSPressGestureRecognizer` on a view that has a `menu` or on an `NSCollectionView`.

Leave alone: framework-provided `NSPopUpButton` and `NSMenuToolbarItem` menus (test with touch, add no recognizers), and views with no menu (do not add a long-press menu where none existed).

## Patterns

### Pattern 1: Prefer migrating to `menu(for:)` or `menu`

Prefer migrating responder methods like `rightMouseDown:` to `menu(for:)` instead of migrating to gesture recognizers if their purpose is presenting a menu. Using `menu(for:)` is back-deployable and handles all input types, including mouse/trackpad/touch. 
If there is conditional logic in the responder method to gate whether a menu is shown, return `nil` from `menu(for:)` instead. 
If there are side-effects related to presenting the menu, such as highlight effects, use `NSMenuDelegate` callbacks to initiate those side-effects instead.

### Pattern 2: Use `clickedRow` as the menu target

**When to use:** a table or outline view menu that works out its target from the event, from `selectedRow`, or by selecting in `menu(for:)`.

`clickedRow` is set for a click, right-click and touch long press. If the clicked row is not in the selection, act on that row only; if it is, act on the whole selection.

```swift
// Before: target from the event, selects as a side effect
override func menu(for event: NSEvent) -> NSMenu? {
    let location = convert(event.locationInWindow, from: nil)
    let row = self.row(at: location)
    if row >= 0 { selectRowIndexes([row], byExtendingSelection: false) }
    return contextMenu
}

// After: menu is a plain property; actions read clickedRow
@objc func renameItems(_ sender: Any?) {
    for item in targetItems(in: tableView) { rename(item) }
}

func targetItems(in table: NSTableView) -> [Item] {
    let clicked = table.clickedRow
    if clicked >= 0, !table.selectedRowIndexes.contains(clicked) {
        return [items[clicked]]                       // act on the clicked row only
    }
    return table.selectedRowIndexes.map { items[$0] } // act on the selection
}
```

For a long press the current event is not a mouse-down at the press location, so event-derived targets pick the wrong row or section. `clickedRow` is `-1` when the menu was not opened over a row: disable or hide row actions. Read it in the action and in `validateMenuItem(_:)`, not in a stored property written earlier. For a per-cell menu, implement `menu(for:)` on the cell view.

### Pattern 3: Use `NSMenuDelegate` callbacks to lazily populate the menu instead of responder methods

**When to use:** the menu is created lazily in an event handler, or a collection view item's menu is set on the item object. A menu created only inside `mouseDown:` is not available for touch long press.

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    let menu = NSMenu()
    menu.delegate = self
    tableView.menu = menu                 // exists before any interaction
}

func menuNeedsUpdate(_ menu: NSMenu) {
    menu.removeAllItems()
    guard tableView.clickedRow >= 0 else { return }   // no row: empty menu, no menu shown
    menu.addItem(withTitle: "Rename", action: #selector(renameItems(_:)), keyEquivalent: "")
    menu.addItem(withTitle: "Delete", action: #selector(deleteItems(_:)), keyEquivalent: "")
}
```

- Keep `menu` and `menu(for:)` cheap; populate in `menuNeedsUpdate(_:)` so content reflects the clicked row at display time. Do not read the current event there either.
- `NSCollectionView`: set `menu` on the item's `view` (in `loadView` or `viewDidLoad` of the `NSCollectionViewItem`), not on the item object.

## Pitfalls

### Pitfall 1: A collection view or custom view never shows a menu on touch

The menu is set late, on the item object instead of its view, or a custom touch press is in the way. Fix: Pattern 3. Ignore touches on an item that has its own menu so two menus do not compete.

### Pitfall 2: A button or segmented control shows its menu and the menu immediately disappears

**Cause:** the control fires on mouse-down (`sendAction(on: .leftMouseDown)`) and the touch-up dismisses the menu.
**Fix:** fire on tracking-ended-inside or the primary action. For `NSControl`, use `addTarget(_:action:for:)` with `.trackingEndedInside` (macOS 11.0+), or `.primaryActionTriggered` / `.menuActionTriggered` (macOS 27.0+).
