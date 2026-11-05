---
REP Number: "0031"
Author: jaredfholgate (Jared Holgate)
Start Date: 2026-11-05
Feature Status: "Not implemented (draft proposal)"
---

# Module identification metadata

## Summary

Introduce module-generic build inputs that Bicep validates and emits into a reserved `metadata._module` object in the compiled ARM template. The object contains `type`, `id`, and `version`. Module publishers can provide the values through explicit CLI options or an explicitly selected metadata file. Published modules retain their own metadata when embedded in a consuming template.

The initial supported module type is `avm`, but the input contract and CLI names are generic. This REP covers supplying, validating, emitting, and preserving the data. How that data is subsequently surfaced or collected is intentionally unspecified.

This is an initial design draft. Proposed CLI names, file-selection rules, and validation details below are subject to review; they are not existing Bicep capabilities.

## Terms and definitions

| Term | Meaning |
| --- | --- |
| Module type | An allowlisted module classification, initially `avm` for Azure Verified Modules. Not an ARM resource type. |
| Module identifier | A publisher-assigned, opaque identifier for a module within a module type. It is not a registry address, resource name, or customer identifier. |
| Module version | The release version of the module, not the Bicep compiler version or ARM `contentVersion`. |
| Publisher build | Compilation of a module before its compiled template is distributed or published to a registry. |
| Consumer build | Compilation of a template that references an already-published module. |
| Reserved metadata | A compiler-managed metadata property with a leading underscore, separate from ordinary Bicep metadata declarations. |

## Motivation

A registry module's reference is used to restore the module during compilation. In the inline deployment model, its template is embedded in the consumer's compiled ARM template; the registry reference is not itself a stable module-identification contract in that output.

Module publishers need a small, consistent way to identify a module and its version in the resulting template, including when nested within another template. Existing descriptive metadata such as `name` and `description` is free-form, is not a constrained identification schema, and should not be repurposed as one.

Release information is often held by publishing automation rather than authored in the Bicep source. Supporting a file and explicit build inputs allows both repository-based authoring and release pipelines without adding publisher-specific syntax to the Bicep language.

The intended outcome is a bounded, machine-readable metadata contract, not an instruction to transmit telemetry or a claim that the module's publisher has been authenticated.

## Detailed design

### Client side changes

#### Existing implementation and required change points

