---
REP Number: '0020'
Author: levimatheri (Levi Muriuki)
Start Date: 2025-12-08
Feature Status: Public
---

# Inline bicepconfig overrides via CLI 

## Summary

Introduce ability to override bicepconfig settings inline for the Bicep CLI commands

## Motivation

Internally, we're working on enhancing external input experience by introducing a plugin-based protocol to help shift errors left and provide users with richer type validation when authoring and when validating their Microsoft-internal deployment artifacts.
There are two distinct scenarios which we need to support:
1. Validation during authoring in VSCode
2. Validation when invoking Microsoft-internal tooling SDK/cmdlets which will internally invoke the Bicep CLI to compile bicep artifacts

For scenario (1), users will provide validation parameters in their bicepconfig.json that the plugin requires to successfully resolve an external input,
```json5
{
    "externalInputResolverConfig": {
        "foo.*": { // external input kind
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
For scenario (2), users will provide the validation parameters via the Microsoft-internal SDK/cmdlet, like so:
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

This spec aims to introduce an override mechanism for bicepconfig.json, and by so doing, the Microsoft-internal cmdlet/SDK would be able to override the `externalInputResolverConfig` parameters when invoking the Bicep CLI with the parameters provided by the user to the cmdlet/SDK.

## Detailed design

### Syntax
Introduce an additional parameter to the CLI commands `--config-override` that takes in a JSON object:

**Example:**
```sh
bicep build --config-override '{"analyzers":{"core":{"enabled":false}},"experimentalFeaturesEnable":{"foo":true}}'
```

The `--config-override` flag should also accept a JSON file (Note, this file will be for specifying the overrides and NOT as a full-replacement of the bicepconfig as in [this feature request](https://github.com/Azure/bicep/issues/5013))

**Example:**

_configOverrides.json_:
```json5
{
  "analyzers": {
    "core": {
      "enabled": false
    }
  },
  "experimentalFeaturesEnabled": {
    "foo": true
  }
}
```

_Bicep command_:
```sh
bicep build --config-override ./configOverrides.json
```

There should only be **one** `--config-override` flag to avoid complication about which one is considered.

### Override behavior

The order of precedence will be:
  1. CLI override (highest)
  2. bicepconfig.json
  3. Default config (lowest)

To keep things simple at start, we would implement the existing override logic that happens when Bicep reads the user's bicepconfig.json and overlays it with the default bicepconfig. This would be sufficient to support our usecase in the Microsoft-internal tooling, i.e.:

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
2. Difficult to determine "effective configuration" when troubleshooting
3. No schema validation/intellisense for overrides unlike when editing bicepconfig.json files in VSCode

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
    --config-override externalInputResolverConfig.foo.*.target=/path/to/plugin \
    --config-override externalInputResolverConfig.foo.*.parameters.serviceGroupRoot=/path/to/SGR \
    --config-override externalInputResolverConfig.foo.*.parameters.rolloutSpecs[0]="RolloutSpec.json" \
    --config-override externalInputResolverConfig.foo.*.parameters.servicePresence[0].rolloutInfra=prod \
    --config-override externalInputResolverConfig.foo.*.parameters.servicePresence[0].regions[0]=australiaeast \
    --config-override externalInputResolverConfig.foo.*.parameters.servicePresence[0].regions[1]=westus
  
  # Compare with JSON syntax (more readable):
  bicep build --config-override '{
    "externalInputResolverConfig": {
      "foo.*": {
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

1. :white_check_mark: Should we implement the more complex JSON-PATCH (RFC 6902) or directive-based override syntax _initially_ or as follow-up? These would likely take time to iron out the design and potentially delay the Microsoft-internal tooling authoring design
    - Implement the simple approach to start with. The more advanced approaches e.g. JSON PATCH is not straightforward to use directly by users (usually it's generated by tooling)
    - If we need in the future to enable the more advanced features, we could allow explicit opt-in via a flag e.g. by adding a `--config-override-type "json-patch"` similar to how this is handled [in kubectl](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_patch/#options). This would allow us to keep the simple override behavior as default

2. :white_check_mark: Should malformed JSON in `--config-override` or unrecognized config properties in overrides by cause errors, emit warnings, or ignored?
    - Malformed or unknown properties should cause result in an error. Users may not pay attention to warnings
3. :white_check_mark: Should we include a `--show-effective-config` flag or similar to help users troubleshoot what configuration is actually being applied after overrides?
    - This is useful; we should include it - can show effective config in the output if the flag is provided