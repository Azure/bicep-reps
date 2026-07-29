---
REP Number: "0023"
Author: shenglol (Shenglong Li)
Start Date: 2026-07-27
Feature Status: Public
Bicep Issue Number(s): "[#5022](https://github.com/Azure/bicep/issues/5022)"
---

# Bicep configuration inheritance

## Summary

This proposal adds configuration inheritance to `bicepconfig.json` through a top-level `extends` property. A configuration can extend one local base configuration, and configurations can form a linear inheritance chain. Derived values are merged over base values using Bicep's existing configuration merge behavior.

Automatic configuration discovery remains unchanged. Each Bicep or Bicep parameters file uses the nearest ancestor file named `bicepconfig.json` as its leaf configuration. That file can use `extends` to compose shared settings from files with any name.

The design preserves declaring-file-relative path behavior and treats every inherited file as a configuration dependency.

## Terms and definitions

- **Built-in configuration**: The default configuration distributed with Bicep.
- **Discovered configuration**: The nearest file named `bicepconfig.json` found by searching from a source file's directory through its ancestor directories.
- **Leaf configuration**: The discovered configuration from which an inheritance chain starts.
- **Base configuration**: A configuration referenced by another configuration's `extends` property.
- **Configuration chain**: The ordered set containing a leaf configuration and all of its transitive base configurations.
- **Effective configuration**: The result of merging the built-in configuration, the configuration chain, and any supported invocation-time override.
- **Declaring configuration**: The configuration file in which a particular value is written. Relative path values are resolved from this file.
- **Configuration document**: One parsed configuration file together with its URI and source information.
- **Configuration source map**: A mapping from values in the merged configuration to the configuration document and source span that declared each value.

## Motivation

