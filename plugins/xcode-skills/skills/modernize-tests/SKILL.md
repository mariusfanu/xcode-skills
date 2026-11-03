---
name: modernize-tests
description: |
  Migrate XCTests and XCUIAutomation suites to Swift Testing,
  and update existing Swift Testing tests to recommended
  patterns. Use when working with XCTest, XCUIAutomation, or when
  improving existing #expect/#require, @Test, and @Suite code.
---
This skill covers two distinct actions:
- Migrating XCTests to Swift Testing, and
- Updating existing Swift Testing tests to recommended patterns

Much of the migration process is standard (mapping XCTAssert functions to
`#expect`, test cases to `@Test`, etc). However, there are specific areas where
the migration may be non-obvious, so this file attempts to describe most of
those places.

## Workflow

Survey the candidate files in one pass before editing any of them, looking for
the patterns described in this skill.

Settle naming and `struct`/`final class` choices up front so they stay
consistent, then migrate in parallel. Test files are typically independent, so
fan out subagents if you feel necessary. For small migrations, skip the fan-out.

Within a file, migrate one test class at a time. A single file may contain both
XCTest and Swift Testing tests during migration, and may import both frameworks.

## What cannot migrate

- Tests using the `measure { ... }` family must remain XCTests. Other methods in
  the same `XCTestCase` can still migrate.

## Behavioral differences

### Foundation is no longer re-exported

XCTest re-exports Foundation, but `Testing` does not. When removing `import
XCTest`, check for Foundation types (`URL`, `CharacterSet`, `ProcessInfo`,
`Data`, etc.) and add `import Foundation`.

### continueAfterFailure affects all assertions after it is set

By default `continueAfterFailure` is true and test failures do not interrupt the
test case. When `continueAfterFailure = false`, test failures throw an exception
that interrupts the test.

- When set in a test method, every subsequent assertion in that method should
use `try #require(x)`, and the method needs `throws`.
- When set in `setUp`: this applies to **all assertions in all test methods** of
that class, including `tearDown` and subsequent `setUp` calls.

Never convert an existing `try #require` down to `#expect`. This would change
test behavior and cause it to continue executing after the failure.

### deinit forces a reference or actor type

Prefer `struct` for suites. If `deinit` is needed (for example, when migrating a
`tearDown`), use `actor` or `final class`.

### Isolation and parallelism

XCTest runs synchronous tests on the main actor, sequentially within an
XCTestCase class. Swift Testing runs test functions concurrently on the
cooperative Swift Concurrency thread pool.

- Add `@MainActor` only where the test explicitly relied on XCTest's implicit
main-actor isolation. Do not add it defensively.
- Add `@Suite(.serialized)` where tests share mutable state.

### No accuracy-based equality

There is no equivalent to `XCTAssertEqual(_:_:accuracy:)`. Compare with explicit
floating-point tolerance. If the client already uses `swift-numerics`, you can
use the `isApproximatelyEqual` method.

### Public API only

Do not introduce usage of underscore-prefixed symbols such as `#_sourceLocation`; only use public API.
For source locations, always use the full `SourceLocation(fileID:filePath:line:column:)` initializer.
If the test suite already uses #_sourceLocation, do not replace the existing usage as part of modernization.

### UI tests with XCUIAutomation

Swift Testing tests in UI test bundles can use XCUIAutomation APIs directly.

In UI test mode, tests are run serially on the main actor. Most XCUIAutomation
API is `@MainActor`-bound, so UI test suites or test functions should be
`@MainActor` so the isolation is explicit and the test doesn't need extra
`await`s.

Failures inside XCUIAutomation APIs will end tests
automatically even if the test function is not marked `throws`. When
a test fails as a result of an automation failure like a missing
element or an event that fails to deliver, the entire test
runner exits and is relaunched to continue running the rest of the
test suite. Keep this in mind as it may cause global values to be
reinitialized, which may perform side effects.

When migrating XCUIAutomation UI tests from XCTest, replace
base XCTAssert calls with Swift Testing equivalents, but do not
insert defensive existence checks unless they match existing
assertion patterns.

Example:

