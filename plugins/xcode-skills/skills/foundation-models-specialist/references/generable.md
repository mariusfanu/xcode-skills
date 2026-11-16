# @Generable types for structured output

`@Generable` macro lets an LLM directly generate the decorated Swift struct or enum, by automatically applying type conformance to the `Generable` protocol. Conforming types are informally called "generable" or "generables". Generables are the powerful bread-and-butter of this framework since they automatically handle constrained decoding and JSON parsing of raw model output to produce strongly-typed instances of Swift structs or enums.

**Prefer creating a `Generable` type using the `@Generable` macro, especially for POD types.** Macro provides an implementation of the `Generable` protocol that handles some complex details and guarantees type-safety.

## Property names and order guide the model

Generable types can be structs or enums and are nestable. Generables are automatically fed to the model as a JSON schema, so the property **names** and **order** of properties is the model's primary semantic signal. structure = semantics + order

A simple example:

```swift
@Generable
struct CatProfile {
    let name: String
    let age: Int
    let profile: String
}
```

- You'll get the most accurate results for the least tokens by making your property names as precise as possible, and always remember the model sees them _together_. So above uses `name` instead of `catName` because adding the "cat" prefix would be redundant: the model will see "CatProfile" too and contextualize.
- Avoid ambiguous names. For example a `@Generable` type `Document` a property named `title` is too vague, and you'll end up using a lot of `Prompt` or `@Guide` tokens trying to steer the model towards what you mean. Using `fileName` or `displayTitle` or `userName` are far more precise because they tell the model the _context_ of use for "title" in very few tokens, and the model is more likely to generate a title that _looks like_ your intended format reliably.

Generate using:

```swift
let response = try await LanguageModelSession().respond(to: "Generate a cute cat", generating: CatProfile.self)
let cat = response.content // Value of type CatProfile
```

You can look under-the-hood if needed to understand what exactly the model is seeing and producing before the framework handles the type-translation to a CatProfile instance. First, when `.respond(to:, generating:)` is called above, `CatProfile` is converted to a JSON schema and added to the prompt. You can take a look for debug purposes by running:

```swift
let typeSchema = CatProfile.generationSchema.debugDescription
```

Then you can see what the model produced for `cat` from the session example above by running:

```swift
let rawOutput = cat.generatedContent.debugDescription
```

which outputs:

```
{"profile":"A cute little cat with soft fur and big eyes.","age":3,"name":"Fluffy"}
```

Finally, if needed you can get the model-specific token-cost using:

```
let model = SystemLanguageModel()
let tokenCost = try await model.tokenCount(for: CatProfile.generationSchema)
```

Generally, the more complicated, long, or nested the generable types, the more token and task pressure you put on the model. So adjust your generable types to match your target model. As a rule-of-thumb, 6-ish max root properties and max 3-depth of type nesting is suitable for `SystemLanguageModel`. If a property is a list like `[String]` or `[MyOtherGenerableType]` account for that complexity with fewer root properties... and reverse if there's no collections you can afford more root properties.

## Use enums for a fixed set of options

A generable `enum` is a powerful tool because you **do not** have to describe it in your `Prompt` or `@Guide` and the model will know all the options, with strong type enforcement from constrained decoding to ensure the model produces only valid cases. For example:

```swift
@Generable
enum MovieGenre {
    case comedy
    case horror
    case romance
    case action
    case kids
}

@Generable
struct MovieNight {
    let theme: String
    let genres: [MovieGenre]
}
```

prompted with:

```swift
let response = try await LanguageModelSession().respond(generating: MovieNight.self) {
    "Help me plan a movie night for my knitting club"
}
```

produces a `MovieNight` object with the properties: `{"theme":"Cozy Night In","genres":["romance","comedy"]}`. Note the model inferred the theme and appropriate genres without any additional prompting: `MovieGenre` and `MovieNight` do the heavy-lifting.

## Best-practices for using @Guide

While a generable type's structure carries the most meaning, `@Guide` and descriptions help you steer the model's tone, clarify what you want for property, and set limits on how much of a property you want.

**Keep it short and precise**: Your `description` should be short (<10-ish words) direct noun phrases that give keywords to steer length ("basic", "one sentence") and keywords for style/tone ("social media", "victorian gothic"). Remember structure = semantics + order, so when the model reads "cat's personality". Don't waste tokens to spell-out what keywords means: most LLMs can very easily infer style from keywords.

**Note for small models**: small or on-device LLMs typically have less context to work with and less ability to reason. It's especially important to keep a `description` short and precise for these models, since they more often struggle to accurately following long or more complex descriptions.