The current [build command](https://github.com/Azure/bicep/blob/2cba03c71c174ea3c811a1c7a7d8e22ec45d98fa/src/Bicep.Cli/Commands/BuildCommand.cs) resolves a file pattern and creates one compilation for each selected entry point. The proposed identity inputs therefore need to flow through the command arguments and into a single compilation; rejecting `--pattern` with any identity input is a deliberate new validation rule, not existing behavior.

The current [template writer](https://github.com/Azure/bicep/blob/2cba03c71c174ea3c811a1c7a7d8e22ec45d98fa/src/Bicep.Core/Emit/TemplateWriter.cs) emits `metadata._generator` before authored metadata. The [metadata symbol validator](https://github.com/Azure/bicep/blob/2cba03c71c174ea3c811a1c7a7d8e22ec45d98fa/src/Bicep.Core/Semantics/MetadataSymbol.cs) already reserves every metadata identifier beginning with `_`. `_module` should use that reserved namespace, but the compiler needs a new compilation input and emitter path; it cannot be produced as an ordinary Bicep metadata declaration.

Nested Bicep modules are emitted by recursively writing their semantic model. Restored ARM JSON modules use an [ARM template writer](https://github.com/Azure/bicep/blob/2cba03c71c174ea3c811a1c7a7d8e22ec45d98fa/src/Bicep.Core/Emit/ArmTemplateWriter.cs) that writes the parsed template object unchanged. Consequently, JSON-module consumption preserves unrecognized top-level metadata, including `_module`, but does not validate it. The proposal validates the closed schema only when a Bicep publisher compilation emits `_module`; consumption deliberately retains pass-through compatibility.

The [source-file grouping](https://github.com/Azure/bicep/blob/2cba03c71c174ea3c811a1c7a7d8e22ec45d98fa/src/Bicep.Core/SourceGraph/SourceFileGroupingBuilder.cs) currently tracks Bicep source files and module artifacts. Bicep also has an [auxiliary-file cache](https://github.com/Azure/bicep/blob/2cba03c71c174ea3c811a1c7a7d8e22ec45d98fa/src/Bicep.Core/SourceGraph/AuxiliaryFileCache.cs) used by `load*Content` functions. An explicitly selected metadata file should reuse that auxiliary-file tracking and invalidation model rather than create a separate cache. Non-CLI callers, such as the language server, need an equivalent explicit input rather than command-only preprocessing.

#### Output contract

When module identification inputs are present and valid, Bicep emits exactly this object at the root of the template being compiled:

```json
{
  "_module": {
    "type": "avm",
    "id": "abcdef0",
    "version": "v0.1.2"
  }
}
```

The object is a property of the template's existing `metadata` object, alongside `_generator`, `name`, and `description`. It is not part of `_generator`, a resource's `tags`, or the deployment resource's own metadata.

Each template has at most one `_module` object describing that template's module. Dependencies retain their own objects; there is no flattened list on the parent. If there are no inputs, the compiler omits the object rather than emitting nulls, empty strings, or inferred defaults.

The name is reserved for the compiler. Ordinary Bicep `metadata` declarations cannot author or override `_module`. This REP does not relax the existing reserved-name restriction.

#### Input option A: explicitly selected metadata file

Proposed file shape:

```json
{
  "module": {
    "type": "avm",
    "id": "abcdef0",
    "version": "v0.1.2"
  }
}
```

The file is used only when selected by the proposed `--module-metadata-file <path>` option:

```powershell
bicep build .\main.bicep --module-metadata-file .\metadata.json
```

There is no automatic search for `metadata.json`, including beside the entry point, in ancestor directories, or beside referenced Bicep files. The file applies only to the invocation entry point. It never supplies identity to local helper modules or restored registry modules.

Only the `module` property is an input to this feature. Existing sibling properties in a selected repository metadata file are permitted, ignored by this feature, and never copied into the ARM template. An AVM metadata file is not automatically compatible: its schema currently rejects unrecognized properties, so adopting this shape requires a coordinated AVM schema change or a separate explicitly selected file.

File behavior:

- Without `--module-metadata-file`, no file is read and no module identification is emitted unless a complete direct CLI tuple is supplied.
- The selected file must contain `module`; validate its complete object against the contract below.
- A selected file that cannot be read, contains malformed JSON or duplicate keys, or is not a JSON object: fail the build with an actionable diagnostic.
- `module: null`, a partial object, or unknown properties inside that object: fail the build.

The selected file is a tracked compilation input. Changing or removing it must invalidate affected compilation and emission caches. The compiler must not inspect arbitrary sibling files or files in the registry cache to derive module identification.

Whether the option name should be `--module-metadata-file` and how non-CLI callers express the same explicit file selection remain unresolved.

#### Input option B: explicit CLI values

Proposed syntax:

```powershell
bicep build .\main.bicep `
  --module-type avm `
  --module-id abcdef0 `
  --module-version v0.1.2
```

All three options are module-generic. There is no `--avm-*` option, and the CLI does not silently assume `avm`. The initial allowlist of one type is a validation constraint rather than an AVM-specific input interface.

Without `--module-metadata-file`, the options form one atomic input: supply all three or none. Each option may occur only once. Values are literal strings; they are not Bicep expressions, environment-variable references, file contents, or values taken from deployment parameters.

With `--module-metadata-file`, the selected file supplies the complete object and `--module-type` or `--module-id` are rejected. `--module-version` is permitted as the only targeted override: it replaces the file's `module.version` before the complete resulting object is validated. This lets publishing automation supply the release version without duplicating stable type and identifier values. A partial direct tuple, an invalid override, or a combination outside these rules fails without falling back to another source.

CLI values and a selected metadata file apply only to the single entry point selected for that invocation. They never relabel referenced local or registry modules. `--pattern` is rejected when any module identity option is present, even if it happens to select one file, so behavior is deterministic.

#### Validation and normalization

The same validator is used for file inputs and CLI inputs. No field may be null, empty, or contain leading or trailing whitespace. JSON values must have the stated types; coercion is not allowed.

| Attribute | Proposed validation | Emitted value |
| --- | --- | --- |
| `type` | String; exactly `avm` in the initial release. Reject unknown values rather than accepting arbitrary classification text. | `avm` |
| `id` | String; exactly seven ASCII hexadecimal characters, matching `^[0-9A-Fa-f]{7}$`. No prefix, separators, paths, or URLs. | Lowercase seven-character string. |
| `version` | String; numeric release core `MAJOR.MINOR.PATCH`, optionally prefixed by lowercase `v`. Each component is an integer from 0 through 2147483647, without leading zeroes except the value `0`. | Canonical `vMAJOR.MINOR.PATCH`, matching the preferred output shape. |

For version input, the lexical form is `^v?(0|[1-9][0-9]{0,9})\.(0|[1-9][0-9]{0,9})\.(0|[1-9][0-9]{0,9})$`, followed by the numeric range checks. Matching is against the entire string, with no trailing newline accepted.

This version grammar deliberately supports only the numeric release core of SemVer, with an optional input prefix. It does not claim to implement full SemVer: prerelease and build-metadata identifiers are rejected in this draft because they allow arbitrary text. Whether to support a constrained prerelease form, and whether to emit the `v` prefix, must be resolved before stabilization.

The schema is closed: exactly `type`, `id`, and `version` are required, with no additional properties. A future module type requires an explicit allowlist/schema change, not a way to bypass validation.

Diagnostics identify the input source, attribute, and violated rule without echoing the rejected value or the complete metadata object. A URL or accidental secret supplied instead of an identifier or version must not be repeated in diagnostics. An invalid input produces a non-zero exit code and no successful template emission; an existing output file must not be presented as the result of that failed build.

Module identifier assignment and uniqueness within a module type remain the module publisher's responsibility. The compiler does not resolve an identifier through a catalog, derive it from a registry address, or check its uniqueness online.

#### Compilation, publishing, and consumption

The publisher workflow is:

1. Supply release identity through the explicitly selected file and optional version override, or the explicit CLI tuple.
1. Validate and normalize the values before emission.
1. Emit `_module` through the normal template emitter alongside existing metadata, before final template hashing.
1. Publish or otherwise distribute that compiled template without stripping the object.

The feature must not be implemented as a post-build text replacement. The current [template writer](https://github.com/Azure/bicep/blob/2cba03c71c174ea3c811a1c7a7d8e22ec45d98fa/src/Bicep.Core/Emit/TemplateWriter.cs) computes `metadata._generator.templateHash` from the generated template before inserting the hash value itself. Emitting `_module` before that calculation gives each generated artifact a hash that matches its own content. Adding or changing `_module` therefore produces a different valid template hash; it does not leave a hash valid for the prior content.

`bicep publish` can compile Bicep source directly, then publish the emitted template. This draft proposes the same explicit file selection, direct tuple, and file-plus-version-override behavior for source-based `publish`, scoped to the module being published. The validator and emitter are shared, not reimplemented in command handlers.

Today, publishing an already-compiled ARM JSON module reads and uploads its bytes directly. It does not parse or validate `_module`. The compiler validates the schema when it emits `_module` from Bicep source; publishing an existing JSON artifact preserves it unchanged. The options must not relabel an already-compiled module.

During a consumer build, the existing ARM JSON writer preserves each restored module's top-level metadata when it is embedded in `Microsoft.Resources/deployments.properties.template`. Consumer CLI values describe only the consumer entry point and must not overwrite those objects. Consumer builds do not validate, normalize, remove, or otherwise rewrite `_module`: an older compiler must preserve future types, fields, and version formats that it does not understand.

External templates with no `_module` object remain supported without inferred module identification. A publisher that needs compiler validation must publish from Bicep source with an identity input; this REP does not add a compatibility-breaking validator for existing JSON artifacts.

This contract applies recursively to inline nested templates, including nested templates already embedded inside a restored module. Reusing the same module more than once preserves its identity on each occurrence. A parent has no module identification object unless its own inputs supply one.

There are no new deployment parameters, resources, resource tags, or runtime expressions. Explicit file selection is a property of compilation, not a request for ARM to read a local file.

#### Release-version workflow

MAR currently stores `MAJOR.MINOR` in `version.json` and calculates `PATCH` from existing release tags. Its release workflow stamps the final version into a local publishing copy of `main.bicep`, then invokes `bicep publish` on that Bicep source. The published module is therefore freshly compiled after version insertion; the workflow does not patch compiled `main.json`.

For this feature, release automation must provide the final version to a fresh Bicep compilation before template-hash generation. A source-controlled development build may use `v0.0.0` as an illustrative file value, then use `--module-version` for the release compilation. This is optional workflow guidance, not a compiler default or a new source-control requirement. It is valid for a checked-in development artifact and a published release artifact to have different module versions and different template hashes, provided each was freshly compiled and carries the hash for its own content.

An implementation must not replace `_module.version` in already-compiled JSON. That would leave the existing `_generator.templateHash` describing different content unless the whole template is regenerated through the compiler.

#### Validation coverage required for implementation

| Area | Required cases |
| --- | --- |
| Attribute validation | Accepted type, unknown type, missing/extra attributes, wrong JSON types, duplicate keys, nulls, whitespace, and case handling. |
| Identifier | Exactly seven hex characters; reject six/eight characters, non-ASCII lookalikes, prefixes, separators, and embedded newlines. Verify lowercase normalization. |
| Version | Optional input prefix, canonical output, zero components, leading zeroes, component bounds, overlong values, prerelease/build rejection, and no trailing newline. |
| File selection and precedence | No file read without the option, malformed/unreadable selected file, missing `module`, no helper-module propagation, selected-file plus version override, rejected file plus type/id options, partial/repeated CLI inputs, and `--pattern` rejection with any identity input. |
| Emission | Metadata beside `_generator`; preserve existing metadata; absent input leaves baseline output unchanged; deterministic output and normal template-hash behavior. |
| Publication and consumption | Source publication parity with build, compiled JSON preservation, restore and consumer build, multiple uses, transitive nested templates, unknown future metadata preservation, and no identity propagation from parents. |
| Dependency tracking | Selected-file addition, modification, and removal invalidate cached compilation through the auxiliary-file mechanism; no stale metadata survives incremental builds. |
| Compatibility and diagnostics | Supported ARM language versions, array- and object-shaped resource collections, older templates without the field, and errors that never echo rejected values. |

### Server side changes (if applicable)

No server-side change is required to carry descriptive metadata in the compiled template. This draft does not define a server-side interpretation of `_module`.

#### Telemetry surfacing

### Microsoft.Resources/deployments API changes (if applicable)

None proposed for carrying the object in existing template metadata.

### Examples

These are proposed output excerpts, not complete deployable templates. The module identifier and release version are illustrative and are not asserted to be assigned to the Container App module.

#### Container App module compiled by its publisher

The existing [Container App compiled template](https://github.com/Azure/bicep-registry-modules/blob/main/avm/res/app/container-app/main.json) contains descriptive metadata. With validated module identification inputs, its top-level metadata would have this shape:

```json
{
  "metadata": {
    "_generator": {
      "name": "bicep",
      "version": "<compiler-version>",
      "templateHash": "<generated-template-hash>"
    },
    "_module": {
      "type": "avm",
      "id": "abcdef0",
      "version": "v0.1.2"
    },
    "name": "Container Apps",
    "description": "This module deploys a Container App."
  }
}
```

#### Container App module embedded in a consumer template

For a consumer using language version `2.0` and symbolic resource names, the corresponding excerpt is:

```json
{
  "languageVersion": "2.0",
  "resources": {
    "containerAppDeployment": {
      "type": "Microsoft.Resources/deployments",
      "apiVersion": "2022-09-01",
      "name": "containerAppDeployment",
      "properties": {
        "template": {
          "metadata": {
            "_generator": {
              "name": "bicep",
              "version": "<compiler-version>",
              "templateHash": "<generated-template-hash>"
            },
            "_module": {
              "type": "avm",
              "id": "abcdef0",
              "version": "v0.1.2"
            },
            "name": "Container Apps",
            "description": "This module deploys a Container App."
          }
        }
      }
    }
  }
}
```

For array-shaped ARM `resources`, the same metadata is located under the matching deployment resource's `properties.template.metadata`. It is never moved to the consumer template's top-level metadata merely because the consumer references the module.

## Tradeoffs

1. **Constrained identity versus flexibility.** Seven hex characters and numeric-only release versions limit free-form text but cannot represent every publisher's identifier or full SemVer. Explicit schema extensions are preferable to an arbitrary string escape hatch.
1. **Explicit configuration versus convenience.** Selecting a file makes the compilation input visible and avoids breaking unrelated `metadata.json` files, but requires a command option. Auxiliary-file tracking and a targeted version override keep the published build deterministic without requiring release automation to duplicate stable values.
1. **Metadata is not provenance.** A reserved name prevents ordinary Bicep declarations from writing the field, but any caller can supply valid CLI inputs or edit ARM JSON. Format validation is neither publisher authentication nor proof that a field contains no sensitive information. Any future collector needs its own trust and validation boundary.
1. **Artifact and compatibility costs.** Each module occurrence adds a small object and changes the compiled artifact. Existing published versions need republishing to gain this field; a consumer build cannot reliably reconstruct it. Strict rejection of malformed reserved objects must be assessed against existing JSON templates and older tooling.

## Alternatives

1. **An integer identifier.** A bounded non-negative integer could further narrow representation and avoid hexadecimal case normalization. It remains a viable alternative, but this draft proposes seven hexadecimal characters. Changing to an integer would require a decision on its JSON type, bounds, and assignment/migration policy.
1. **AVM-specific options or an implicit type.** Shorter for the initial adopter but couples a general compiler interface to one module system. Generic `--module-type`, `--module-id`, and `--module-version` inputs and an explicit allowlisted type keep the interface extensible.
1. **Ordinary Bicep metadata declarations.** No new input mechanism would be needed, but release automation would have to modify source and the reserved schema would not be enforced as a compiler-managed contract. Descriptive user metadata remains available for other purposes.
1. **Put module identification inside `_generator`.** That conflates module identity with compiler identity. A sibling `_module` object keeps their lifecycles and meanings separate.
1. **Derive identity from a registry reference.** Registry hosts, repository paths, aliases, and tags are not a constrained opaque identity and can change through mirroring or republishing. This also fails to cover locally compiled modules.
1. **Accept arbitrary metadata or full SemVer strings.** More flexible, but introduces free-form text without addressing the reason for strict attribute validation. This proposal starts with a closed object and numeric release core.
1. **Use only a file or only CLI values.** Either reduces precedence rules, but requires either source-controlled release data or repetitive release arguments. The draft retains both paths, with explicit file selection and only a version override.

## Rollout plan

1. Resolve the explicit-file option spelling, version grammar, source-archive behavior, and non-CLI input shape through this REP.
1. Implement the shared validator and emission path, initially behind an experimental gate. Cover build, source-based publish, compilation dependency tracking, and consumer preservation together.
1. Pilot publisher and consumer workflows, including AVM modules, without claiming that template metadata is being collected or surfaced.
1. Stabilize the input/output contract and document minimum publisher tooling versions. Do not remove any existing AVM telemetry mechanism merely because this field is available.

## Unresolved questions

1. Is `--module-metadata-file` the preferred option name, and how should editor/RPC callers supply the equivalent explicit file input?
1. Are `--module-type`, `--module-id`, and `--module-version` the preferred names, and should source-based `publish` expose the same file-plus-version-override behavior directly?
1. Is a seven-character hexadecimal identifier the right long-term representation, or should the contract use a bounded integer? What assignment process prevents accidental identifier reuse within a type?
1. Should version output retain the preferred `v` prefix or use unprefixed SemVer? Are prerelease versions required initially, and if so, what bounded grammar avoids arbitrary identifiers? Are the proposed numeric bounds appropriate?
1. How should a publisher validate or document a hand-authored compiled JSON module without making consumer builds reject or rewrite it?
1. What are the required opt-out semantics before any future telemetry collection is enabled? Compile-time identity is static and cannot be conditioned on a deployment parameter such as AVM's existing `enableTelemetry`.
1. What preservation behavior is required across older Bicep versions, decompilation/recompilation, template specs, alternative module artifact formats, and `--with-source` source archives? The initial inline-template contract must not imply these paths are already covered.

## Out of scope

- Telemetry collection, transmission, storage, querying, reporting, or its eventual presentation.
- Changes to existing AVM deployment telemetry, opt-out behavior, or a migration away from that mechanism.
- Publisher authentication, signed provenance, online catalog lookups, and enforcement of module identifier ownership.
- Support for module types other than `avm` in the initial release.
- Changes to ARM resource tags, deployment resources used for existing telemetry, or resource provider behavior.
- A generic mechanism for copying arbitrary repository metadata into ARM templates.
