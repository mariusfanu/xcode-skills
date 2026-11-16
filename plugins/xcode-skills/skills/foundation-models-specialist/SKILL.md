---
name: foundation-models-specialist
description: "Use Apple's language models from Swift with FoundationModels: LanguageModelSession, prompting, @Generable structured output, tool calling, on-device versus server models."
---
This is official Apple guidance written by the FoundationModels framework engineers, so consider this guide ground-truth and fully up-to-date. The guide's focus is idioms, patterns, performance, and strategy, so it _augments_ the FoundationModels docs. Still refer to the docs for essentials and API reference.

Read Essential Tips first, then dig into topics as needed with References.

# Essential Tips

FoundationModels is the recommended way to run an LLM within a Swift app in the Apple ecosystem. Apple Intelligence refers to the user experience on Apple platforms, while Apple Foundation Models (AFM) refer to specific LLMs used by Apple Intelligence. FoundationModels is app developers' access to AFM models ("Apple models"), but the framework is compatible with non-Apple custom LLM integrations too in 27.0+ targets.

### Sessions

Always verify model availability before starting a session, otherwise the session will throw. See `references/apple-models.md` for `SystemLanguageModel` and `PrivateCloudComputeLanguageModel` availability checks.

**`LanguageModelSession`**: can handle 1 request at a time. You _can_ create and run multiple in different Swift Tasks at a time, but for on-device models they'll end up running sequentially so it might not be a savings.

- Don't add timeout logic unless the user specifically asks for it or has tried a session without it first.
- If you're using `SystemLanguageModel` no need to pass it for the `model` session argument; it's the default.
- If you need to cancel a `respond` or `streamResponse` call, cancel the Swift Task enclosing the `try await` part. If the user no longer needs the result, always cancel the Task right away to reduce memory load and latency so other sessions/prompts in your experience aren't queued waiting on the old session to finish.
- Model, tools and instructions go on the session initializer, in that order, never on `respond` or `streamResponse`. Passing `model: SystemLanguageModel.default` together with a trailing instructions closure is ambiguous and won't compile, so omit `model:` for the system model:

```swift
let session = LanguageModelSession(tools: [FindHotels()]) {
    "You are a friendly travel assistant."
}
let response = try await session.respond(to: "Find a hotel in Kyoto for 2 nights", generating: HotelPick.self)
```

⚠️ **Security: never put untrusted content in Instructions** AFM models like `SystemLanguageModel` are trained to trust `Instructions` as higher authority than anything arriving in a `Prompt`.

**`Instructions`**: corresponds to the "system prompt" in other LLM frameworks: add `Instructions` to tell the model how it should handle an incoming prompt. `SystemLanguageModel` works best when you give the model a role with clearly stated abilities at the start of `Instructions`:

```swift
let instructions = Instructions {
    """
    You are a friendly travel assistant with tools to access and manage a user's bookings.\
    Help the user plan out their trip or answer questions about their bookings.
    """
}
```

