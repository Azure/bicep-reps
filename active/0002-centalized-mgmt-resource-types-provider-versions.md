---
REP Number: 0002
Author: asilverman (Ariel Silverman)
Start Date: 2023-12-19
Feature Status: Private Preview
Bicep Issue Number(s): [12202](https://github.com/Azure/bicep/issues/12202)
---

# Title - Centralized management of resource type provider versions using bicepconfig.json

## Summary

This document proposes a feature that enables users to manage Bicep resource type provider versions from a central location. This enhances convenience by sparing the user from making changes in multiple files in order to upgrade a dynamically loaded resource types provider package version.

## Terms and definitions

- **Resource Types Provider Package aka provider package** - The OCI Registry artifact that contains the resource type definitions used by Bicep to identify the resource types properties, functions and other relevant type information that it uses to generate ARM-JSON.
- **types.json protocol** - It is a serialization protocol used by Bicep to encode type information as data that can be loaded to the Bicep runtime.

## Motivation

The [resource type providers dynamic loading feature](https://github.com/Azure/bicep/issues/5453) introduces a change in provider declaration statement syntax to include the registry address and version that Bicep will use to resolve resource type declarations.

**Provider Declaration Syntax**

```bicep
provider 'br:mcr.microsoft.com/bicep/providers/az@1.2.3' // fully-qualified
provider 'br/public:az@1.2.3' // uses provider aliasing defined in bicepconfig.json
```

We also currently support the following syntax for "built-in" providers:

```bicep
provider 'sys@1.0.0'
```

In the above cases, the provider can be referenced by using its symbol which is derived from the segment to the left of the '@' sign. (`az` and `sys` for the declarations above)

Both syntaxes also support aliasing of that symbol reference:

```bicep
provider 'br/public:az@0.2.3' as myAz
provider 'sys@1.0.0' as mySys
```

The current syntax (shown above) presents several challenges:

1. The new syntax is quite verbose, and cannot be controlled centrally. For users with a large set of files, picking up the latest version of `az` will require modifying every single file.
1. The aliasing (assuming that the last segment of the registry namespace == the provider name) & versioning (having an `@` character instead of a `:` which is familiar to module users) behavior of the new syntax can be confusing.
1. Its deceiving and confusing to allow `sys` to have a pinned versionis fixed version `1.0.0` - this **should** imply that there are no changes between Bicep versions, which isn't realistic.

- It's an "either/or" - either users follow the current behavior (`az` imported by default) **or** they're using the new syntax. We don't have a plan for how to combine the two.
- With [Radius](https://github.com/radius-project/radius), it's conceivable that we may want users to **not** have `az` imported by default.

### Proposed Changes

- [Resolution of namespace identifiers by VSCode extension when inspecting published module sources](#resolution-of-namespace-identifiers-by-vscode-extension-when-inspecting-published-module-sources)
- [Wildcard semantics for provider versions in `bicepconfig.json`](#wildcard-semantics-for-provider-versions-in-bicepconfigjson)

## Detailed design

### Client side changes

#### Add a new syntax for the provider import declaration statements: `provider {providerName} [as {optionalAlias}]`

The example below shows a usage of the proposed syntax

```bicep
provider az
provider kubernetes as k8s // uses optional aliasing of the
```

The resolution of the symbols will be resolved from the builtin `bicepconfig.json` under a new section:

```json
{
  // prior bicepconfig.json sections
  // ...

  "providers": {
    "az": {
      // A package reference string (below) gestures Bicep to source the types from the registry address.
      "source": {
        "registry": "mcr.microsoft.com/bicep/providers",
        "version": "0.2.3",
      },
      "isDefaultImport": true // Gestures Bicep to implicitly import the namespace identifier to the global scope
    },
    "kubernetes": {
      // Using the value "builtin" (below) gestures Bicep to source types from the statically compiled NuGet package reference
      "source": {
        "builtin": true
      },
      "isDefaultImport": false
    }
  }
}
```

Since previous provider declaration syntax forms continue to be supported, we introduce the following constraints:

- To ensure consistency with pre-existing handling of `bicepconfig.json` the configuration file closest to the Bicep file in the directory hierarchy is used.
- The `sys` namespace is coupled to the Bicep bits version and cannot be overriden or dynamically uploaded using the mechanism described above

If a user specifies an entry called `sys` in the section above, this will be identified by the json schema as forbidden value. The reason is to prevent someone from overriding this reserved identifier.

The syntax below currently in use to import built-in providers will become invalid and result in a diagnostic as a result of this change

```bicep
provider 'sys@1.0.0'
provider 'kubernetes@1.0.0' as k8s
```

#### Change the delimiter '@' used by the supported provider declaration syntax to ':' to align with module reference syntax and increase consistency

The syntax shown below uses ':' instead of '@' as a delimiter for provider declaration statements

```bicep
provider 'br:mcr.microsoft.com/bicep/providers/az:0.2.3' // notice its NOT using az@0.2.3
provider 'br/public:az:0.2.3' as myAz
```

## Drawbacks

- Costumers may be confused by the configt merge semantics behavior
- Costumers will not be able to tell from inspecting sources alone what is the provider version used to resolve resource types

## Alternatives

- Using a separate file other than `bicepconfig.json` to store resource type provider version information was evaluated but adds complexity to UX as users need to be aware of two separate merge semantics (one for config and one for provider packages). Although techically this solves the problem, it would have a detrimental effect in the ease of use and reasoning of Bicep projects and so it is dismissed as a non starter.

## Rollout plan

This change affects the client side only and is applied only when the pre-existing feature flag `DynamicTypeLoadingEnabled` is set.

## Out of scope

- [Title - Centralized management of resource type provider versions using bicepconfig.json](#title---centralized-management-of-resource-type-provider-versions-using-bicepconfigjson)
  - [Summary](#summary)
  - [Terms and definitions](#terms-and-definitions)
  - [Motivation](#motivation)
    - [Proposed Changes](#proposed-changes)
  - [Detailed design](#detailed-design)
    - [Client side changes](#client-side-changes)
      - [Add a new syntax for the provider import declaration statements: `provider {providerName} [as {optionalAlias}]`](#add-a-new-syntax-for-the-provider-import-declaration-statements-provider-providername-as-optionalalias)
      - [Change the delimiter '@' used by the supported provider declaration syntax to ':' to align with module reference syntax and increase consistency](#change-the-delimiter--used-by-the-supported-provider-declaration-syntax-to--to-align-with-module-reference-syntax-and-increase-consistency)
  - [Drawbacks](#drawbacks)
  - [Alternatives](#alternatives)
  - [Rollout plan](#rollout-plan)
  - [Out of scope](#out-of-scope)
    - [Updating `bicepconfig.json` merge semantics](#updating-bicepconfigjson-merge-semantics)
    - [Resolution of namespace identifiers by VSCode extension when inspecting published module sources](#resolution-of-namespace-identifiers-by-vscode-extension-when-inspecting-published-module-sources)
    - [Wildcard semantics for provider versions in `bicepconfig.json`](#wildcard-semantics-for-provider-versions-in-bicepconfigjson)
    - [Deciding if `sys` is a compile time import or a resource types provider package](#deciding-if-sys-is-a-compile-time-import-or-a-resource-types-provider-package)

### Updating `bicepconfig.json` merge semantics

The proposal declare the provider information in `bicepconfig.json` and as result is ruled by `bicepconfig.json` merge semantics. The merge semantics for `bicepconfig.json` specify that for a given Bicep source file, the configuration file closest to the Bicep file in the directory hierarchy is used. This behavior may not be expected by customers and may cause users that specified multiple config files in their project file-structure to experience consuding behaviors that may include build errors when Bicep attempts to resolve a namespace identifier. This is illustrated by the example below.

**Example: Provider resolution error due as a result of `bicepconfig.json` merge semantics**

Consider a Bicep project under folder `src` with the following files:

- `src/main.bicep`

```bicep
provider foo as mainFooProvider
```

- `src/bicepconfig.json`

```bicep
{
    "providers":{
        "foo": {
            "source": {
                "registry": "mcr.microsoft.com/bicep/providers",
                "version": "1.2.3",
            },
            "isDefaultImport": false
        },
        "bar" : {
            "source": {
                "registry": "mcr.microsoft.com/bicep/providers",
                "version": "3.2.1"
            },
            "isDefaultImport": false
        }
    }
}
```

- `src/modules/mod.bicep`

```bicep
provider bar as modBarProvider
provider foo as modFooProvider
```

- `src/modules/bicepconfig.json`

```bicep
{
     "experimentalFeaturesEnabled": {
        "compileTimeImports": false
     },
}
```

The user may expect Bicep to resolve the provider details of `modFooProvider` by inspection of `src/bicepconfig.json` and for this project to compile successfully in a way that feels familiar with other programming languages where merge semantics are applied on code dependencies, however, Bicep config merge semantics uses the configuration file closest to the Bicep file in the directory hierarchy and merges them to the builtin config. It doesn't merge other configs in the file hierarchy prior to merging with the builtin config.

In this case Bicep will try to resolve the `modFooProvider` details from `src/modules/bicepconfig.json`. The config doesn't specify a `providers` section, and the builtin config doesn't include a definition for provider `foo` and so the compiler cannot resolve the provider declaration statement and will emit a diagnostic.

If we replace `foo` with `az` in the example above, Bicep resolves `modFooProvider`'s version to be the version of `az` specified in the builtin configuration and not the version `1.2.3` specified in `src/modules/mod.bicep` (inherited from `src/bicepconfig.json`) which may also be unexpected. The reson for this behavior is that current `bicepconfig.json` merge semantics merge `src/modules/bicepconfig.json` with the builtin config and the version applied would be the value specified in the builtin az provider version.

This is especially problematic for 3rd party namespace identifiers specified in a `bicepconfig.json` up a file hierarchy, since they would result in a compilation error because they are unspecified in the builtin configuration.

### Resolution of namespace identifiers by VSCode extension when inspecting published module sources

The ability to inspect sources is being implemented in Bicep to allow users to see what sources were used to compile a Bicep module, since the version information is codified in the `bicepconfig.json` and that is not published with the sources, the costumer will not be able to immediately know what version of a provider was used to resolve the resources in the souces file. This is illustrated in the example below.

**Example: User consumes a published module (that uses new provider syntax) and inspects its sources**

main.bicep

```bicep
provider az

module myMod 'br:mcr.microsoft.com/bicep/path/to/my_mod:1.2.3' = {
  name: 'example'
  params: {}
}
```

Let's assume that `my_mod` was published from a Bicep project with the following `bicepconfig.json`:

bicepconfig.json

```jsonc
{
  "providers": {
    "az": {
      "source": {
        "registry": "mcr.microsoft.com/bicep/providers",
        "version": "0.2.3"
      },
      "isDefaultImport": true
    }
  }
}
```

When the user inspects the sources of `my_mod` they would see the below source, notice how there is no way for the user to know that the `az` namespace is using resources from a package with version `0.1.2` from looking at this file.

my_mod.bicep

```bicep
provider az
param location string = 'eastus'

resource sa 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'test'
  location: location
  kind: 'StorageV2'
  sku: {
    name: 'Standard_LRS'
  }
}
```

### Wildcard semantics for provider versions in `bicepconfig.json`

At this time we are not proposing support for wildcard semantics on provider versions, this design doesn't preclude us from supporting this in the future.

### Deciding if `sys` is a compile time import or a resource types provider package

It is possible to think of `sys` more like a compile-time namespace import rather than a provider, since all it contains are some built-in functions. We used to share the keyword import between providers and imports, but now that we changed the keyword for providers, at some point we will need to update the implementation for sys to make it a compile-time import. This is out of scope of this REP however.