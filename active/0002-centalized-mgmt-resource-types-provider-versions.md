---
REP Number: 0002
Author: asilverman (Ariel Silverman)
Start Date: 2023-12-19
Feature Status: Private Preview
Bicep Issue Number(s): [12202](https://github.com/Azure/bicep/issues/12202)
---

# Title - Centralized management of resource type provider versions using bicepconfig.json

## Summary

This document proposes a feature that enables users to manage Bicep resource type provider versions from a central location. This improves convenience by sparing users from making changes in multiple files in order to upgrade a dynamically loaded resource types provider package version.

## Terms and definitions

- **Resource Types Provider Package aka provider package** - The OCI Registry artifact that contains the resource type definitions used by Bicep to identify the resource types properties, functions and other relevant type information that it uses to generate ARM-JSON.
- **types.json protocol** - It is a serialization protocol used by Bicep to encode type information as data that can be loaded to the Bicep runtime.

## Motivation

The [resource type providers dynamic loading feature](https://github.com/Azure/bicep/issues/5453) introduces a change in provider declaration statement syntax to include the registry address and version that Bicep will use to resolve resource type declarations.

**Provider Declaration Syntax**

```bicep
provider 'br:mcr.microsoft.com/bicep/providers/az@1.2.3' // fully-qualified
provider 'br/public:az@1.2.3' // with provider aliasing defined in bicepconfig.json
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
1. Its deceiving and confusing to allow `sys` to have a pinned version fixed to `1.0.0` - this **should** imply that there are no changes between Bicep versions, which isn't realistic.
1. It's an "either/or" - either users follow the current behavior (`az` imported by default) **or** they're using the new syntax. We don't have a plan for how to combine the two.
1. With [Radius](https://github.com/radius-project/radius), it's conceivable that we may want users to **not** have `az` imported by default. We want to allow users to opt-out from the implicit importing of the `az` namespace 

## Proposed Changes
<!-- no toc -->
- [Client side changes](#client-side-changes)
  - [Add a new syntax for the provider import declaration statements: `provider {providerName} [as {optionalAlias}]`](#add-a-new-syntax-for-the-provider-import-declaration-statements-provider-providername-as-optionalalias)
  - [Change the delimiter '@' used by the supported provider declaration syntax to ':' to align with module reference syntax and increase consistency](#change-the-delimiter--used-by-the-supported-provider-declaration-syntax-to--to-align-with-module-reference-syntax-and-increase-consistency)
  - [Allow users to configure providers that are implicitly imported](#allow-users-to-configure-providers-that-are-implicitly-imported)
## Detailed design

### Client side changes

#### Add a new syntax for the provider import declaration statements: `provider {providerName} [as {optionalAlias}]`

The example below shows a usage of the proposed syntax

```bicep
provider az
provider kubernetes as k8s // uses optional aliasing of the
```

The resolution of the identifiers will be determined from the closest `bicepconfig.json` to the file merged with the default configuration. We propose the following changes:

- Add new `providers` and `implicitProviders` sections to the `bicepconfig.json`.
- Use the scheme `builtin:` as a sentinel value to declare the use of the NuGet package reference for a resource types provider
- Use the scheme `br:` to prefix the fully qualified address of a provider package

Example: The new default configuration
```json
{
  // prior bicepconfig.json sections
  // ...
  "providers": {
    "az": "builtin:",
    "kubernetes": "builtin:",
    "microsoftGraph": "builtin:"
  },
  "implicitProviders": [
    "az"
  ]
}
```

Users can opt-in to using centralized provider version management by specifying a `bicepconfig.json` in their project structure.

Example: A `bicepconfig.json` that defines a dynamically loaded provider
```json
{
 "providers": {
    "az": "br:mcr.microsoft.com/bicep/providers/az:0.2.3"
  },
  "implicitProviders": [
    "kubernetes"
  ]
}
```

Bicep will replace-merge the contents in the `bicepconfig.json` with the default resulting the following configuration being loaded.

`bicepconfig.json`
```json
{
 "providers": {
    "az": "br:mcr.microsoft.com/bicep/providers/az:0.2.3",
    "kubernetes": "builtin:",
    "microsoftGraph": "builtin:",
  },
  "implicitProviders": [
    "kubernetes"
  ]
}
```

Given legacy provider declaration syntax grammar continues to be supported, we introduce the following constraints:

- To ensure consistency with pre-existing handling of `bicepconfig.json` the configuration file closest to the Bicep file in the directory hierarchy is used.
- The `az` identifier is implicitly imported into the global scope, the author can override the type definitions used in the namespace by adding an entry in the `providers` section (as shown above). 
- The versions used to resolve built-in providers are determined by the NuGet package reference dependency in `Bicep.Core.csproj`.
- The keys of the `providers` object must be distinct. Notice how the source is the full repository path, so its possible to disambiguate in the case an author chosses to consume a provider with the same name from separate sources.
- The keys of the `providers` section will be used as the identifier (`{providerName}`) in the `provider {providerName} [as {optionalAlias}]` declaration statement.
- The `sys` identifier is coupled to the Bicep bits and its version cannot be overriden or dynamically uploaded using the mechanism described above
- If a user specifies an entry called `sys` in the `providers` section above, it will result in a json schema violation. This behavior is to prevent a user from overriding the sys namespace.

Using the syntax below to import built-in providers will be deprecated and result in a diagnostic error as a result of the implementation of this proposal

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

#### Allow users to configure providers that are implicitly imported

The `implicitProviders` section of `bicepconfig.json` allows users to opt-in/out of implicit import functionality for a given provider. This section also follows the existing config merge semantics so users interested in keeping with the legacy behavior must opt-in by specifying the `az` provider in the array.

## Drawbacks

- Users may be confused by the config merge semantics behavior
- Users will not be able to tell from inspecting sources alone what is the provider version used to resolve resource types

## Alternatives

- Using a separate file other than `bicepconfig.json` to store resource type provider version information was evaluated but adds complexity to UX as users need to be aware of two separate merge semantics (one for config and one for provider packages). Although techically this solves the problem, it would have a detrimental effect in the ease of use and reasoning of Bicep projects and so it is dismissed as a non starter.

## Rollout plan

This change affects the client side only and is applied only when the pre-existing feature flag `DynamicTypeLoadingEnabled` is set.

## Out of scope

<!-- no toc -->
* [Updating `bicepconfig.json` merge semantics](#updating-bicepconfigjson-merge-semantics)
* [Resolution of namespace identifiers by VSCode extension when inspecting published module sources](#resolution-of-namespace-identifiers-by-vscode-extension-when-inspecting-published-module-sources)
* [Wildcard semantics for provider versions in `bicepconfig.json`](#wildcard-semantics-for-provider-versions-in-bicepconfigjson)
* [Deciding if `sys` is a compile time import or a resource types provider package](#deciding-if-sys-is-a-compile-time-import-or-a-resource-types-provider-package)
* [Implicit import of provider identifiers to global scope](#implicit-import-of-provider-identifiers-to-global-scope)

### Updating `bicepconfig.json` merge semantics

The proposal declares the provider information in `bicepconfig.json` and as result is ruled by `bicepconfig.json` merge semantics. The merge semantics for `bicepconfig.json` specify that for a given Bicep source file, the configuration file closest to the Bicep file in the directory hierarchy is used. This behavior may not be expected by customers and may cause users that specified multiple config files in their project file-structure to experience consuding behaviors that may include build errors when Bicep attempts to resolve a namespace identifier. This is illustrated by the example below.

**Example: Provider resolution error due as a result of `bicepconfig.json` merge semantics**

Consider a Bicep project under folder `src` with the following files:

`src/main.bicep`

```bicep
provider foo as mainFooProvider
```

`src/bicepconfig.json`

```json
{
  "providers":{
    "foo": "mcr.microsoft.com/bicep/providers/foo:1.2.3",
    "bar" : "mcr.microsoft.com/bicep/providers/bar:3.2.1"
  }
}
```

`src/modules/mod.bicep`

```bicep
provider bar as modBarProvider
provider foo as modFooProvider
```

`src/modules/bicepconfig.json`

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

The ability to inspect sources is being implemented in Bicep to allow users to see what sources were used to compile a Bicep module, since the version information is codified in the `bicepconfig.json` and that is not published with the sources, the user will not be able to immediately know what version of a provider was used to resolve the resources in the souces file. This is illustrated in the example below.

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
    "az": "br:private.azurecr.io/bicep/providers/az:0.2.3"
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

There is an ongoing discussion on the topic of `sys` being a compile-time import or a resource types provider. Until the topic is settled, we will treat it as a provider and allow users the option to override the identifier `sys` if they chose in `bicepconfig.json`. Doing so will effectively result in users losing access to basic bicep functions, although this scenario is unlikely considering current usage patterns.
