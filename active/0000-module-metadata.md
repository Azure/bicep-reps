---
REP Number: "0000"
Author: jaredfholgate (Jared Holgate)
Start Date: 2026-11-05
Feature Status: "Not implemented (draft proposal)"
---

# Module identification metadata

## Summary

Introduce module-generic build inputs that Bicep validates and emits into a reserved `metadata._module` object in the compiled ARM template. The object contains `type`, `code`, and `version`. Module publishers can provide these values through a module-local `metadata.json` file or explicit CLI options. Published modules retain their own metadata when embedded in a consuming template.

The initial supported module type is `avm`, but the input contract and CLI names are generic. This REP covers supplying, validating, emitting, and preserving the data. How that data is subsequently surfaced or collected is intentionally unspecified.

This is an initial design draft. Proposed CLI names, file discovery rules, and validation details below are subject to review; they are not existing Bicep capabilities.

## Terms and definitions

| Term | Meaning |
| --- | --- |
| Module type | An allowlisted module classification, initially `avm` for Azure Verified Modules. Not an ARM resource type. |
| Module code | A publisher-assigned, opaque identifier for a module within a module type. It is not a registry address, resource name, or customer identifier. |
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

#### Output contract

When module identification inputs are present and valid, Bicep emits exactly this object at the root of the template being compiled:

```json
{
  "_module": {
    "type": "avm",
    "code": "abcdef0",
    "version": "v0.1.2"
  }
}
```

The object is a property of the template's existing `metadata` object, alongside `_generator`, `name`, and `description`. It is not part of `_generator`, a resource's `tags`, or the deployment resource's own metadata.

Each template has at most one `_module` object describing that template's module. Dependencies retain their own objects; there is no flattened list on the parent. If there are no inputs, the compiler omits the object rather than emitting nulls, empty strings, or inferred defaults.

The name is reserved for the compiler. Ordinary Bicep `metadata` declarations cannot author or override `_module`. This REP does not relax the existing reserved-name restriction.

#### Input option A: module-local metadata file

Proposed file shape:

```json
{
  "module": {
    "type": "avm",
    "code": "abcdef0",
    "version": "v0.1.2"
  }
}
```

The proposed discovery rule is to look for `metadata.json` immediately beside the Bicep source file being compiled. For a conventional module entry point, this is the module root containing `main.bicep`. There is no search through ancestor directories and no inheritance from a parent module's file.

Only the `module` property is an input to this feature. Existing sibling properties in a repository's metadata file are permitted, ignored by this feature, and never copied into the ARM template. There is no dependency on AVM's repository metadata schema. An existing AVM `metadata.json` does not automatically supply module identification merely because it exists.

File behavior:

- No adjacent file, or a valid JSON object without `module`: emit no module identification for that source.
- A `module` property: validate the complete object against the contract below.
- A discovered file that cannot be read, contains malformed JSON or duplicate keys, or is not a JSON object: fail the build with an actionable diagnostic.
- `module: null`, a partial object, or unknown properties inside that object: fail the build.

The file is a tracked compilation input. Changing or removing it must invalidate affected compilation and emission caches. When referenced local Bicep modules are compiled, each resolves its own adjacent file independently. The compiler must not inspect arbitrary files in the registry cache to derive new module identification for already-compiled external modules.

Automatic discovery is proposed for discussion. Whether discovery should require explicit opt-in, and whether a configurable metadata file path is preferable, remain unresolved.

#### Input option B: explicit CLI values

Proposed syntax:

```powershell
bicep build .\main.bicep `
  --module-type avm `
  --module-code abcdef0 `
  --module-version v0.1.2
