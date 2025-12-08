---
REP Number: <Fill me in with a four-digit number matching the pull request number; Update AFTER PR is approved and BEFORE is merged.>
Author: levimatheri (Levi Muriuki)
Start Date: 2025-12-08
---

# Inline bicepconfig overrides via CLI 

## Summary

Introduce ability to override bicepconfig settings inline for the Bicep CLI commands

## Motivation

Internally, we're working on enhancing external input experience by introducing a plugin-based protocol to help shift errors left and provide users with richer type validation when authoring and when validating their Microsoft-internal deployment artifacts.
There are two distinct scenarios which we need to support:
1. Validation during authoring in VSCode
2. Validation when invoking SDK/cmdlets which will internally invoke the Bicep CLI to compile bicep artifacts

For scenario (1), users will provide validation parameters in their bicepconfig.json that the plugin requires to successfully resolve an external input,
```json5
{
    "externalInputResolverConfig": {
        "ev2.*": { // external input kind
            "target": "path-to-plugin-binary",
            "parameters": { // parameters to be sent over to the plugin
                "serviceGroupRoot": "/path/to/SGR",
                "rolloutSpecs": [ "**/*/RolloutSpec.json" ],
                "servicePresence": [
                    {
                        "rolloutInfra": "test",
                        "regions": [ "eastus", "westus" ]
                    }
                ]
            }
        }
    }
}
```
For scenario (2), users will provide the validation parameters via the SDK/cmdlet, like so:
```ps1
Test-AzureArtifacts `
    -ServiceGroupRoot "/path/to/SGR"
    -RolloutSpec "RolloutSpec.json"
    -RolloutInfra "prod"
    -Select "regions(australiaeast)"
    -ConfigurationOverrides '{"Settings": { "ReleaseId": "release-a.b.c" } }'
