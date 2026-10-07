---
REP Number: "0029"
Author: shenglol (Shenglong Li)
Start Date: 2026-10-07
Feature Status: Public
Bicep Issue Number(s): "[#5013](https://github.com/Azure/bicep/issues/5013)"
Enhances: "0023"
---

# Bicep CLI `--config` option

## Summary

This proposal adds an optional `--config <path>` argument to Bicep CLI commands that read a Bicep configuration. When specified, the given file replaces automatic `bicepconfig.json` discovery for every source file processed by that invocation. The file is the leaf configuration, so it can use `extends` ([REP 0023](../final/0023-bicep-configuration-inheritance.md)) and is merged over the built-in configuration like any discovered configuration.

```text
bicep build ./main.bicep --config ./bicepconfig.prod.json
```

## Terms and definitions

- **Discovered configuration**: The nearest file named `bicepconfig.json` found by searching from a source file's directory through its ancestors.
- **Leaf configuration**: The configuration from which an inheritance chain starts (see REP 0023).
- **Explicit configuration**: The file passed with `--config`.
- **Effective configuration**: The result of merging the built-in configuration with the leaf configuration and its `extends` chain.

## Motivation

This proposal addresses [Azure/bicep#5013](https://github.com/Azure/bicep/issues/5013). Today the only way to influence which configuration Bicep uses is to place a file named `bicepconfig.json` in an ancestor directory of the source file. That does not work for several common scenarios:

- **Per-environment settings**: Building the same template for test and production against different module registries or aliases ([discussions#10712](https://github.com/Azure/bicep/discussions/10712)), or with different analyzer levels.
- **Centrally managed linting in CI**: A pipeline wants to enforce a common analyzer configuration that lives outside the project, such as in the engineering-system directory of a monorepo, without copying it into every project.
- **Build systems that generate or relocate files**: The configuration is not an ancestor of the source files being compiled.

REP 0023 lets configurations share settings through `extends`, but discovery still decides which file is the starting point. A CLI option gives callers explicit control over that starting point.

## Detailed design

### Client side changes

#### Option

Commands that load a Bicep configuration accept:

```text
--config <path>
```

`<path>` is a local file path, absolute or relative to the current working directory. It may have any file name. The file must exist and be valid configuration, otherwise the command fails with an error diagnostic and a non-zero exit code. The option may be specified at most once.

Supported commands: `build`, `build-params`, `lint`, `format`, `publish`, `publish-extension`, `restore`, `generate-params`, `snapshot`, `test`, `jsonrpc` (per-request, see below), and any later command that compiles Bicep or Bicep parameters files. Commands that do not read configuration (for example `--version`) do not accept it. Commands that already take multiple inputs (such as `build` with several files or globs) apply the one explicit configuration to all of them.

#### Resolution

When `--config` is specified, the explicit configuration is used as the leaf configuration for **every** source file loaded by the invocation, in place of discovery. This includes the entrypoint, referenced local modules, `using` targets of parameters files, and `extends` bases of the explicit file. Discovered `bicepconfig.json` files are ignored.

Resolution is:

```text
effective = merge(builtIn, base2, base1, explicitConfig, invocationOverride)
```

where `invocationOverride` is the optional inline override proposed in [REP 0020](https://github.com/Azure/bicep-reps/pull/20), which keeps highest precedence.

Replacing rather than layering over discovered files makes the result independent of where sources happen to live, which is the property CI and per-environment scenarios need. Callers that want to retain project-specific settings can have the explicit file `extends` the project's configuration.

Relative paths inside the explicit file or its bases (for example local module aliases or `extends`) are resolved from the directory of the file that declares them, per REP 0023, not from the working directory.

Configuration files are treated as inputs of the invocation: a file passed with `--config` and every file it extends participates in the same caching and dependency tracking as discovered configuration files.

#### Scope of effect

Explicit configuration affects the same settings that file-based configuration does: analyzers, formatting, module and extension aliases, registry credentials options, experimental features, and so on. Because it replaces configuration for referenced modules too, restored external modules are not affected: modules published to a registry have already been compiled and their configuration is not consulted.

#### Language server and `jsonrpc`

The language server continues to use discovery. The `jsonrpc` interface accepts an optional `configPath` field on requests that compile or lint files, with the same semantics as `--config`. Editor support for choosing a configuration is out of scope (see below).

### Server side changes (if applicable)

None.

### Microsoft.Resources/deployments API changes (if applicable)

None.

### Examples

- Build for production with a registry alias specific to production:

```text
bicep build ./main.bicep --config ./bicepconfig.prod.json
```

```jsonc
// bicepconfig.prod.json
{
  "extends": "./bicepconfig.json",
  "moduleAliases": {
    "br": {
      "shared": { "registry": "prodacr.azurecr.io", "modulePath": "bicep/modules" }
    }
  }
}
```

- Enforce a central lint policy in CI from a monorepo-wide location:

```text
bicep lint ./services/api/main.bicep --config ./eng/common/bicepconfig.json
```

- Missing file:

```text
bicep build ./main.bicep --config ./missing.json
error BCP...: Could not find configuration file "./missing.json".
```

## Tradeoffs

- **Replaces, not merges with, discovered files.** Callers who pass `--config` lose nested project configuration unless the explicit file extends it. This is predictable but can surprise users who expect an overlay. Mitigation: documentation and the `extends` example above.
- **Editor and CLI can diverge.** The language server uses discovery, so a build with `--config` can produce different diagnostics than the editor. Mitigation: this is inherent to environment-specific configuration, and the options in "Out of scope" can close the gap later.
- **One configuration for the whole module graph.** Local modules that deliberately carry their own `bicepconfig.json` are overridden for the invocation, which may be unwanted in some repositories.
- **Larger CLI surface.** Many commands gain an option, all needing consistent behavior and tests.

## Alternatives

- **Overlay on discovered configuration.** Merge the explicit file over the discovered one. This preserves project settings, but results depend on file locations, array and object merge semantics make it hard to remove a setting, and it does not satisfy the "central policy" scenario. Callers can obtain it with `extends`.
- **Apply only to the entrypoint.** Modules would keep discovering their own configuration. This is hard to reason about, as aliases and analyzer rules would differ across a single compilation, and is inconsistent with how the entrypoint's aliases resolve its modules.
- **Environment variable (for example `BICEP_CONFIG`)**. Easy for CI, but implicit and harder to debug. It could be added later with `--config` taking precedence.
- **Inline overrides only (REP 0020).** Good for a few values, but unwieldy for whole shared files.
- **Wait for a redesigned parameters file format**, as suggested in the issue discussion, so environment settings live in parameters. Configuration (analyzers, registries) and deployment parameters are different concerns, and the CLI-level need exists independent of that redesign.

## Rollout plan

Implement in the Bicep CLI and the `jsonrpc` interface as a general feature, without an experimental flag, since it is opt-in and changes nothing for invocations that omit it. Document the option in each command's help text and on the configuration documentation page. No ARM feature flag is needed.

## Unresolved questions

- Should the option be accepted for `build` with multiple inputs only when all inputs are intended to share one configuration, or should per-input configuration be possible?
- Should an explicit configuration that cannot be found fall back to discovery with a warning? This proposal fails instead to avoid silently using the wrong settings.
- Should remote (`https://` or registry) configuration locations be supported? This proposal limits the option to local files, consistent with REP 0023's local-only `extends`.

## Out of scope

- Editor and language server selection of a configuration file, including a VS Code setting.
- An environment variable or user-level default configuration location.
- Merging multiple `--config` files; `extends` covers composition.
- Changes to the discovery algorithm.