### Using `@Generable(description:)` and `@Guide(description:)`

Since a `@Generable(description:)` or `@Guide(description:)` is more of a "nudge" and costs tokens, try writing your generable without them to start, see what results you get, and add in descriptions to refine. Keep in mind descriptions are "fine tuning", so if the generable output is truly far from your intention it will be way more effective to iterate on the generable structure _first_. The cheapest and most effective change is to rename a property to a clearer name.

```swift
@Generable(description: "A fun basic profile suitable for social media")
struct CatProfile {
    var name: String
    var age: Int

    @Guide(description: "A one sentence profile about the cat's personality")
    var profile: String
}
```

- `@Generable(description:)` sets the overall style. You could do that in the session `Prompt` as well, so prefer the `description` approach only if you need to re-use `CatProfile` in multiple sessions that have different prompts.

- Note `name` and `age` don't need a `@Guide(description:)` because they have self-explanatory names and we can trust most any LLM to know typical cat names and ages. Not using a `@Guide` here is a great way to save tokens.

- `@Guide(description:)` is used for `profile` because in the generation run above we saw the model generated _"A cute little cat with soft fur and big eyes"_ when the user actually wanted the profile to focus on personality. This is a great use of `@Guide(description:)` for clarification.

### Working with other LLM providers

Schemas with some certain shapes don't always work with all LLM providers. Common incompatibility and workarounds:

- Top-level enums are not allowed: Nest the enum in a top level struct.
- Absent properties are not allowed as nullable values: Use `@Generable(representNilExplicitlyInGeneratedContent: true)`
- Arbitrary nested shape of `@Guide`s are not supported: Use natural language description to give hints instead.

### Using other `@Guide` options

Generally the count-related `@Guide` options are the most effective. `@Guide` can have multiple options like this:

```swift
@Generable
struct FortuneCookie {
    var fortuneText: String

    @Guide(description: "More stars means a stronger fortune", .maximum(5))
    var starCount: Int

    @Guide(.element(.range(0...99)), .count(4))
    var luckyNumbers: [Int]
}
```

Use `key` to change the property name the model generates without renaming the Swift property:

```swift
@Guide(key: "number_of_stars", description: "More stars means a stronger fortune", .maximum(5))
var starCount: Int
```

In this example `element`, `range`, and `count` are composed together to enforce exactly 4 lucky numbers between 0 and 99. Note we don't need a `description` because `FortuneCookie` and `luckyNumbers` are sufficient context clues. For `starCount` here `maximum` is used, but you could use `.range` instead to more effectively communicate the lower bound. The description explains the direction and semantic meaning of the `starCount` score, which is a great use of description to tune the model to your expected scale.

Notes:

- `.pattern` works with limited subset of regexes. Workaround: provide natural language description, regex literal, or give examples in `@Guide(description:)`.
- `.anyOf` is great for choosing runtime options:
    ```swift
    @Guide(.element(.anyOf(OptionsProvider.shared.options)))
    var options: [String]
    ```
- `.constant` is equivalent to immutable swift properties: `let age = 5`.

## Advanced: DynamicGenerationSchema

Think of `DynamicGenerationSchema` as dynamic type creation: reach for it when you can't a generable as a static types at compile time. This commonly happens in database scenarios where you don't really know what properties need to be generated until runtime.

For example, imagine in a restaurant game you want the model to generate a customer's order from the meal-set the player created:

```swift
// Creates the schema at runtime.
let mealSetOrderSchema = DynamicGenerationSchema(
    name: "MealSetOrder",
    properties: [
        DynamicGenerationSchema.Property(
            name: "dailySoup",
            schema: DynamicGenerationSchema(
                name: "dailySoup",
                anyOf: ["Tomato", "Chicken Noodle", "Clam Chowder"]
            )
        )

        // ... additional properties in the meal
    ]
)
```

Once you've built the schema, compile it and pass it to `schema:` instead of `generating:`, which is for static generable types only:

```swift
// Compilation throws if your schema is missing pieces or is invalid
let compiled = try GenerationSchema(root: mealSetOrderSchema, dependencies: [])

let response = try await LanguageModelSession().respond(to: "Generate an order for a hungry student", schema: compiled)
```

Now getting the output is a little tricky, since you need to query 1 property at a time:

```swift
// Produces a raw GeneratedContent since there's no static type to convert it to.
let meal = response.content

// Use value to pull out properties 1-by-1 in your desired type.
let soup = try meal.value(String.self, forProperty: "dailySoup")
```
