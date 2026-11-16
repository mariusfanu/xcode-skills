# Build agentic experiences

`LanguageModelSession` by default runs on a fixed set of instructions and tools, and its transcript is append-only by default. For more complex AI workflows, or to compose an agentic harness (the software infrastructure surrounding a model that turns raw model reasoning into reliable, multi-step action), `LanguageModelSession.DynamicProfile` lets you update instructions, tools, models or the transcript on-the-fly. Use to orchestrate a workflow, switch tone when someone flips a setting, offer different tools while they’re editing an image, or load context and tools after launch.

## Consider fixed instructions and tools first

**⚠️ Performance:** `DynamicProfile` APIs are flexible, but frequent changes can invalidate the KV cache, increasing latency and cost. For progressive disclosure, prefer using a tool to fetch information over changing instructions. Appending new information as tool output or prompt to the transcript is _much cheaper_ than appending new info to the `Instructions`.

Prefer static `Instructions` and tools when behavior and capabilities remain stable.

Use `DynamicProfile` when:

- app state meaningfully changes what the model should know, which tools it should consider, or how it should respond.
- app relies on handoff between different models due to privacy or cost considerations.
- app relies on the lifecycle hooks exposed as modifiers.

When dynamic context is necessary, keep the stable portion of the configuration unchanged and place conditional instructions and tools toward the _end_ of the builder, to help preserve the KV cache.

## DynamicProfile essentials

The `DynamicProfile` API family turns a static `LanguageModelSession` into a builder:

```swift
// Some dynamic app state that changes in a UI somewhere
@Observable
final class AssistantSettings {
    var likesBrevity = true
}

struct TravelProfile: LanguageModelSession.DynamicProfile {
    let tripName: String  // not observable, fixed-value state
    let settings: AssistantSettings  // observable, dynamic state

    var body: some LanguageModelSession.DynamicProfile {
        // A nested DynamicProfile is aliased as just "Profile"
        Profile {
            // Inside the builder is of type DynamicInstructions

            // Instructions auto-update when the `settings` property changes
            Instructions {
                """
                You are a travel assistant. When a user asks for your help for their trip \
                "\(tripName)", use available tools to get the information they need, then \
                \(settings.likesBrevity ? "answer in one sentence." : "explain thoroughly.")
                """
            }

            // Use if/conditional statements to swap out tools as needed
            FindHotels(tripName: tripName)
            // ... other tools here
        }
        .onPrompt { prompt in  // Example of a lifecycle modifier
            print("Model received: \(prompt)")
        }
        // .model(settings.model) use modifier to make model choice dynamic
    }
}
```

Now initialize a `LanguageModelSession` with `profile` to let your `DynamicProfile` run the session:

```swift
let session = LanguageModelSession(profile: TravelProfile(tripName: tripName, settings: userSettings))
```

The session will re-evaluate the `body` before each turn and each time an `@Observable` property (including `@SessionProperty`) changes, so everything stays up-to-date.

In the simple cases, use a value type (e.g. `let tripName: String`) for static configuration that's fixed for the whole session, and use an `@Observable` class for dynamic app state the session should react to as it changes. Prefer mutating `@Observable` data inside a lifecycle hook (`onPrompt`, `onResponse`, `onActivate`, …) or a tool’s `call` because the session is guaranteed _not_ to run inference during either, so re-evaluation of the body always completes before the next prompt reaches the model.

## Use @SessionProperty for state owned/mutated by the session

Whereas the session _reacts to_ `@Observable` state owned by the app,
state _owned by_ a session should use the session's own `@SessionProperty` to make values mutable and observable. The built-in `\.history` lets you directly manage the session transcript, for example, to compact it after a response to stay within the context window:

```swift
struct CompactingProfile: LanguageModelSession.DynamicProfile {
    @SessionProperty(\.history) var history

    var body: some LanguageModelSession.DynamicProfile {
        Profile { Instructions("You are a helpful assistant.") }
        .onResponse {
            if history.count > 0 {
                history = history.suffix(50)
            }
        }
    }
}
```

Mutating a `@SessionProperty` causes the session to immediately update. Mutating `transcript` outside a lifecycle hook or tool `call` while a response is in progress makes that response throw `LanguageModelSession.Error.transcriptMutationWhileResponding`.

Trimming the stored transcript directly (as `CompactingProfile` does above) changes the prefix the model reuses and can invalidate its KV cache; prefer `.historyTransform` when you don’t need to mutate the stored history. It does a per-request trim, shaping only what this one request sees.

Define your own key with `@SessionPropertyEntry`, then read it with `@SessionProperty` inside `DynamicProfile`, `DynamicInstructions`, or `Tool`.

```swift
extension SessionPropertyValues {
    @SessionPropertyEntry var activatedSkills: [String: Bool] = [:]
}

// Add this line inside of a Tool or a DynamicProfile to access the session property
@SessionProperty(\.activatedSkills) var activatedSkills
```

### Your UI can observe properties too