```

All three options are module-generic. There is no `--avm-*` option, and the CLI does not silently assume `avm`. The initial allowlist of one type is a validation constraint rather than an AVM-specific input interface.

The options form one atomic input: supply all three or none. Each option may occur only once. Values are literal strings; they are not Bicep expressions, environment-variable references, file contents, or values taken from deployment parameters.

An explicit tuple takes precedence as a whole and bypasses metadata-file discovery for the entry point. It is not merged field-by-field with a file. This lets a release pipeline supply complete release data without depending on repository metadata and avoids mixing a code from one source with a version from another. An invalid explicit tuple fails; it does not fall back to a file.

CLI values apply only to the single entry point selected for that invocation. They never relabel referenced local or registry modules. If a build invocation selects multiple entry points, these options must fail with a diagnostic requesting a separate invocation per module.

#### Validation and normalization

The same validator is used for file inputs and CLI inputs. No field may be null, empty, or contain leading or trailing whitespace. JSON values must have the stated types; coercion is not allowed.

| Attribute | Proposed validation | Emitted value |
| --- | --- | --- |
| `type` | String; exactly `avm` in the initial release. Reject unknown values rather than accepting arbitrary classification text. | `avm` |
| `code` | String; exactly seven ASCII hexadecimal characters, matching `^[0-9A-Fa-f]{7}$`. No prefix, separators, paths, or URLs. | Lowercase seven-character string. |
| `version` | String; numeric release core `MAJOR.MINOR.PATCH`, optionally prefixed by lowercase `v`. Each component is an integer from 0 through 2147483647, without leading zeroes except the value `0`. | Canonical `vMAJOR.MINOR.PATCH`, matching the preferred output shape. |

For version input, the lexical form is `^v?(0|[1-9][0-9]{0,9})\.(0|[1-9][0-9]{0,9})\.(0|[1-9][0-9]{0,9})$`, followed by the numeric range checks. Matching is against the entire string, with no trailing newline accepted.

This version grammar deliberately supports only the numeric release core of SemVer, with an optional input prefix. It does not claim to implement full SemVer: prerelease and build-metadata identifiers are rejected in this draft because they allow arbitrary text. Whether to support a constrained prerelease form, and whether to emit the `v` prefix, must be resolved before stabilization.

The schema is closed: exactly `type`, `code`, and `version` are required, with no additional properties. A future module type requires an explicit allowlist/schema change, not a way to bypass validation.

Diagnostics identify the input source, attribute, and violated rule without echoing the rejected value or the complete metadata object. A URL or accidental secret supplied instead of a code or version must not be repeated in diagnostics. An invalid input produces a non-zero exit code and no successful template emission; an existing output file must not be presented as the result of that failed build.

Module code assignment and uniqueness remain the module publisher's responsibility. The compiler does not resolve a code through a catalog, derive it from a registry address, or check its uniqueness online.

#### Compilation, publishing, and consumption

The publisher workflow is:

1. Supply release identity through the adjacent file or the explicit CLI tuple.
1. Validate and normalize the values before emission.
1. Emit `_module` through the normal template emitter alongside existing metadata, before final template hashing.
1. Publish or otherwise distribute that compiled template without stripping the object.

The feature must not be implemented as a post-build text replacement. Compilation inputs must participate in normal dependency tracking, deterministic output, and template-hash generation.

`bicep publish` can compile Bicep source directly. To avoid a publisher build carrying module identification that is then lost during publication, this draft proposes the same file discovery and explicit tuple on source-based `publish`, scoped to the module being published. The validator and emitter are shared, not reimplemented in command handlers. Publication of an already-compiled ARM JSON module preserves and validates any existing `_module`; these options must not be used to relabel an already-compiled module.

During a consumer build, restored ARM JSON modules retain their own `_module` objects when embedded in `Microsoft.Resources/deployments.properties.template`. Consumer CLI values describe only the consumer entry point and must not overwrite those objects. Other existing template metadata is preserved.

A present reserved object in an external JSON template is validated using the same closed schema before it is emitted into the consumer output. Non-canonical but otherwise valid values can be normalized by a publisher build; already-published templates must contain the canonical output form and are not rewritten during consumption. Invalid reserved metadata is a build error, not silently trusted or stripped. External templates with no such object remain supported without inferred module identification.

This contract applies recursively to inline nested templates, including nested templates already embedded inside a restored module. Reusing the same module more than once preserves its identity on each occurrence. A parent has no module identification object unless its own inputs supply one.

There are no new deployment parameters, resources, resource tags, or runtime expressions. File discovery is a property of compilation, not a request for ARM to read a local file.

#### Validation coverage required for implementation

| Area | Required cases |
| --- | --- |
| Attribute validation | Accepted type, unknown type, missing/extra attributes, wrong JSON types, duplicate keys, nulls, whitespace, and case handling. |
| Code | Exactly seven hex characters; reject six/eight characters, non-ASCII lookalikes, prefixes, separators, and embedded newlines. Verify lowercase normalization. |
| Version | Optional input prefix, canonical output, zero components, leading zeroes, component bounds, overlong values, prerelease/build rejection, and no trailing newline. |
| Discovery and precedence | Absent file/property, malformed/unreadable file, independent sibling/child modules, no ancestor inheritance, complete CLI override, partial/repeated CLI inputs, and multi-entry-point rejection. |
| Emission | Metadata beside `_generator`; preserve existing metadata; absent input leaves baseline output unchanged; deterministic output and normal template-hash behavior. |
| Publication and consumption | Source publication parity with build, compiled JSON publication, restore and consumer build, multiple uses, transitive nested templates, valid/invalid reserved metadata, and no identity propagation from parents. |
| Dependency tracking | File addition, modification, and removal invalidate cached compilation; no stale metadata survives incremental builds. |
| Compatibility and diagnostics | Supported ARM language versions, array- and object-shaped resource collections, older templates without the field, and errors that never echo rejected values. |

### Server side changes (if applicable)

No server-side change is required to carry descriptive metadata in the compiled template. This draft does not define a server-side interpretation of `_module`.

#### Telemetry surfacing

### Microsoft.Resources/deployments API changes (if applicable)

None proposed for carrying the object in existing template metadata.

### Examples

These are proposed output excerpts, not complete deployable templates. The module code and release version are illustrative and are not asserted to be assigned to the Container App module.

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
      "code": "abcdef0",
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
              "code": "abcdef0",
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
1. **Convenient discovery versus hidden inputs.** An adjacent file avoids repetitive command options but adds a compilation dependency and may expose malformed existing files. Exact-directory discovery, dependency tracking, explicit CLI precedence, and a decision on opt-in are necessary mitigations.
1. **Metadata is not provenance.** A reserved name prevents ordinary Bicep declarations from writing the field, but any caller can supply valid CLI inputs or edit ARM JSON. Format validation is neither publisher authentication nor proof that a field contains no sensitive information. Any future collector needs its own trust and validation boundary.
1. **Artifact and compatibility costs.** Each module occurrence adds a small object and changes the compiled artifact. Existing published versions need republishing to gain this field; a consumer build cannot reliably reconstruct it. Strict rejection of malformed reserved objects must be assessed against existing JSON templates and older tooling.

## Alternatives

1. **An integer code.** A bounded non-negative integer could further narrow representation and avoid hexadecimal case normalization. It remains a viable alternative, but this draft proposes seven hexadecimal characters. Changing to an integer would require a decision on its JSON type, bounds, and assignment/migration policy.
1. **AVM-specific options or an implicit type.** Shorter for the initial adopter but couples a general compiler interface to one module system. Generic `--module-type`, `--module-code`, and `--module-version` inputs and an explicit allowlisted type keep the interface extensible.
1. **Ordinary Bicep metadata declarations.** No new input mechanism would be needed, but release automation would have to modify source and the reserved schema would not be enforced as a compiler-managed contract. Descriptive user metadata remains available for other purposes.
1. **Put module identification inside `_generator`.** That conflates module identity with compiler identity. A sibling `_module` object keeps their lifecycles and meanings separate.
1. **Derive identity from a registry reference.** Registry hosts, repository paths, aliases, and tags are not a constrained opaque identity and can change through mirroring or republishing. This also fails to cover locally compiled modules.
1. **Accept arbitrary metadata or full SemVer strings.** More flexible, but introduces free-form text without addressing the reason for strict attribute validation. This proposal starts with a closed object and numeric release core.
1. **Use only a file, only CLI values, or an explicit file path.** Each reduces discovery/precedence complexity or makes inputs more explicit. The draft presents both requested authoring paths; selecting one path or an explicit `--module-metadata-file` input remains open for review.

## Rollout plan

1. Resolve the file-discovery/opt-in policy, CLI spelling, version grammar, and compatibility behavior through this REP.
1. Implement the shared validator and emission path, initially behind an experimental gate. Cover build, source-based publish, compilation dependency tracking, and consumer preservation together.
1. Pilot publisher and consumer workflows, including AVM modules, without claiming that template metadata is being collected or surfaced.
1. Stabilize the input/output contract and document minimum publisher tooling versions. Do not remove any existing AVM telemetry mechanism merely because this field is available.

## Unresolved questions

1. Should adjacent `metadata.json` discovery be automatic, opt-in, or replaced by an explicit path? Which non-CLI compilation entry points should enable discovery, and how should editor/RPC callers supply equivalent explicit inputs?
1. Are `--module-type`, `--module-code`, and `--module-version` the preferred names, and should source-based `publish` expose all of them directly?
1. Is a seven-character hexadecimal code the right long-term representation, or should the contract use a bounded integer? What assignment process prevents accidental code reuse within a type?
1. Should version output retain the preferred `v` prefix or use unprefixed SemVer? Are prerelease versions required initially, and if so, what bounded grammar avoids arbitrary identifiers? Are the proposed numeric bounds appropriate?
1. How should existing malformed `_module` in hand-authored JSON be handled without an unintended compatibility break? What resource/dependency traversal is needed to validate inline templates without treating unrelated JSON objects as templates?
1. What are the required opt-out semantics before any future telemetry collection is enabled? Compile-time identity is static and cannot be conditioned on a deployment parameter such as AVM's existing `enableTelemetry`.
1. What preservation behavior is required across older Bicep versions, decompilation/recompilation, template specs, and alternative module artifact formats? The initial inline-template contract must not imply these paths are already covered.

## Out of scope

- Telemetry collection, transmission, storage, querying, reporting, or its eventual presentation.
- Changes to existing AVM deployment telemetry, opt-out behavior, or a migration away from that mechanism.
- Publisher authentication, signed provenance, online catalog lookups, and enforcement of module-code ownership.
- Support for module types other than `avm` in the initial release.
- Changes to ARM resource tags, deployment resources used for existing telemetry, or resource provider behavior.
- A generic mechanism for copying arbitrary repository metadata into ARM templates.
