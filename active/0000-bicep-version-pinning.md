---
REP Number: <Fill me in with a four-digit number matching the pull request number; Update AFTER PR is approved and BEFORE is merged.>
Author: levimatheri (Levi Muriuki)
Start Date: 2024-11-20
Feature Status: Private Preview
Bicep Issue Number(s): [#12290](https://github.com/Azure/bicep/issues/12290), [#8978](https://github.com/Azure/bicep/issues/8978)
---

# Bicep version pinning

## Summary

This document proposes a feature that enables users to pin specific versions of Bicep for their deployments.

## Terms and definitions
### Definitions
- **AzCLI:** Azure Command-Line Interface, a set of commands used to create and manage Azure resources.
- **ADO:** Azure DevOps, a set of development tools for software development teams.
- **Ev2:** A deployment system used internally at Microsoft for deploying infrastructure and services.
- **bicepconfig.json:** A configuration file used to specify settings for the Bicep compiler.
- **Registry Module:** A reusable component or package that can be published and shared within a registry.
- **Entrypoint Bicep File:** The main Bicep file that serves as the starting point for a compilation.
- **Compilation:** The process of converting Bicep code into an Azure Resource Manager (ARM) template.

## Motivation

Currently, several tools allow the use of Bicep without pinning to a specific version, which is a common practice in CI pipelines. These tools include:
- AzCLI
- ADO task (AzureResourceManagerTemplateDeployment@3)
- GitHub Actions (azure/arm-deploy@v1)

This approach poses a significant risk because when a new Bicep version is released, users are automatically upgraded to the latest version without their explicit consent. This can lead to unexpected compilation issues, especially if the new version introduces breaking changes.

Implementing Bicep version pinning would ensure reproducible Bicep compilations across all deployment workflows, whether executed locally or within CI/CD pipelines. This feature would enhance stability and predictability, providing users with greater control over their deployment environments. Additionally, it would be crucial for enabling the use of Bicep natively in Ev2 to ensure safe deployment practices.

## Detailed design
### Add new version property in bicepconfig.json

The following example shows the proposed `version` property syntax to be added in the _bicepconfig.json_ file. See [version constraint syntax below](#version-constraint-syntax) for more details:

```json5
{
    "bicep": { // section for bicep compiler options
        "version": "0.31.92" // semantic version syntax or range to match Bicep version number
    }
}
```

The appropriate Bicep version will be determined by using the closest `bicepconfig.json` file to the Bicep file being compiled, merged with the default configuration.

----------------

### Version constraint syntax

Users can specify version constraints in Bicep using either exact versions or version ranges. This flexibility allows for precise control over the versions used in Bicep projects. Bicep adheres to the [Semantic Versioning 2.0 specification](https://semver.org/spec/v2.0.0.html) for its [published releases](https://github.com/Azure/bicep/releases), ensuring consistency and predictability in versioning.

However, the SemVer spec does not include a standard for expressing version ranges.

Inspired by the [npm version range specification](https://docs.npmjs.com/cli/v6/using-npm/semver), the following outlines the version syntaxes we will use for Bicep:

**1. Exact version syntax** 

For exact version constraint, the allowable syntax should be the version itself, i.e. `1.2.3`.

**2. Version range syntax**
#### Comparators
Expresses a set of comparators which specify versions that satisfy the range using the following `>, <, >=, <=` symbols for comparisons.

Examples:
- `>=0.31.0` - accepts any version above and including 0.31.0
- `<0.31.0` - accepts any version below, but not including 0.31.0
- `>=0.31.0 <1.0.0` - accepts any versions between 0.31.0 (inclusive), and 1.0.0 (exclusive)
- `=0.31.0` - equivalent to `0.31.0`
- `>=1.2` - equivalent to `>=1.2.0`
- `>=1` - equivalent to `>=1.0.0`

#### Tilde Ranges
Uses `~` to allow patch-level changes if a minor version is specified on the comparator. Allow minor-level changes if not. 

Examples:
- `~1.2.3` - equivalent to `>=1.2.3 <1.3.0`
- `~1.2` - equivalent to `>=1.2.0 <1.3.0`
- `~1` - equivalent to `>=1.0.0 <2.0.0`

#### Caret Ranges
Uses `^` to allow minor and patch-level changes.

Examples:
- `^1.2.3` - equivalent to `>=1.2.3 <2.0.0`
- `^1.2` - equivalent to `>=1.2.0 <2.0.0`
- `^1` - equivalent to `>=1.0.0 <2.0.0`


### Update Bicep compiler 
#### Version check during compilation
The Bicep CLI should be updated to check for the version constraint in the configuration.

If a user runs Bicep CLI directly to build a bicep file, we will try to resolve the appropriate version constraint from the closest `bicepconfig.json`:
- If the resolved version matches the locally-installed Bicep version, the compilation should continue normally using the resolved version.
- If the resolved version constraint does not match the locally-installed Bicep version, the compilation should fail with a clear message indicating the constraint violation.
- If no version constraint is specified anywhere in the `bicepconfig.json` resolution chain, the compilation should fallback to using the currently installed version.

#### Linter rules
- Since we've chosen to use bicepconfig.json for version constraints, the current expectation is consistent resolution logic (e.g., the closest `bicepconfig.json` to a Bicep file is used). 
- This also means users may specify different version constraints in different `bicepconfig.json` files. 
- External tools do not understand how Bicep source files are grouped, therefore they can't detect version constraints in referenced modules. 
- External tools can parse the version constraint for the entrypoint Bicep file, but conflicts won't be detected in referenced modules until the installed Bicep CLI is invoked.

- In order to help the users prevent such runtime issues, we will two linter rules:
1. **Incompatible Module Constraints:**
    - If the version constraints specified in `bicepconfig.json` files across modules cannot be combined into a single valid version range (i.e. disjoint ranges), emit an error. For example:
    - Module A requires `>=0.31.0 <0.32.0`.
    - Module B requires `>=0.15.0 <0.16.0`.
    - These constraints cannot be satisfied simultaneously, so the emit an error.

2. **Looser Constraints in a parent module:**
   - If the version constraint for the bicep file being edited is looser than any of the referenced modules' constraints, emit an error. For example:
   - Entrypoint file requires `>=0.15.0`.
   - Referenced module requires `<0.17.0`.
   - Because the external tool only looks at the entrypoint file, it may download a version that is incompatible with a referenced modules (e.g. version `0.21.0` in the example).
----------------

### VSCode experience
Because the VSCode extension version is tightly coupled to the Bicep compiler version. If we fail the Bicep compilation, the user if forced to update their VSCode extension version; this is not a great user experience especially when working across multiple Bicep repos that they even might not own. 

Ideally, we would decouple the extension from the Bicep version, but that would be an expensive undertaking.

An acceptable solution here is: instead of throwing compiler errors, we will warn the user if their VSCode extension version doesn't match the configured version constraint. 
This way the user is aware of the mismatch, but we do not completely block them from proceeding with compilation.
We will need to clearly document this behavior in which the pinning is not strictly enforced in VSCode, so that users do not have the wrong expectations.

--------------

### AzCLI updates
#### Basic mechanism
- The AzCLI bicep module should be enhanced to parse the appropriate `bicepconfig.json` file by recursively searching upwards from the current directory. It should then install the latest Bicep version that satisfies the specified version constraints. 
    - For example, given the version constraint `>=0.31.0 <0.32.0` and the available Bicep release tags `[0.21.14, 0.31.1, 0.31.92, 0.32.45]`, AzCLI would download version `0.31.92`.
- If no version constraints are found, AzCLI should default to using locally installed version (or downloading the latest version if no Bicep is not installed locally).

#### Note about `use_binary_from_path` config value
When `use_binary_from_path` AzCLI config is set to `true`, or `if_found_in_ci`, AzCLI will invoke the bicep binary found in the PATH environment variable. With version pinning, the version in PATH would not necessarily correspond to the constraint resolved from the `bicepconfig.json`. 

We should therefore set `use_binary_from_path` to `false` if a version constraint is found (this is similar to the behavior when a user runs `az bicep install --version` to install a specific version of Bicep. See more details on [this PR](https://github.com/Azure/azure-cli/pull/25541)).

#### Version syntax parsing
Since there is no official standard for the syntax of semver ranges, different tools/libraries that implement the npm-like syntax often have subtle or major differences. As a result, we will implement our own parser in Bicep so that we can better control what is supported & not supported. We can then utilize Copilot to help translate the parsing logic to other languages, i.e. Python for AzCLI.

## Drawbacks
- **Degraded visibility into registry module version constraints:** When inspecting published module source bicep file, it wouldn't be immediately clear what version was used to compile the module just by looking at the bicep file. This is because version information would be codified in the `bicepconfig.json` which is not published with the sources. However, the user can still discern the version used with the aid of the VSCode extension; the user can navigate to the compiled JSON which contains metadata about which version was used to compile the module. 
- **Inability to specify version override for specific individual files:** Due to how `bicepconfig.json` works, it applies to all files in the directory it is placed in. Users cannot easily override the Bicep version for individual files. This limitation can potentially be restrictive in scenarios where there may be a need to compile specific individual files with different Bicep versions for compatibility or testing purposes.

## Alternatives
### Specify Bicep version in `.bicep` files:
This might look something like the following:

```bicep
bicep.version = '0.31.92'

resource test 'Test.RP' = {...}
```
This approach provides more granular control over the Bicep version used for each individual file, which makes it clear which version is required for a specific file, which could be useful in scenarios where different files need to be compiled with different versions. It would also be easier to discern what version was used to compile a published module by inspecting the source.

However, the downside with this approach is if usage of the same version for multiple Bicep files is desired, it would require specifying the `version` in each file which can be a maintenance overhead.

### Version syntaxes considered
#### Option 1: NuGet-like syntax:

The main distinguishing feature of the NuGet version range is it uses a syntax similar to the [mathematical interval notation](https://www.math.net/interval-notation), i.e. `[]`, `()` to specify inclusivity vs exclusivity respectively.
For example:

- `[1.0.0,2.0.0]` - accepts any versions between from 1.0.0 to 2.0.0, inclusive
- `(4.1,3,)` - accepts any version above, but not including 4.1.3
- `[1.0.0,)` - accepts any version above and including 1.0.0
- `1.0.0` - accepts only an exact version, 1.0.0

See syntax reference on [NuGet docs](https://learn.microsoft.com/nuget/concepts/package-versioning?tabs=semver20sort#version-ranges).

*Tools that can parse this syntax*:
- .NET [NuGet.Versioning](https://www.nuget.org/packages/NuGet.Versioning/)

*Advantages*:

1. Familiarity for Bicep users with a NuGet or mathematical interval notation background.

*Disadvantages*:

1. For those unfamiliar with the notation, it may be confusing and require a learning curve, as this notation is not commonly used in other tools.
2. Tools that can parse this syntax are scarce

#### Option 2: npm syntax:

Inspired by [npm semver range spec](https://docs.npmjs.com/cli/v6/using-npm/semver#ranges).
Uses `>, <, =, >=, <=` symbols for comparisons.
For example:
- `>=0.31.0` - accepts any version above and including 0.31.0
- `<0.31.0` - accepts any version below, but not including 0.31.0
- `>=0.31.0 <1.0.0` - accepts any versions between 0.31.0 (inclusive), and 1.0.0 (exclusive)
- `=0.31.0` - equivalent to `0.31.0`
- `>=1.2` - equivalent to `>=1.2.0`
- `>=1` - equivalent to `>=1.0.0`
- `^1.2.3` - equivalent to `>=1.2.3 <2.0.0`
- `~1.2.3` - equivalent to `>=1.2.3 <1.3.0`

*Tools that can parse this syntax*:
    - .NET ([semver package](https://semver-nuget.org/v3.0.x/)) - this would be useful for parsing in the Bicep & Ev2 .NET SDK codebase
    - Python ([packaging library](https://python-semanticversion.readthedocs.io/en/latest/guide.html#the-simplespec-scheme)) - this would be useful for parsing in the AzCLI codebase
    - Javascript ([node-semver](https://github.com/npm/node-semver)) - would be useful for bicep-node
    - Go ([semver package](https://pkg.go.dev/github.com/Masterminds/semver/v3@v3.3.1)) - this would be useful for parsing in the Ev2 CLI codebase

*Advantages*:

1. The syntax is easier to read and understand as the symbols used are commonly used for comparison operations within Bicep and other tools.
2. Would require minimal or no learning curve for customers coming from an npm or terraform background
3. Many programming languages  have libraries that provide functions to parse this syntax, including:
    - .NET ([semver package](https://semver-nuget.org/v3.0.x/))
    - Python ([packaging library](https://packaging.pypa.io/en/stable/specifiers.html#packaging.specifiers.SpecifierSet))
    - Javascript ([node-semver](https://github.com/npm/node-semver))
    - Go ([semver package](https://pkg.go.dev/github.com/Masterminds/semver/v3@v3.3.1))

*Disadvantages*:

1. Could be considered more verbose in certain scenarios, e.g. to specify Bicep to match any patch version of `0.31`, the syntax would be `>=0.31.0 <0.32.0`. Likewise to match any minor version of `0`, the syntax would be `>=0 <1`.

**3. Wildcard syntax**

Users can use `*` to indicate that anything in the specified segment of the version is acceptable. This can be used as syntactic sugar for the version comparison syntax. The following are some examples:
- `1.*` - equivalent to `>=1.0.0,<2.0.0`
- `1.2.*` - equivalent to `>=1.2.0,<1.3.0`
- `*` -  equivalent to `>=0.0.0`

However, we should impose the following restrictions:
1. The wilcard `*` cannot be used together with the version comparison syntax. 
    - Reason: This is to prevent ambiguous snytaxes like this: `>=1.*`. It is hard to reason about the allowed range in this case, i.e. does the range start from `1.0.0` or `1.1.2` or `1.3.5` or something else?
2. If a version number is replaced with a wildcard, then all later version numbers must also be wildcards. Thus `2.*.6` is an invalid wildcard version. 
    - Reason: Most semver parsing tools (including the [tools sampled above](#option-2-npmterraform-syntax)) either do not support this, or they ignore anything after the `*`.

✅ We chose to go with Option 2 as it is more commonly used, flexible, and easy to read. See more details under [unresolved questions](#unresolved-questions).

## Rollout plan

1. This proposal only involves client-side changes, and will be an opt-in feature enabled when the bicep.version configuration property is specified (essentially acting as the feature flag).
2. Updates to AzCLI should soon thereafter follow the Bicep codebase changes

## Unresolved questions
1. Technically, the published Bicep binary versions on Github begin with a `v` which seems superfluous. Do we need to include this prefix as pattern of the constraint format?

    ✅  The `v` is a prefix used for the GitHub release tags, but we can assume 'v1.2.3' is equivalent to '1.2.3'. SemVer parsing libraries also handle this.

2. [From PR comment] Are more users using Bicep CLI or Azure CLI. We may need to ask the community. If more are using Bicep CLI, we may need to decide on how to provide a better experience there for users working across multiple repos that they may not necessarily own.

    ✅  Feedback from community was AzCLI is used more widely. Decision was to go ahead and implement for AzCLI, and revisit Bicep CLI experience later.

3. Which versioning syntax to go with?

    - Some concerns brought up was consistency with future module/extension version range addition, since OCI artifact versions are arbitrary and not necessarily numeric or semver.
    - Some ideas were brought up:
        - Use `*` wildcard. However isn't versatile enough for non-numeric usage.
        - Use Regex. This is powerful, but could potentially intimidating/barrier to entry for unfamiliar users.

    ✅ Decision was to stick to `>,<,>=,<=` syntax for version range syntax for both Bicep and module/extension versioning. The caveat is that version ranges will not be supported for module/extension OCI artifacts not following semver. In those cases, users will need to use exact versions.

4. Should we support `^` and `~` prefixes as used by [npm](https://docs.npmjs.com/cli/v6/using-npm/semver#advanced-range-syntax)?

    ✅ Decision was to include them as they is widely used. We will need to have good documentation about them as it is not immediately obvious what the symbols mean.

5. Should we build our own parsers or use libraries/packages?

    ✅ Start with libraries/packages as long as they are consistent with each other and support all that we need. Reconsider writing our own if there are concerns.

    
## Out of scope
1. Making this work in AzPwsh would be nice to have as the owning team is now more open for us to make changes, but would involve significant work to make it at par with AzCLI. This can be considered future enhancement.
2. There were concerns about how this would work with bicepconfig.json resolution mechanism. Today, the closest bicepconfig.json file is used to resolve configurations relative to the bicep/bicepparam file being compiled. Since there exists no merge process for the bicepconfig.json files, this could be problematic in the cases where users want to have repo-wide settings honored even with child folders containing their own bicepconfig.json files. This limitation, however, has existed already, and hence orthogonal to this proposal; we could adopt this proposal without necessarily solving the bicepconfig.json limitation. 