Observation works in both directions. `session.properties` is itself observable, so a SwiftUI view that reads a property from it re-renders whenever that property changes, whether a tool updated it or the profile did:

```swift
// A view that reads session.properties re-renders when that state changes.
// (todos is a custom @SessionPropertyEntry, declared like `activatedSkills` above.)
ForEach(session.properties.todos, id: \.content) { todo in
    Text(todo)
}
```

## Use lifecycle modifiers

Lifecycle modifiers let you run code at specific points during a session. They come in two kinds:

1. **Turn phases** fire around a single `respond(to:)` call, in this order: `onPrompt` (before) → model runs → `onReasoning` (as thoughts arrive) → `onToolCall` (per call) → `onToolOutput` (per result) → `onResponse` (after). These are `async throws` since a thrown error propagates out of the caller’s `respond`.

2. **Activation** fires as a configuration enters or leaves the resolved state: `onActivate` when it enters the resolved config, `onDeactivate` when it leaves. These are `async` and don't throw.

Note any mutating closure in a modifier (e.g. updating app state inside `onPrompt`) needs to be sendable or you'll get a Swift concurrency error at build time.

### Control flow usage patterns

- `.onActivate` fires when this profile/instructions first becomes live (incl. session creation). It's great for loading expensive context or running setup tasks:

  ```swift
  struct AsyncContextProfile: LanguageModelSession.DynamicProfile {
      @SessionProperty(\.loadedContext) var loadedContext
      let loader: ContextLoader
      let appPage: AppPage  // @Observable app state

      var body: some LanguageModelSession.DynamicProfile {
          // Use an enum + switch to change which Profile is driving the session
          switch appPage {
          case .orderHistory:
              Profile {
                  if loadedContext.isEmpty {
                      Instructions("Context is still loading. Ask the user to wait a moment.")
                  } else {
                      Instructions {
                          "Use this context to answer:"
                          loadedContext
                      }
                  }

              }
              .onActivate {
                  Task(priority: TaskPriority.userInitiated) {
                      loadedContext = await loader.load()
                  }
              }
          case .inventorySearch:
              Profile {
                  // ... some other Profile
              }
          }
      }
  }

  @Observable
  @MainActor
  final class ContextLoader {
      func load() -> String {
          return "... context..."
      }
  }

  enum AppPage {
      case orderHistory
      case inventorySearch
  }

  extension SessionPropertyValues {
      @SessionPropertyEntry var loadedContext = ""
  }
  ```

  Similarly `.onDeactivate` fires when the Profile stops being live. It's great for teardown tasks.

- `.onPrompt { prompt in }` is useful for a “look at the prompt, then specialize” pattern: read the incoming prompt, update your state, and the profile reactively swaps in the right instructions before the model runs.

  ```swift
  Profile { Instructions(specialist(for: skim.topic)) }
  .onPrompt { prompt in
      await skim.classify(prompt) // sets skim.topic
  }
  ```

- `.onResponse { response in }` runs after the model responds. It's particularly useful for recording turn counts or compacting the transcript.

- `.onToolCall { call in }` / `.onToolOutput { call, output in }` runs when tool is invoked / returns. Great for tool telemetry and side effects.

### Debug Tips

All the lifecycle modifiers are great for logging model activity, so the developer can watch what the model is doing live and spot where an issue occurs. You can also create a general-purpose logging modifier and add it to all `Profile` in the session:

```Swift
extension LanguageModelSession.DynamicProfile {
    func debugLog() -> some LanguageModelSession.DynamicProfile {
        return onPrompt { prompt in
            Logging.general.log("Prompt: \(prompt)")
        }
        .onToolCall { toolCall in
            Logging.general.log("Tool call: \(toolCall)")
        }
        .onResponse { response in
            Logging.general.log("Response: \(response)")
        }
    }
}
```

#### Tool usage debugging

To fix runaway tool-calls or to restrict how many times the model is allowed to call a tool:

1. Make a tool counter observable state, and update the tool counter in a `toolCall` (to budget _all_ tool calls) or inside of a given Tool instance (to budget for a specific tool)
2. Turn OFF the model's ability to call a tool when the counter reaches the desired budget using this pattern.

   ```swift
   extension SessionPropertyValues {
       @SessionPropertyEntry var toolCallCount = 0
   }

   struct SomeProfile: LanguageModelSession.DynamicProfile {
       @SessionProperty(\.toolCallCount) var toolCallCount

       var body: some LanguageModelSession.DynamicProfile {
           Profile {
               // ...
           }
           .toolCallingMode(toolCallCount >= 5 ? .disallowed : .allowed)
           .onToolCall {
               // increment count
               toolCallCount += 1
           }
           .onResponse {
               // reset count on the model response IF the budget is per-turn
               toolCallCount = 0
           }
       }
   }
   ```

   to gate a specific tool instead use:

   ```swift
   .onToolCall { toolCall in
       if toolCall.toolName == ToolA().name {
           // increment count
           toolCallCount += 1
       }
   }
   ```

You can also use this strategy to force the model to call tools, using `.toolCallingMode(.required)`