This proposal addresses [Azure/bicep#5022](https://github.com/Azure/bicep/issues/5022), which requests TypeScript-style configuration inheritance for Bicep.

Bicep currently merges one user configuration with the built-in configuration. When multiple ancestor directories contain `bicepconfig.json`, Bicep uses only the nearest file and ignores the others. This is predictable, but it requires users to duplicate shared analyzer, formatter, extension, and artifact settings whenever a nested project needs a small customization.

This is especially costly in monorepos and centrally managed build systems:

- An organization may maintain common analyzer levels while each project adds a few project-specific rules.
- A repository may define shared module or artifact aliases while individual projects override only a registry or subscription.
- A shared configuration may live outside a project's directory and otherwise must be copied into every consuming project.
- A configuration containing relative local paths must remain portable when it is inherited from projects at different depths.

Two related proposals inform this design:

- [Inline bicepconfig overrides](https://github.com/Azure/bicep-reps/pull/20) proposes invocation-specific value overrides. Inheritance produces the effective file-based configuration over which an applicable invocation override is merged.
- [Artifact aliases and redirects](https://github.com/Azure/bicep-reps/pull/22) defines configuration values whose relative paths belong to the configuration file that declares them. Inheritance preserves that property-level origin instead of rebasing all inherited paths to the leaf configuration.

The goal is to make configuration reuse explicit, deterministic, and portable without changing discovery or existing projects that do not opt in.

## Detailed design

### Client side changes

#### Configuration resolution overview

Bicep performs the existing nearest-ancestor discovery independently for each Bicep or Bicep parameters source file. If discovery finds a `bicepconfig.json`, that file is the leaf configuration. If discovery finds no configuration, the built-in configuration is used by itself.

Existing source-file-specific configuration boundaries are preserved. For example, a local module with a nearer `bicepconfig.json` continues to use that configuration even if its entrypoint discovers a different one. Either discovered file can explicitly extend a shared base.

After finding a leaf configuration, Bicep loads its inheritance chain and merges layers in the following order, from lowest to highest precedence:

1. Built-in configuration.
1. The most distant base configuration.
1. Each successive derived configuration.
1. The leaf configuration.
1. An invocation-time configuration override, if supported and applicable to that source file.

Conceptually, for a chain `leaf -> base1 -> base2`, resolution is:

```text
effective = merge(builtIn, base2, base1, leaf, invocationOverride)
```

The built-in configuration is merged exactly once. `extends` is loading metadata and is not included in the effective configuration.

This proposal does not change the syntax, scope, or path semantics of the [inline configuration override proposal](https://github.com/Azure/bicep-reps/pull/20). When an override applies to a source file, Bicep first resolves that file's discovered inheritance chain and then merges the override at the highest precedence.

#### The `extends` property

A configuration can contain one top-level `extends` property whose value is a non-empty string:

```jsonc
{
  "extends": "../shared/bicepconfig.base.json",
  "analyzers": {
    "core": {
      "rules": {
        "no-hardcoded-env-urls": {
          "level": "error"
        }
      }
    }
  }
}
```

The referenced file:

- Must be specified as a relative filesystem path. Absolute paths and URI references are invalid.
- Is resolved relative to the configuration containing `extends`.
- Must identify one local file that can be parsed using the same JSON-with-comments rules as `bicepconfig.json`.
- Does not need to be named `bicepconfig.json`.
- Is not subject to filename or extension inference. The path must identify the complete filename.
- Can contain its own `extends` property.

The relative path can contain `..` segments and can therefore reference a file outside the workspace.

Only one base can be named directly. A linear chain covers the shared-base scenario while avoiding precedence ambiguity and duplicate-base loading. Multiple inheritance is discussed under Alternatives.

#### Merge behavior

Each configuration layer is merged using Bicep's existing recursive configuration merge behavior:

- Objects are merged recursively by property name.
- When both layers contain the same object property, the derived value is merged over the base value.
- Arrays are replaced in full by the derived array.
- Primitive values and other non-object values are replaced by the derived value, except that JSON `null` retains Bicep's existing merge behavior and does not replace a non-null base value. A `null` value is still an error wherever the property's validation contract does not allow `null`.

For example:

```jsonc
// bicepconfig.base.json
{
  "analyzers": {
    "core": {
      "enabled": true,
      "rules": {
        "no-unused-params": {
          "level": "warning"
        },
        "no-unused-vars": {
          "level": "warning"
        }
      }
    }
  },
  "cloud": {
    "credentialPrecedence": [
      "AzureCLI",
      "AzurePowerShell"
    ]
  }
}
```

```jsonc
// ci/bicepconfig.json
{
  "extends": "../bicepconfig.base.json",
  "analyzers": {
    "core": {
      "rules": {
        "no-unused-params": {
          "level": "error"
        }
      }
    }
  },
  "cloud": {
    "credentialPrecedence": [
      "ManagedIdentity"
    ]
  }
}
```

The effective user-authored portion is:

```jsonc
{
  "analyzers": {
    "core": {
      "enabled": true,
      "rules": {
        "no-unused-params": {
          "level": "error"
        },
        "no-unused-vars": {
          "level": "warning"
        }
      }
    }
  },
  "cloud": {
    "credentialPrecedence": [
      "ManagedIdentity"
    ]
  }
}
```

The derived configuration replaces the complete `credentialPrecedence` array but overrides only one analyzer rule. This proposal does not add a deletion sentinel or a way to replace an object as an atomic unit. A derived configuration can override an inherited value, but it cannot remove one inherited property from an object map.

#### Relative path provenance

Configuration properties retain their existing property-specific path semantics. This proposal does not make every string that resembles a relative path configuration-file-relative.

For a property whose contract defines its value as relative to the configuration file, the value is resolved relative to the configuration file that declares that value. Merging must therefore preserve the source configuration URI for those values in the effective configuration. Current examples include `moduleAliasesMock.br.*.mapToFilePath`; the artifact file aliases and redirect targets proposed by [artifact aliases and redirects](https://github.com/Azure/bicep-reps/pull/22) follow the same rule. `extends` itself always resolves a relative value from the file containing that `extends` property.

Properties with another established base keep that behavior. In particular, this proposal does not rebase `cacheRootDirectory` or change its existing handling of relative values and `~`. A future configuration property must define its own path base; it is configuration-file-relative only when its contract says so.

##### Source-aware configuration representation

Inheritance cannot be represented correctly by attaching one configuration URI to the complete merged object. The shared configuration manager therefore represents each parsed file as a configuration document and produces both merged JSON and a configuration source map.

The merge applies the following origin rules:

- A primitive or other replacing value receives the origin of the derived value that survives.
- A recursively merged object retains the origin of each surviving child independently.
- A replacing array and every entry in that array receive origins from the replacing layer.
- The built-in configuration has a synthetic built-in origin rather than a filesystem URI.
- An applicable invocation-time override has the origin defined by the inline override proposal and never inherits a file origin implicitly.

After effective validation, the binder converts configuration-file-relative values to normalized absolute `IOUri` values before exposing typed configuration sections to artifact, extension, or other consumers. Consumers do not resolve those values through a single leaf `ConfigFileUri`. The source map remains available for diagnostics, editor navigation, effective-configuration inspection, and any future property that needs its declaring document. This keeps provenance inside the configuration subsystem instead of coupling every configuration consumer to inheritance.

Consider this layout:

```text
repo/
  config/
    bicepconfig.base.json
    environments/
      bicepconfig.dev.json
  modules/
    storage.bicep
  apps/
    storefront/
      bicepconfig.json
      main.bicep
```

The base configuration declares a module alias mock relative to `repo/config`:

```jsonc
// repo/config/bicepconfig.base.json
{
  "moduleAliasesMock": {
    "br": {
      "shared": {
        "mapToFilePath": "../modules/storage.bicep"
      }
    }
  }
}
```

The development configuration inherits it:

```jsonc
// repo/config/environments/bicepconfig.dev.json
{
  "extends": "../bicepconfig.base.json",
  "analyzers": {
    "core": {
      "enabled": false
    }
  }
}
```

The discovered leaf for the storefront explicitly selects the development layer:

```jsonc
// repo/apps/storefront/bicepconfig.json
{
  "extends": "../../config/environments/bicepconfig.dev.json"
}
```

`main.bicep` discovers `repo/apps/storefront/bicepconfig.json`, which loads the development and base configurations transitively. The inherited `moduleAliasesMock.br.shared.mapToFilePath` continues to resolve from `repo/config`, producing `repo/modules/storage.bicep`. It is not resolved from `repo/config/environments`, `repo/apps/storefront`, or the process working directory. If the derived configuration replaces `mapToFilePath`, the new value is resolved from the derived configuration's directory.

For a configuration-file-relative property in a replaced array, all path entries in that array belong to the replacing configuration. For recursively merged objects, unchanged configuration-file-relative child values retain their original declaring configuration while replaced child values use the replacing configuration.

An invocation-time override is not an inherited configuration file and does not acquire the leaf configuration's path base. Its path provenance remains defined by the inline override proposal.

#### Loading, validation, and cycles

Bicep parses the complete chain before merging. Each configuration document must contain valid JSON with a top-level object and a valid `extends` value when present. Bicep then removes `extends`, merges the built-in configuration and all file layers, and validates the resulting effective configuration using the same runtime configuration rules used today.

A base configuration can be partial; omitted values can be supplied by another layer or the built-in configuration. A value that is completely replaced by a derived layer does not need to form a valid effective value on its own. This proposal does not introduce stricter runtime unknown-property or duplicate-property validation for configurations that use inheritance. Editor schema validation remains advisory, as it is for `bicepconfig.json` today.

`extends` is loading metadata. It is not merged into the effective configuration and does not appear in effective-configuration serialization. An invocation-time override containing `extends` is invalid and cannot load another file.

Configuration loading fails when:

- `extends` is not a string, is empty or whitespace-only, or is not a relative filesystem path.
- A referenced file does not exist, cannot be read, or is not a file.
- A referenced file contains invalid JSON or does not contain a top-level object.
- A chain contains a cycle.
- The final effective configuration is invalid.
- The chain contains more than 64 files, including the leaf.

Cycle detection uses normalized absolute file identities after resolving each relative `extends` value. Implementations use canonical real paths when available and otherwise use normalized paths with the platform's filesystem comparison rules. The 64-file limit provides deterministic termination when canonical identity is unavailable or the chain is pathologically deep.

The cycle diagnostic includes the complete cycle so users can locate every participating `extends` property:

```text
bicepconfig.a.json -> bicepconfig.b.json -> bicepconfig.a.json
```

An invalid discovered or inherited configuration is an error. Bicep must not silently ignore a layer or continue with a partially merged configuration.

Diagnostics for inherited-file failures identify the failing file and the `extends` location through which it was loaded. Effective-configuration diagnostics are reported against the leaf, with related information for a contributing inherited value when available. CLI diagnostics include the path and inheritance chain.

The Bicep configuration JSON schema is updated with `extends`, including relative-path completion and a description of its resolution base. The language server validates the merged effective configuration and reports constraints that can only be evaluated after merging.

The extension provides navigation from an `extends` value to the referenced file and recognizes transitively loaded files as Bicep configuration documents even when they have a nonstandard filename.

#### CLI and Visual Studio Code behavior

This proposal adds no CLI argument, Visual Studio Code setting, or configuration-selection command. The CLI and language server continue to use automatic nearest-ancestor discovery. After discovering a leaf configuration, both use the shared configuration manager to load and merge its inheritance chain.

The Visual Studio Code extension adds schema support, relative-path completion, navigation, and diagnostics for `extends`. It also tracks inherited configurations as dependencies so editing a base file refreshes every affected Bicep or Bicep parameters compilation. Once a nonstandard file is reached through `extends`, the extension recognizes it as a Bicep configuration document for that workspace session.

The extension continues to honor Visual Studio Code Workspace Trust. Extending a configuration does not grant configuration-driven operations privileges that are unavailable to an automatically discovered configuration.

#### Configuration graph watching and caching

The compiler, language server, and extension treat every file in an inheritance chain as an input dependency.

- Editing, creating, deleting, or renaming any discovered or inherited configuration invalidates every dependent effective configuration.
- The language server watches inherited files, including relative paths that resolve outside the workspace folder where the client supports such watching.
- Changing an `extends` value updates the dependency set and recompiles affected source files.
- Configuration cache entries account for the leaf, every base, and any invocation override.
- Compilation and artifact-resolution caches include the effective configuration fingerprint where configuration can affect their results.

Implementations can cache a parsed base shared by multiple leaves, but cached parsing must retain property-level source provenance.

#### Interaction with existing configuration behavior

This proposal preserves the following behavior when `extends` is not used:

- Only a file named `bicepconfig.json` participates in automatic discovery.
- The nearest discovered configuration wins; ancestor configurations are not merged automatically.
- Different local modules can discover different configurations.
- User configuration is recursively merged over the built-in configuration.
- Existing configuration property validation and unknown-property behavior are unchanged.

A nested project can opt into ancestor inheritance explicitly:

```jsonc
// repo/services/api/bicepconfig.json
{
  "extends": "../../bicepconfig.json",
  "analyzers": {
    "core": {
      "rules": {
        "use-recent-api-versions": {
          "level": "error"
        }
      }
    }
  }
}
```

This makes the relationship visible and prevents unrelated configuration files placed in intermediate directories from entering the merge implicitly.

For configuration introduced by the [artifact aliases and redirects proposal](https://github.com/Azure/bicep-reps/pull/22):

- Alias and redirect maps merge recursively like other configuration objects.
- A derived configuration can override one alias or redirect without copying unrelated entries.
- Relative file alias paths and redirect targets retain the configuration file that declares each value as their resolution base.
- Existing `moduleAliases`, `moduleAliasesMock`, and artifact compatibility guarantees are unchanged.

Changing the discriminator of an inherited typed configuration object can leave inherited properties that are invalid for the new type because objects merge recursively. The resulting effective configuration fails validation. Users should use a new key when changing the type of an inherited alias. This proposal does not add atomic-object replacement semantics.

#### Backward compatibility

This proposal is opt-in and does not change the result for existing invocations and configurations:

- A configuration without `extends` loads as it does today.
- Existing automatic discovery and local-module configuration boundaries remain unchanged.
- Generated ARM templates contain no configuration metadata and require no deployment-engine changes.

Older Bicep versions do not understand `extends`. Projects that adopt inheritance must use a Bicep CLI and extension version that supports this proposal. This is normal feature-version compatibility rather than a change to existing files.

### Server side changes

None. Configuration inheritance is a Bicep client behavior.

### Microsoft.Resources/deployments API changes

None. The effective configuration influences client compilation and tooling only and is not emitted into the ARM template.

### Examples

#### Shared organization and project configuration

```text
repo/
  eng/
    bicepconfig.organization.json
  services/
    payments/
      bicepconfig.json
      main.bicep
```

```jsonc
// eng/bicepconfig.organization.json
{
  "analyzers": {
    "core": {
      "enabled": true,
      "rules": {
        "no-hardcoded-location": {
          "level": "error"
        },
        "secure-parameter-default": {
          "level": "error"
        }
      }
    }
  },
  "formatting": {
    "indentKind": "Space",
    "indentSize": 2
  }
}
```

```jsonc
// services/payments/bicepconfig.json
{
  "extends": "../../eng/bicepconfig.organization.json",
  "analyzers": {
    "core": {
      "rules": {
        "no-hardcoded-location": {
          "level": "warning"
        }
      }
    }
  }
}
```

`main.bicep` automatically discovers the service configuration, inherits both organization rules and the formatter settings, and lowers only `no-hardcoded-location` to a warning.

#### Nested configuration boundaries

```text
repo/
  eng/
    bicepconfig.base.json
  bicepconfig.json
  main.bicep
  modules/
    bicepconfig.json
    storage.bicep
```

The root configuration extends the shared base and defines repository-wide aliases:

```jsonc
// bicepconfig.json
{
  "extends": "./eng/bicepconfig.base.json",
  "moduleAliases": {
    "br": {
      "company": {
        "registry": "contoso.azurecr.io",
        "modulePath": "bicep/modules"
      }
    }
  }
}
```

The nested module configuration explicitly inherits the root configuration and strengthens one analyzer rule:

```jsonc
// modules/bicepconfig.json
{
  "extends": "../bicepconfig.json",
  "analyzers": {
    "core": {
      "rules": {
        "no-unused-params": {
          "level": "error"
        }
      }
    }
  }
}
```

`main.bicep` discovers the root `bicepconfig.json`. `modules/storage.bicep` independently discovers `modules/bicepconfig.json`, which inherits the root configuration and the shared base transitively. Existing nearest-file boundaries remain intact; inheritance between them occurs only because the nested configuration declares it.

## Tradeoffs

- Effective configuration becomes harder to infer from one file. Editor navigation and diagnostics that include the inheritance chain mitigate this cost.
- Property-level source provenance and transitive file watching add implementation complexity to the configuration manager and language server. Resolving all paths from the leaf would be simpler but would make reusable base configurations location-dependent and would conflict with artifact path semantics.
- Recursive object merging has no delete or atomic-replace operation. A derived configuration cannot remove an inherited object property and can produce an invalid typed object if it changes a discriminator while retaining base properties.
- Relative paths containing `..` can reference base configurations outside the workspace, expanding the set of files the extension must watch and trust. Existing Workspace Trust restrictions continue to apply.

This proposal does not introduce a breaking change for configurations that do not use `extends`.

## Alternatives

### Automatically merge every ancestor `bicepconfig.json`

Bicep could merge all discovered ancestor configurations from the filesystem root to the source directory. This would require less explicit configuration, but it would change existing behavior and allow a newly added file in any ancestor directory to alter builds unexpectedly. It would also make the effective project boundary unclear. Explicit `extends` provides the same layering while recording the dependency in source control.

### Support an array of base configurations

Allowing `"extends": ["base1.json", "base2.json"]` can reduce chain files, but it introduces ordering, duplicate-base, diamond inheritance, and path-provenance questions. A single base supports deliberate layering through a linear chain. Multiple inheritance can be proposed later without changing the string form.

### Use only inline configuration overrides

Inline overrides are suitable for changing a few invocation-specific values. They are not a replacement for a validated, navigable, reusable full configuration file and do not provide inheritance for automatically discovered configurations.

## Rollout plan

This feature requires coordinated client-side changes to Bicep Core, the language server, the configuration schema, and the Visual Studio Code extension. Existing CLI commands consume the behavior through the shared configuration manager without adding command-line options. The feature does not require an ARM feature flag or service deployment.

The feature will ship with the following work:

1. Add inheritance loading, property provenance, cycle diagnostics, cache invalidation, and unit tests to the shared configuration manager.
1. Add `extends` to the configuration schema and provide relative-path completion, navigation, and diagnostics for referenced files.
1. Add CLI integration tests covering local modules, Bicep parameters files, invalid chains, relative path resolution, and configuration-relative property resolution.
1. Add language-server support for transitive configuration dependency watching and invalidation.
1. Document precedence, path bases, automatic-discovery compatibility, and interaction with configuration overrides.

The feature does not need an experimental configuration flag. Such a flag would itself depend on successfully loading the configuration whose loading behavior is being enabled. Use of `extends` is explicit, and configurations that omit it retain their existing behavior. Any telemetry added for the feature must not collect configuration paths or values.

## Unresolved questions

None at this time.

## Out of scope

- Multiple direct bases or diamond inheritance.
- User-wide or machine-wide implicit configuration layers.
- Automatically merging all ancestor `bicepconfig.json` files.
- Explicit configuration selection, including a CLI argument, Visual Studio Code setting or command, environment variable, Bicep parameters association, or source syntax. This can be addressed independently as part of [#5013](https://github.com/Azure/bicep/issues/5013).
- New deletion, array merge, JSON Patch, or atomic-object replacement semantics.
- Defining the inline value syntax and path behavior of `--config-override`, which belongs to the related inline override proposal.
- Changing the configuration used when an external registry artifact was originally compiled.