```

The implication here is that in scenario (2), the parameters provided via the cmdlet/SDK are likely to be different from the parameters provided in the bicepconfig.json. If the cmdlet-provided-parameters are not taken into account, this could cause confusion when we emit validation diagnostics.

Additionally, there are parameters such as `ConfigurationOverrides` that only make sense to be provided via the cmdlet/SDK and not on the bicepconfig.json, since they are typically provided in a CI pipeline context. Such parameters should similarly be taken into account when performing external input validation.

This spec aims to introduce an override mechanism for bicepconfig.json, and by so doing, the cmdlet/SDK would be able to override the `externalInputResolverConfig` parameters when invoking the Bicep CLI with the parameters provided by the user to the cmdlet/SDK.

## Detailed design

### Syntax
Introduce an additional parameter to the CLI commands `--config-override` that takes in a JSON object:

**Example:**
```sh
bicep build --config-override '{"analyzers":{"core":{"enabled":false}},"experimentalFeatures":["lambda"]}'
```

### Override behavior
To keep things simple at start, we would implement the existing override logic that happens when Bicep reads the user's bicepconfig.json and overlays it with the default bicepconfig. This would be sufficient to support our usecase in Ev2, i.e.:

**Primitives:** 
Fully replaced by override
```jsonc
// Original
{ "foo": "bar" }
// Override
{ "foo": "baz" }
// Result
{ "foo": "baz" }
```

**Objects:**
Merged recursively by key; override replaces keys if present
```jsonc
// Original
{
  "settings": {
    "foo": { "enabled": true, "options": { "bar": "value1", "baz": "value2" } }
  }
}
// Override
{
  "settings": {
    "foo": { "enabled": false }
  }
}
// Result
{
  "settings": {
    "foo": { "enabled": false, "options": { "bar": "value1", "baz": "value2" } }
  }
}
```

**Arrays:**
Replaced entirely (including arrays of objects)
```jsonc
// Original
{ "items": ["foo", "bar"] }
// Override
{ "items": ["baz"] }
// Result
{ "items": ["baz"] }
```

As a follow-up if needed to support more complex usecases, we can introduce a way for users to provide their intent of how the override should behave. To do this, some options include:

- Provide overrides as [JSON-PATCH RFC 6902](https://www.rfc-editor.org/rfc/rfc6902) operations, i.e.
    ```jsonc
    [
        { "op": "replace", "path": "/baz", "value": "boo" },
        { "op": "add", "path": "/hello", "value": ["world"] },
        { "op": "add", "path": "/contacts/-", "value": ["John"] },
        { "op": "remove", "path": "/foo" }
    ]
    ```
- Directive based overrides, e.g.
    - Replace object:
    ```jsonc
    // Original
    {
        "settings": {
            "foo": { "enabled": true, "options": { "bar": "value1", "baz": "value2" } }
        }
    }

    // Override
    {
        "settings": {
            "foo": {
                "$replace": {
                    "enabled": false
                }
            }
        }
    }

    // Result (replaced entirely, options removed)
    {
        "settings": {
            "foo": { "enabled": true }
        }
    }
    ```
    - Merge array of objects by key - similar to `kubectl patch` [strategic merge](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/#use-a-strategic-merge-patch-to-update-a-deployment):
    ```jsonc
    // Original
    {
        "resources": [
            { "name": "foo", "value": "old" },
            { "name": "bar", "value": "xyz" }
        ]
    }

    // Override
    {
        "resources": {
            "$mergeByKey": "name",
            "items": [
                { "name": "foo", "value": "new" },
                { "name": "baz", "value": "abc" }
            ]
        }
    }

    // Result (merged by name key)
    {
        "resources": [
            { "name": "foo", "value": "new" },
            { "name": "bar", "value": "xyz" },
            { "name": "baz", "value": "abc" }
        ]
    }
    ```
### Which bicepconfig files are overridden?
The override would apply to the bicepconfig.json that's closest to the bicep/bicepparam file being compiled, as well as any referenced modules (provided the module does not already have a closer associated bicepconfig.json). 
This is to keep things consistent with the current behavior of bicepconfig [merge process](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/bicep-config#understand-the-merge-process).

## Drawbacks
1. The proposed override behavior would not support more complex/highly targetted override behaviors e.g. merging arrays partially, or replacing entire objects
2. There's overlap with an [existing feature request](https://github.com/Azure/bicep/issues/5013) to support overriding using a bicepconfig.json file path. It's likely users would desire/push for both features to be implemented soon after, which increases implementation cost

## Alternatives

### Dot syntax 
Specify overrides using dotted keys to indicate nested properties, i.e.
```sh
bicep build --config-override analyzers.core.enabled=false
```
**Pros:**
- Compact, convenient for small overrides.

**Cons**:
- Keys that include literal dots (e.g., `"my.setting.with.dots"`) require escaping or special syntax for keys with dots, e.g., `foo[my.setting.with.dots].bar`.
- Nested objects and arrays become verbose for multiple overrides
  ```sh
  # Example: Overriding multiple nested properties becomes unwieldy
  bicep build \
    --config-override externalInputResolverConfig.ev2.*.target=/path/to/plugin \
    --config-override externalInputResolverConfig.ev2.*.parameters.serviceGroupRoot=/path/to/SGR \
    --config-override externalInputResolverConfig.ev2.*.parameters.rolloutSpecs[0]="RolloutSpec.json" \
    --config-override externalInputResolverConfig.ev2.*.parameters.servicePresence[0].rolloutInfra=prod \
    --config-override externalInputResolverConfig.ev2.*.parameters.servicePresence[0].regions[0]=australiaeast \
    --config-override externalInputResolverConfig.ev2.*.parameters.servicePresence[0].regions[1]=westus
  
  # Compare with JSON syntax (more readable):
  bicep build --config-override '{
    "externalInputResolverConfig": {
      "ev2.*": {
        "target": "/path/to/plugin",
        "parameters": {
          "serviceGroupRoot": "/path/to/SGR",
          "rolloutSpecs": ["**/*/RolloutSpec.json"],
          "servicePresence": [{
            "rolloutInfra": "test",
            "regions": ["eastus", "westus"]
          }]
        }
      }
    }
  }'
  ```

## Rollout plan

The usual experimental feature flagging (via bicepconfig) doesn't seem applicable for this proposal. Instead, we could print out a warning message stating that this feature is experimental when the commands are run with `--config-override` like we do with experimental commands.

## Unresolved questions

1. Should we implement the more complex JSON-PATCH (RFC 6902) or directive-based override syntax _initially_ or as follow-up? These would likely take time to iron out the design and potentially delay the authoring design
2. How does this feature interact with the planned feature ([#5013](https://github.com/Azure/bicep/issues/5013)) for bicepconfig file path overrides
    - What is the current state of the existing feature request?