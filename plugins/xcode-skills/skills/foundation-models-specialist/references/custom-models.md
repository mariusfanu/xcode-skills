# Using a non-Apple model with FoundationModels

The framework is not limited to Apple's own models. Other LLMs can be integrated as a `LanguageModel` protocol conformance, and from the call site it then looks exactly like `SystemLanguageModel`: the same `respond`/`streamResponse`, the same `@Generable` output, the same tools, the same
transcript.

| If your model is…                                                      | Reach for                                                                                                              | Notes                                                                                                                                                                                                                     |
| ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A **custom on-device model** (your own weights, running locally)       | [**CoreAI**](https://github.com/apple/coreai-models) (preferred) or [**MLX**](https://github.com/ml-explore/mlx-swift-lm) (backup) | Both ship ready-made `LanguageModel` conformances as Swift packages: `CoreAILanguageModel` (`CoreAILanguageModels`) and `MLXLanguageModel` (`MLXFoundationModels`).                                                                                                                                                      |
| Behind an **OpenAI-compatible `/chat/completions` endpoint**           | `ChatCompletionsLanguageModel`, from [FoundationModelsUtilities](https://github.com/apple/foundation-models-utilities) | Construct it with your URL and `additionalHeaders` and you're done — no conformance to write.                                                                                                                             |
| Served by a **major provider** (Anthropic, Google, others)             | That vendor's own Foundation Models package                                                                            | Several providers ship and maintain a `LanguageModel` conformance for their own API. Check the vendor's SDK documentation or GitHub org before building a parallel one.                                                   |
| _Not supported by any of the above_                                    | Your own `LanguageModel` conformance                                                                                   | Write your own when the existing options don't fit for any reason: missing feature support, a different API, an unsupported auth handshake, a non-HTTP transport, or custom content types.                            |

**`ChatCompletionsLanguageModel` needs the [foundation-models-utilities](https://github.com/apple/foundation-models-utilities) package.** Add the `FoundationModelsUtilities` product to the target before writing `import FoundationModelsUtilities`, otherwise the import won't resolve. If you can't edit the project's dependencies, tell the user to add the package:

```swift
import FoundationModelsUtilities

let model = ChatCompletionsLanguageModel(
    name: "your-model-name",
    url: URL(string: "https://api.example.com")!,
    additionalHeaders: ["Authorization": "Bearer \(apiKey)"]
)
let session = LanguageModelSession(model: model)
```

**Some LLM vendors provide SDKs for Foundation Models integrations.**
These SDKs usually come with a custom `LanguageModel`. You can use it with `LanguageModelSession`, `respond`/`streamResponse`, `@Generable` output, tools, transcript. Every other reference in this skill applies to custom language models.

**When you need direct access to low level operations in `LanguageModel`, consider CoreAI and MLX integrations first.** CoreAI and MLX already solve the hard parts of local inference: model loading, memory residency and eviction, compute placement, tokenization, KV-cache reuse, and expose the result through this same protocol. Using these packages simplifies your code and delivers great performance.