```swift
@MainActor
struct MyUITests {
    let app = XCUIApplication()

    init() {
        app.launch()
    }

    @Test
    func `Increment button adds 1 to title`() throws {
        let incrementButton = app.buttons["Increment"].firstMatch
        let valueText = app.textFields["Value"].firstMatch
        // Ensure the text is 0 to begin with, otherwise
        // the post-tap comparison will be wrong.
        try #require(valueText.value as? String == "0")
        incrementButton.tap()
        #expect(valueText.value as? String == "1")
    }
}
```

## Conversion techniques

### setUp state

Frequently, `XCTestCase` classes will declare implicitly-unwrapped stored
properties and initialize them in `setUp`. Swift Testing does not have the same
two-stage initialization; instead, initializers are run fresh before each test.

These properties should be initialized inline, or in `init` if the
initialization is complex, throwing, or async. If the property has value
semantics and the suite is a `struct`, methods that mutate this property must be
marked `mutating`. Throwing during `init` is treated as a test failure.

```swift
struct MyTests {
    var fixture = Fixture()
    @Test mutating func `Fixture behaves as expected`() {
        #expect(fixture.doSomething())
    }
}
```

### Naming

If converting an XCTest, drop the `test` prefix and add `@Test`.

In Swift Testing, if a test name reads as a description of a behavior,
an assertion, or a multi-word claim about program behavior, use a raw
identifier in sentence case.

Simple names, however, are fine to remain lowerCamelCase.

- `func testIgnition()` -> `@Test func ignition()`
- `func testEngineDoesNotStall()` -> ``@Test func `Engine does not stall`()``
- `@Test func clampsAtBoundaries()` -> ``@Test func `Clamps at boundaries`()``

### Errors

`#expect(throws:)` replaces `XCTAssertThrowsError`.
`#expect(throws: Never.self)` replaces `XCTAssertNoThrow`.
When the error type is `Equatable` and the value is known, pass the specific
error rather than `(any Error).self`.
`#expect(throws:)` returns the thrown error (or `nil` if the function didn't
throw). If the test inspects the thrown error, save it to a local variable:

```swift
let error = #expect(throws: (any Error).self) { try f() }
```

### Promote XCTFail and Issue.record

Where it is not disruptive, convert `XCTFail`/`Issue.record` into `#expect` or
`#require`, choosing based on whether the test exits afterward. Where the
expectation's source is self-explanatory, drop the now-redundant message.

```swift
// Before

guard let object = somethingOptional() else {
    Issue.record("Could not get object")
    return
}
guard object.isAvailable() else {
    Issue.record("Object not available")
    return
}
if !object.performOperation() {
    Issue.record("Failed to perform operation")
}

// After

let object = try #require(somethingOptional(), "Could not get object")
try #require(object.isAvailable())
#expect(object.performOperation())
```

### Expectations become confirmations

`XCTestExpectation` + `fulfill()` + `await fulfillment(of:)` becomes
`confirmation()`:

```swift
await confirmation("...") { confirm in
    doWork(completion: {
        confirm()
    })
}
```

When `assertForOverFulfill = false` with an `expectedFulfillmentCount`, use a
range: `expectedCount: 10...`.

### Skipping

- `try XCTSkipIf(condition)` -> `@Test(.disabled(if: condition))`
- `try XCTSkipUnless(condition)` -> `@Test(.enabled(if: condition))`
- `throw XCTSkip("reason")` mid-test -> `try Test.cancel("reason")`

When the function skips based on OS version or platform availability, use an
`@available` attribute on the test function rather than `.enabled(if:)`.

### Known issues

`XCTExpectFailure("...", ...) { ... }` -> `withKnownIssue("...") { ... }`.

The `.nonStrict()` option (or `strict: false`) becomes `isIntermittent: true`.
Conditional and matching forms use the `when:` and `matching:` trailing
closures.

### Attachments

`XCTAttachment` + `self.add(attachment)` → `Attachment.record(value)`.

The value must conform to `Attachable`, which is automatic for `Codable` and
`NSSecureCoding` types when Foundation is imported.

## Improving existing Swift Testing code

- Collapse tests that loop over inputs, or repeat the same logic across
near-identical functions, into `@Test(arguments:)` parameterized tests.
- Look for `Issue.record` that should be an expectation (see above).
- Check that `@MainActor` and `.serialized` are present only where actually
required.