- Refer to the model as "You" and default to the word "user" to describe whom will be sending prompts to the session (even if it's not the app user), since most LLMs expect this format.
- Try to mention tools if the session will include tool-calling, to help the model understand your intended tool usage policy.
- If no `Instructions` are given to a `LanguageModelSession` it will fall back on a model's default instructions.

**`Prompt`**: corresponds to the "user prompt" in other LLM frameworks, and is your primary input to the model.

### Streaming

Prefer `streamResponse` for content that should animate in for UI, and prefer `respond` for non-UI tasks and background Swift tasks. Content streams in as snapshots: partial text for String generation and optional properties for Generable generation:

```swift
let session = LanguageModelSession()

for try await update in session.streamResponse(
    to: "Generate a list of suggested search terms for a trip planner.",
    generating: SearchSuggestions.self
) {
    withAnimation(.bouncy) {
        // Overwrite the UI state to update with the latest snapshot
        searchTerms = update.content.searchTerms ?? []
    }
}
```

Each snapshot holds everything generated so far, so replace UI state on every snapshot instead of appending to it. Elements of a streamed array are `PartiallyGenerated` too, so their properties are optional:

```swift
for try await snapshot in session.streamResponse(to: notes, generating: CarePlan.self) {
    // Replace, don't append: the snapshot already includes earlier elements
    rows = (snapshot.content.actions ?? []).compactMap { action -> Row? in
        guard let title = action.title else { return nil }
        return Row(title: title, detail: action.detail ?? "")
    }
}
```

### Error handling

⚠️ **GenerationError is deprecated**: Use `LanguageModelError` instead.

Why? `GenerationError` has been replaced with `LanguageModelError` as of 27.0+ targets. The error cases are very similar between `LanguageModelSession.GenerationError` and `LanguageModelError` but the base error type was changed in 27.0 to change error ownership from the session _to_ the model. This was needed because 27.0 introduced the ability to run many LLMs (not just `SystemLanguageModel`), and different models have different error conditions.

- The new error type `LanguageModelError` has a few base error cases and more model-specific error cases. Know which model(s) your session will use.
- `LanguageModelError` is nonexhaustive, so a `switch` over it needs a `default`. Don't invent cases; these are the ones that exist:

  ```swift
  do {
      // ... call try await `respond` or `streamResponse`
  } catch let error as LanguageModelError {
      switch error {
      case .contextSizeExceeded: message = "That's too long. Try a shorter request."
      case .guardrailViolation, .refusal: message = "This request can't be handled."
      case .rateLimited: message = "Too many requests. Try again shortly."
      case .unsupportedLanguageOrLocale: message = "This language isn't supported."
      case .timeout: message = "The request timed out."
      default: message = error.localizedDescription
      }
  }
  ```

- If the app target still includes 26.0+ targets, you'll need to handle both error types and ignore the deprecation warning to keep legacy support:

  ```swift
  do {
      // ... call try await `respond` or `streamResponse`
  } catch {
      if #available(iOS 27.0, macOS 27.0, visionOS 27.0, *) {
          if let error = error as? LanguageModelError {
              // New error handling logic
          } else {
              // Rethrow
          }
      } else {
          // Legacy support: this will produce a deprecation warning
          if let error = error as? LanguageModelSession.GenerationError {
              // Pre-existing error handling logic
          } else {
              // Rethrow
          }
      }
  }
  ```

# References

Fundamental topics:

- `references/apple-models.md`: Use when checking availability, choosing SystemLanguageModel vs PCC, entitlement/quotaUse, and performance best-practices.
- `references/generable.md`: Use for generating structured output. Covers performance, `@Generable` and `@Guide` best-practices, `GeneratedContent`, `GenerationSchema`, and `DynamicGenerationSchema`

Only load a reference below when its topic is actually in play.

- `references/tool-calling.md`: Use for defining a `Tool` for a session to call. Covers static tools vs. runtime-supplied dynamic tools.(`GeneratedContent` arguments plus a hand-built `GenerationSchema`), hiding a tool's schema from instructions with `includesSchemaInInstructions`, and how tool calls and outputs land in the transcript. Read `references/generable.md` first.
- `references/agentic-experiences.md`: _(27.0 min target)_ Use when building a session whose instructions, tools, or model change with app state. Covers the `DynamicProfile` / `DynamicInstructions` / `Profile` layering, state and session lifecycle hooks, and when to reach for dynamic profiles versus fixed instructions. Read `references/tool-calling.md` first.
- `references/custom-models.md`: _(27.0 min target)_ Use whenever the model isn't one of Apple's — a server-side or third-party LLM, a vendor's Swift SDK, or your own weights running on device — including when an app currently calls an LLM's REST API by hand with `URLSession`. Covers the out-of-the-box paths as well as fully custom paths.