---
REP Number: 0000
Author:asilverman (Ariel Silverman)
Start Date: 2023-12-19
Feature Status: Private Preview
Bicep Issue Number(s): [12202](https://github.com/Azure/bicep/issues/12202)
---

# Title - Centralized management of resource type provider versions using package.json

## Summary

This document proposes a feature that enables users to manage Bicep resource type provider versions from a central location. This enhances convenience by sparing the user from making changes in multiple files in order to upgrade a dynamically loaded resource types provider package version.

## Terms and definitions

- **Resource Types Provider Package aka provider package** - The OCI Registry artifact that contains the resource type definitions used by Bicep to identify the resource types properties, functions and other relevant type information that it uses to generate ARM-JSON.
- **types.json protocol** - It is a serialization protocol used by Bicep to encode type information as data that can be loaded to the Bicep runtime.

## Motivation

The [resource type providers dynamic loading feature](https://github.com/Azure/bicep/issues/5453) introduces a change in provider declaration statement syntax to include the registry address and version that Bicep will use to resolve resource type declarations.

** Provider Declaration Syntax**

```bicep
import 'br:mcr.microsoft.com/bicep/providers/az@1.2.3' // fully-qualified
import 'br/public:az@1.2.3' // uses provider aliasing defined in bicepconfig.json
```

We also currently support the following syntax for "built-in" providers:

```bicep
import 'sys@1.0.0'
```

In the above cases, the provider can be referenced by using its symbol which is derived from the segment to the left of the '@' sign. (`az` and `sys` for the declarations above)

Both syntaxes also support aliasing of that symbol reference:

```bicep
import 'br/public:az@0.2.3' as myAz
import 'sys@1.0.0' as mySys
```

The current syntax (shown above) presents several challenges:

1. The new syntax is quite verbose, and cannot be controlled centrally. For users with a large set of files, picking up the latest version of `az` will require modifying every single file.
1. The aliasing (assuming that the last segment of the registry namespace == the provider name) & versioning (having an `@` character instead of a `:` which is familiar to module users) behavior of the new syntax can be confusing.
1. Its deceiving and confusing to allow `sys` to have a pinned versionis fixed version `1.0.0` - this **should** imply that there are no changes between Bicep versions, which isn't realistic.

- It's an "either/or" - either users follow the current behavior (`az` imported by default) **or** they're using the new syntax. We don't have a plan for how to combine the two.
- With [Radius](https://github.com/radius-project/radius), it's conceivable that we may want users to **not** have `az` imported by default.

### Proposed Changes

[Add a new syntax for the provider import declaration statements: `import {providerName} [as {optionalAlias}]`](#add-a-new-syntax-for-the-provider-import-declaration-statements-import-providername-as-optionalalias)
[Remove support for syntax `import '{providerName}@{providerVersion}' [as {optionalAlias}]`](#remove-support-for-syntax-import-providernameproviderversion-as-optionalalias)
[Change the delimiter '@' used by the supported provider declaration syntax to ':' to align with module reference syntax and increase consistency](#change-the-delimiter--used-by-the-supported-provider-declaration-syntax-to--to-align-with-module-reference-syntax-and-increase-consistency)

## Detailed design

### Client side changes

#### Add a new syntax for the provider import declaration statements: `import {providerName} [as {optionalAlias}]`

The example below shows a usage of the proposed syntax

```bicep
import az
import kubernetes as k8s // uses optional aliasing of the
```

The resolution of the symbols will be resolved from the builtin `bicepconfig.json` under a new section:

```json
{
  // prior bicepconfig.json sections
  // ...

  "providers": {
    "az": {
      // A package reference string (below) gestures Bicep to source the types from the registry address.
      "source": "mcr.microsoft.com/bicep/providers/az:0.2.3", 
      "isDefaultImport": true // Gestures Bicep to implicitly import the namespace identifier to the global scope
    },
    "kubernetes": {
      // Using the value "builtin" (below) gestures Bicep to source types from the statically compiled NuGet package reference 
      "source": "builtin", 
      "isDefaultImport": false
    }
  }
}
```

Since previous provider declaration syntax forms continue to be supported, we introduce the following constraints:

- To ensure consistency with pre-existing handling of `bicepconfig.json` the configuration file closest to the Bicep file in the directory hierarchy is used. The [drawbacks](#drawbacks) of this constraint are described below.
- The `sys` namespace is coupled to the Bicep bits version and cannot be overriden or dynamically uploaded using the mechanism described above

#### Remove support for syntax `import '{providerName}@{providerVersion}' [as {optionalAlias}]`

The syntax below currently in use to import built-in providers will become invalid and result in a diagnostic as a result of this change

```bicep
import 'sys@1.0.0'
import 'kubernetes@1.0.0' as k8s
```

#### Change the delimiter '@' used by the supported provider declaration syntax to ':' to align with module reference syntax and increase consistency

The syntax shown below uses ':' instead of '@' as a delimiter for provider declaration statements

```bicep
import 'br:mcr.microsoft.com/bicep/providers/az:0.2.3' // notice its NOT using az@0.2.3
import 'br/public:az:0.2.3' as myAz
```

## Drawbacks

> Discuss any potential drawbacks associated with this approach. Why might this _not_ be a suitable choice? Consider implementation costs, integration with existing and planned features, and the migration cost for Bicep users. Clearly identify if it constitutes a breaking change.

1. The proposal uses `bicepconfig.json` to declare the provider information and is ruled by `bicepconfig.json` merge semantics. The merge semantics for `bicepconfig.json` specify that for a given Bicep source file, the configuration file closest to the Bicep file in the directory hierarchy is used. This behavior is not intuitive and may cause users that have specified multiple config files in their project file-structure to experience unexpected behaviors that may include build errors when Bicep attempts to resolve a namespace identifier. This is illustrated by the example below.

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
                "registry": "mcr.microsoft.com/bicep/providers/az",
                "version": "1.2.3",
            },
            "isDefaultImport": false
        },
        "bar" : {
            "source": {
                "registry": "mcr.microsoft.com",
                "repository": "bicep/providers",
                "name":
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

The user may intuitively expect Bicep to resolve the provider details of `modFooProvider` by inspection of `src/bicepconfig.json` and for this project to compile successfully in a way that feels familiar with other programming languages where merge semantics are applied on code dependencies (e.g. NPMs `package.json`), however, Bicep config merge semantics uses the configuration file closest to the Bicep file in the directory hierarchy and merges them to the builtin config.

In this case Bicep will try to resolve the `modFooProvider` details from `src/modules/bicepconfig.json`. The config doesn't specify a `providers` section, and the builtin config doesn't include a definition for provider `foo` and so the compiler cannot resolve the provider declaration statement and will emit a diagnostic.

If we replace `foo` with `az` in the example above, Bicep resolves `modFooProvider`'s version to be the version of `az` specified in the builtin configuration and not the version `1.2.3` specified in `src/modules/mod.bicep` (inherited from `src/bicepconfig.json`) which is also not intuitive. The reson for this behavior is that current `bicepconfig.json` merge semantics merge `src/modules/bicepconfig.json` with the builtin config and the version applied would be the value specified in the builtin az provider version.

This is especially problematic for 3rd party namespace identifiers specified in a `bicepconfig.json` up a file hierarchy, since they would result in a compilation error because they are unspecified in the builtin configuration.

1. TODO: Potential drawback when trying to inspect module sources 
<!--
Good morning Stephen & Anthony, I am currently working on a design draft for a feature proposal that may have an overlap with the work you are doing with publishing of sources. The proposal highlights are described here: Proposal - Simplification of "built-in" vs "registry-supplied" providers · Issue #12202 · Azure/bicep (github.com)
 
One potential consideration is how would the identifiers be resolved when a user is looking at module sources, specifically consider the scenario below. 
Stephen, I would really appreciate your thoughts on whether this new syntax affects the source publish requirements  or has some other kind of implications in the ability to resolve published sources in the editor.
 
Scenario: User creates and publishes with sources using the new provider syntax 
 
Question: How does the VSCode extension  resolve the registry location and provider version it needs to resolve the namespace identifier (`az`) when the consumer of this module want's to inspect the sources?
 
my_mod.bicep
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


Let's say that the project where the user is publishing this module had the following bicepconfig.json:
bicepconfig.json
{
  "providers": {
    "az": {
      "source": {
        "registry": "mcr.microsoft.com/bicep/providers/az",
        "version": "0.2.3",
      },      "isDefaultImport": true    },    
      "sys": {
          "source": {
          "builtin": true
      },
      "isDefaultImport": true
    }
  }
}
-->

## Alternatives

- Using a separate file other than `bicepconfig.json` to store resource type provider version information was evaluated but adds complexity to UX as users need to be aware of two separate merge semantics (one for config and one for provider packages) 

## Rollout plan

This change affects the client side only and is applied only when the pre-existing feature flag `DynamicTypeLoadingEnabled` is set. 

## Unresolved questions

> - What aspects of the design do you expect to resolve through the REP process before merging?
> - What parts of the design do you anticipate resolving through feature implementation before stabilization?

## Out of scope

- `bicepconfig.json` merge semantics
- resolution of namespace identifiers by VSCode extension when inspecting published module sources  

**NB**: This section is not intended to capture unfinished elements of a design. For draft proposals, anything that is yet to be decided but must be addressed in order for a feature to be functional or GA should be explicitly called out in the **Design** section (or the **Unresolved questions** section). This section is instead intended to identify related issues or features on which you do not wish to make definitive decisions/foreclose any possibilities. Carefully think through how your design will avoid impacting these out of scope questions -- if your design does impact anything in this section, that is a strong indication that whatever is impacted was not out of scope at all.
