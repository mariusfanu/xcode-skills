# Apple Foundation Models in the framework

`SystemLanguageModel` is the default model of the framework, also known as the Apple Intelligence on-device model. `SystemLanguageModel` basically means "the system's LLM" which can actually be a different AFM model depending on the OS and hardware of an individual device.

- `SystemLanguageModel()` is identical to `SystemLanguageModel.default`
- On 27.0+ targets use `SystemLanguageModel.default.variant` to identify the exact model.
- Try the model without `GenerationOptions` first, since temperature/sampling have less effect steering model behavior than prompting.

`PrivateCloudComputeLanguageModel` is only available to developers who have been approved for a special entitlement. `PrivateCloudComputeLanguageModel` refers to Apple Intelligence running on Private Cloud Compute (PCC), which means it's a server model with strong privacy guarantees. `PrivateCloudComputeLanguageModel`'s nickname is "PCC model" or `pccModel`, and it comes with per-app and per-user daily usage limits.

**Check model availability before starting a session**: `PrivateCloudComputeLanguageModel` or `SystemLanguageModel` model availability can change while the app is running. For `SystemLanguageModel` use:

```swift
let model = SystemLanguageModel()

switch model.availability {
case .available:
    print("The model is available and ready to use.")
case .unavailable(let reason):
    print(reason)
}
```

**Check language support**: use `model.supportsLocale()` (defaults to `Locale.current`) or `model.supportedLanguages`. Output quality can vary across supported languages, so test with the languages your users actually use.

The PCC model availability API has the same shape with different unavailability reasons.

- Note: Availability doesn't check the `com.apple.developer.private-cloud-compute` entitlement, but sending a request without it is a precondition failure.

```swift
let pccModel = PrivateCloudComputeLanguageModel()

switch pccModel.availability {
case .available:
    print("PCC model is available and ready to use. Check quota next.")
case .unavailable(let reason):
    print(reason)
}
```
