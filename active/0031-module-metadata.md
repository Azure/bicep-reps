---
REP Number: "0031"
Author: jaredfholgate (Jared Holgate)
Start Date: 2026-11-05
Feature Status: "Not implemented (draft proposal)"
---

# Module identification metadata

## Summary

Introduce optional module-generic build inputs that Bicep validates and emits into a reserved `metadata._module` object in the compiled ARM template. The object contains `type`, `id`, and `version`. Module publishers provide the values through explicit CLI options or the equivalent JSON-RPC compile input. If no input is supplied, Bicep creates no `_module` object and behavior remains unchanged. Existing compiled JSON without `_module` also remains unchanged; when `_module` is present, publication and consumption validate its closed canonical schema before preserving it in the resulting template.

The initial supported module type is `avm`, but the input contract and CLI names are generic. This REP covers supplying, validating, emitting, and preserving the data. How that data is subsequently surfaced or collected is intentionally unspecified.

This is an initial design draft. Proposed CLI names and validation details below are subject to review; they are not existing Bicep capabilities.

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

Embedding the validated identity in compiled ARM JSON carries it through build, artifact, and deployment workflows to ARM, where it is available to read during deployment. This avoids relying on source-only metadata, CLI state, or an external sidecar that may be absent from the deployment payload. This REP defines availability only: it does not define an ARM reader, collection, interpretation, transmission, storage, or surfacing mechanism.

Release information is often held by publishing automation rather than authored in the Bicep source. Explicit build inputs let release pipelines supply it without adding publisher-specific syntax to the Bicep language.

The intended outcome is a bounded, machine-readable metadata contract, not an instruction to transmit telemetry or a claim that the module's publisher has been authenticated.

## Detailed design

### Client side changes

#### Existing implementation and required change points

The current [build command](https://github.com/Azure/bicep/blob/2cba03c71c174ea3c811a1c7a7d8e22ec45d98fa/src/Bicep.Cli/Commands/BuildCommand.cs) resolves a file pattern and creates one compilation for each selected entry point. The proposed identity inputs therefore need to flow through the command arguments and into a single compilation; rejecting `--pattern` with any identity input is a deliberate new validation rule, not existing behavior.

The current [template writer](https://github.com/Azure/bicep/blob/2cba03c71c174ea3c811a1c7a7d8e22ec45d98fa/src/Bicep.Core/Emit/TemplateWriter.cs) emits `metadata._generator` before authored metadata. The [metadata symbol validator](https://github.com/Azure/bicep/blob/2cba03c71c174ea3c811a1c7a7d8e22ec45d98fa/src/Bicep.Core/Semantics/MetadataSymbol.cs) already reserves every metadata identifier beginning with `_`. `_module` should use that reserved namespace, but the compiler needs a new compilation input and emitter path; it cannot be produced as an ordinary Bicep metadata declaration.

Nested Bicep modules are emitted by recursively writing their semantic model. Restored ARM JSON modules use an [ARM template writer](https://github.com/Azure/bicep/blob/2cba03c71c174ea3c811a1c7a7d8e22ec45d98fa/src/Bicep.Core/Emit/ArmTemplateWriter.cs) that writes the parsed template object unchanged. The proposal adds validation before that write: when `_module` is present in a compiled JSON artifact, validate it on raw JSON publication and at consumer restore boundaries, including embedded inline nested templates. A missing object is ignored. A valid object is retained without normalization or rewrite; an invalid object fails the operation.

The [CLI JSON-RPC protocol](https://github.com/Azure/bicep/blob/main/src/Bicep.Cli/Rpc/ICliJsonRpcProtocol.cs) currently exposes `bicep/compile` with a path and optional configuration path, plus `bicep/compileParams`; it has no JSON-RPC publish operation. The initial implementation must extend `bicep/compile` for `.bicep` entry points with the optional structured module tuple. It must use the shared validator and emitter, apply only to that entry point, and never propagate to referenced modules. `bicep/compileParams` produces deployment parameters rather than a module template and does not receive this input. This requires protocol and compiler coverage, not an editor UI change.

#### Wrapper integration

The current [Azure CLI Bicep commands](https://learn.microsoft.com/en-us/cli/azure/bicep#az-bicep-publish) expose `az bicep build` and `az bicep publish`, but do not yet expose the proposed identity options. Follow-up Azure CLI work must add named `--module-type`, `--module-id`, and `--module-version` options to both commands, forward them to Bicep, update help and tests, and enforce the minimum Bicep version that supports them with consistent unsupported-binary errors.

The current [Publish-AzBicepModule](https://learn.microsoft.com/en-us/powershell/module/az.resources/publish-azbicepmodule) cmdlet likewise does not expose identity parameters. Follow-up Azure PowerShell work must add `-ModuleType`, `-ModuleId`, and `-ModuleVersion`, forward them to Bicep, and cover help, tests, and minimum-version behavior. No public Azure PowerShell build cmdlet is identified; callers building from PowerShell continue to use standalone `bicep build`.

Shared Bicep core owns validation and emission; wrappers only provide argument plumbing, help, tests, and capability handling. This requires separate Azure/azure-cli and Azure/azure-powershell changes. Wrapper parity is not an initial rollout or GA prerequisite: MAR invokes standalone `bicep publish` directly, so day-one AVM adoption uses standalone Bicep build and publish. Existing wrapper commands continue to work without identity inputs; callers that need them use standalone Bicep until the wrapper updates ship.

#### Output contract

When module identification inputs are present and valid, Bicep emits exactly this object at the root of the template being compiled:

```json
{
  "_module": {
    "type": "avm",
    "id": "abcdef0",
    "version": "0.1.2"
  }
}
```

The object is a property of the template's existing `metadata` object, alongside `_generator`, `name`, and `description`. It is not part of `_generator`, a resource's `tags`, or the deployment resource's own metadata.

Each template has at most one `_module` object describing that template's module. Dependencies retain their own objects; there is no flattened list on the parent. Identity input is never required: if there are no inputs, the compiler omits the object rather than emitting nulls, empty strings, inferred defaults, or any other new metadata.

The name is reserved for the compiler. Ordinary Bicep `metadata` declarations cannot author or override `_module`. This REP does not relax the existing reserved-name restriction.

#### Input option: explicit CLI values

Proposed syntax:

```powershell
bicep build .\main.bicep `
  --module-type avm `
  --module-id abcdef0 `
  --module-version 0.1.2
```

All three options are module-generic. There is no `--avm-*` option, and the CLI does not silently assume `avm`. The initial allowlist of one type is a validation constraint rather than an AVM-specific input interface.

The options form one optional atomic input: supply all three or none. Each option may occur only once. When none are supplied, the build and output remain unchanged from today. Values are literal strings; they are not Bicep expressions, environment-variable references, file contents, or values taken from deployment parameters. A partial tuple or invalid value fails without falling back to another source.

CLI values apply only to the single entry point selected for that invocation. They never relabel referenced local or registry modules. `--pattern` is rejected when any module identity option is present, even if it happens to select one file, so behavior is deterministic.

#### Input option: JSON-RPC compile request

The optional `module` tuple on a `bicep/compile` request contains exactly `type`, `id`, and `version`. It has the same all-or-none validation and normalization as the CLI tuple. An absent tuple emits no `_module` object and preserves the request's existing behavior; a partial tuple fails. It is valid only when the request path is a `.bicep` entry point; it is not inherited by helper or dependency modules.

#### Validation and normalization

The shared validator applies to CLI and JSON-RPC inputs. No field may be null, empty, or contain leading or trailing whitespace. Values are strings; coercion is not allowed.

| Attribute | Proposed validation | Emitted value |
| --- | --- | --- |
| `type` | String; exactly `avm` in the initial release. Reject unknown values rather than accepting arbitrary classification text. | `avm` |
| `id` | String; exactly seven ASCII hexadecimal characters, matching `^[0-9A-Fa-f]{7}$`. No prefix, separators, paths, or URLs. | Lowercase seven-character string. |
| `version` | String; numeric release core `MAJOR.MINOR.PATCH`, optionally prefixed by lowercase `v`. Each component is an integer from 0 through 2147483647, without leading zeroes except the value `0`. | Canonical `MAJOR.MINOR.PATCH`. |

The seven-character hexadecimal identifier deliberately bounds metadata size and excludes free-form module names, URLs, customer names, and other identifying strings that could otherwise be accidentally carried in ARM or deployment telemetry. Its `16^7` (268,435,456) possible values per type provide ample space for collision-free allocation at expected catalog scale, with uniqueness enforced by the publisher. It constrains accidental exposure and payload size; it is not proof that an identifier is non-sensitive or impossible to encode identifying data.

For version input, the lexical form is `^v?(0|[1-9][0-9]{0,9})\.(0|[1-9][0-9]{0,9})\.(0|[1-9][0-9]{0,9})$`, followed by the numeric range checks. Matching is against the entire string, with no trailing newline accepted.

This initial version grammar deliberately supports only the numeric release core of SemVer. It accepts an optional lowercase input prefix and emits the unprefixed canonical form. It does not claim to implement full SemVer: prerelease and build-metadata identifiers are rejected initially because they allow arbitrary text.

The schema is closed: exactly `type`, `id`, and `version` are required, with no additional properties. A future module type requires an explicit allowlist/schema change, not a way to bypass validation.

For a `_module` object already present in compiled JSON, the same closed schema is required in its canonical emitted form: `type` is `avm`, `id` is seven lowercase ASCII hexadecimal characters, and `version` is an unprefixed numeric `MAJOR.MINOR.PATCH` version within the specified bounds. CLI and JSON-RPC inputs may use the accepted uppercase hexadecimal or lowercase `v` input forms because the compiler canonicalizes them before emission. Existing JSON is never normalized or rewritten, so noncanonical values are rejected rather than changed.

Diagnostics identify the input source or artifact boundary, attribute, and violated rule without echoing the rejected value or the complete metadata object. A URL or accidental secret supplied instead of an identifier or version must not be repeated in diagnostics. An invalid input or present `_module` object produces a non-zero exit code and no successful template emission or publication; an existing output file must not be presented as the result of that failed build. This validation reduces accidental data exposure; it does not prove that accepted values are non-sensitive.

Module identifier assignment and uniqueness within a module type remain the module publisher's responsibility, including detecting duplicate `(type, id)` entries in its maintained mapping at assignment time. Reporting consumers can resolve an opaque `(type, id)` to a module name through that externally maintained mapping, consistent with current AVM reporting. That lookup is not part of this compiler contract: the compiler does not resolve an identifier through a catalog, derive it from a registry address, or check its uniqueness online.

#### Compilation, publishing, and consumption

When a complete identity tuple is supplied, the publisher workflow is:

1. Supply release identity through the explicit CLI tuple, including source-based `bicep publish`, or through the equivalent JSON-RPC tuple when compiling with `bicep/compile`.
1. Validate and normalize the values before emission.
1. Emit `_module` through the normal template emitter alongside existing metadata, before final template hashing.
1. Publish or otherwise distribute that compiled template without stripping the object.

The feature must not be implemented as a post-build text replacement. The current [template writer](https://github.com/Azure/bicep/blob/2cba03c71c174ea3c811a1c7a7d8e22ec45d98fa/src/Bicep.Core/Emit/TemplateWriter.cs) computes `metadata._generator.templateHash` from the generated template before inserting the hash value itself. Emitting `_module` before that calculation gives each generated artifact a hash that matches its own content. Adding or changing `_module` therefore produces a different valid template hash; it does not leave a hash valid for the prior content.

`bicep publish` can compile Bicep source directly, then publish the emitted template. This draft proposes the same direct tuple for source-based `publish`, scoped to the module being published. The validator and emitter are shared, not reimplemented in command handlers.

Today, publishing an already-compiled ARM JSON module reads and uploads its bytes directly. This proposal adds validation only when that artifact contains `_module`: a missing object remains a no-op, while a present object must satisfy the closed canonical schema. A valid artifact is published without metadata normalization, rewrite, or template-hash adjustment; an invalid object is rejected before publication. The options must not relabel an already-compiled module.

During a consumer build, the existing ARM JSON writer preserves each restored module's top-level metadata when it is embedded in `Microsoft.Resources/deployments.properties.template`. Consumer CLI values describe only the consumer entry point and must not overwrite those objects. A consumer validates every present `_module` object in restored registry templates and embedded inline nested templates against the closed canonical schema. Valid values are retained unchanged; missing objects remain untouched; invalid, unknown-type, unknown-field, or noncanonical objects fail the build without being normalized or rewritten. Future schema or type additions therefore require consumers to upgrade Bicep before they can consume artifacts that use them.

External templates with no `_module` object remain supported without inferred module identification, validation failure, or any other action. A publisher that needs compiler-generated identity must publish from Bicep source with an identity input.

This contract applies recursively to inline nested templates, including nested templates already embedded inside a restored module. Reusing the same module more than once preserves its identity on each occurrence. A parent has no module identification object unless its own inputs supply one.

There are no new deployment parameters, resources, resource tags, or runtime expressions.

#### Release-version workflow

MAR currently stores `MAJOR.MINOR` in `version.json` and calculates `PATCH` from existing release tags. Its release workflow stamps the final version into a local publishing copy of `main.bicep`, then invokes `bicep publish` on that Bicep source. The published module is therefore freshly compiled after version insertion; the workflow does not patch compiled `main.json`.

For this feature, release automation must provide the stable type and identifier plus the final version to a fresh Bicep compilation before template-hash generation. A development build may supply `0.0.0` in the explicit tuple; this is optional workflow guidance, not a compiler default or a new source-control requirement. It is valid for a checked-in development artifact and a published release artifact to have different module versions and different template hashes, provided each was freshly compiled and carries the hash for its own content.

An implementation must not replace `_module.version` in already-compiled JSON. That would leave the existing `_generator.templateHash` describing different content unless the whole template is regenerated through the compiler.

#### Validation coverage required for implementation

| Area | Required cases |
| --- | --- |
| Attribute validation | Accepted type, unknown type, missing/partial/repeated CLI values, missing/partial JSON-RPC tuples, nulls, whitespace, and case handling. |
| Identifier | Exactly seven hex characters; reject six/eight characters, non-ASCII lookalikes, prefixes, separators, and embedded newlines. Verify lowercase normalization. |
| Version | Accept both `0.1.2` and `v0.1.2`; emit canonical `0.1.2`; cover zero components, leading zeroes, component bounds, overlong values, prerelease/build rejection, and no trailing newline. |
| CLI input semantics | Complete tuple, partial/repeated values, and `--pattern` rejection with any identity input. |
| JSON-RPC input semantics | `bicep/compile` `.bicep` entry point with and without the tuple, shared diagnostics and emission, and no propagation to dependencies. |
| Emission | Metadata beside `_generator`; preserve existing metadata; absent input leaves baseline output unchanged; deterministic output and normal template-hash behavior. |
| Publication and consumption | Source publication parity with build; raw JSON publication and consumer acceptance of missing `_module`; valid canonical metadata preservation; rejection of invalid, noncanonical, unknown-type, and unknown-field metadata at registry and inline nested-template boundaries; multiple uses, transitive nested templates, and no identity propagation from parents. |
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
      "version": "0.1.2"
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
              "version": "0.1.2"
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

1. **Constrained identity versus flexibility.** Seven hex characters bound the metadata payload and reduce accidental inclusion of identifiable or sensitive free-form text in ARM or deployment telemetry, but cannot represent every publisher's identifier. A full module name would be simpler to interpret, but its flexibility is intentionally rejected; report consumers use an external `(type, id)` mapping instead. Numeric-only release versions similarly do not represent full SemVer. Explicit schema extensions are preferable to an arbitrary string escape hatch.
1. **Explicit values versus source coupling.** Requiring the complete tuple keeps release identity independent of repository file conventions and metadata schemas, but publishing automation must supply all three values for each module.
1. **Metadata is not provenance.** A reserved name prevents ordinary Bicep declarations from writing the field, but any caller can supply valid CLI inputs or edit ARM JSON. Format validation is neither publisher authentication nor proof that a field contains no sensitive information. Any future collector needs its own trust and validation boundary.
1. **Artifact and compatibility costs.** Each module occurrence adds a small object and changes the compiled artifact. Existing published versions need republishing to gain this field; a consumer build cannot reliably reconstruct it. Present malformed or future-schema `_module` objects fail publication or consumption, so consumers must upgrade before accepting future types or fields.

## Alternatives

1. **An integer identifier.** A bounded non-negative integer could further narrow representation and avoid hexadecimal case normalization. It remains a viable alternative, but this draft proposes seven hexadecimal characters. Changing to an integer would require a decision on its JSON type, bounds, and assignment/migration policy.
1. **Seven ASCII alphanumeric characters.** A seven-character ASCII `[0-9A-Za-z]` identifier would retain the bounded length while increasing the assignment space to `36^7` (78,364,164,096) if case-insensitive, or `62^7` (3,521,614,606,208) if case-sensitive. It also admits more readable names and account-like identifiers, so it has a privacy tradeoff that requires review rather than equivalent-safety assumptions. A chosen alphabet must explicitly define case sensitivity and canonicalization, exclude Unicode `\w` characters, and retain publisher-side uniqueness checks; it does not add a compiler catalog lookup.
1. **AVM-specific options or an implicit type.** Shorter for the initial adopter but couples a general compiler interface to one module system. Generic `--module-type`, `--module-id`, and `--module-version` inputs and an explicit allowlisted type keep the interface extensible.
1. **Ordinary Bicep metadata declarations.** No new input mechanism would be needed, but release automation would have to modify source and the reserved schema would not be enforced as a compiler-managed contract. Descriptive user metadata remains available for other purposes.
1. **Put module identification inside `_generator`.** That conflates module identity with compiler identity. A sibling `_module` object keeps their lifecycles and meanings separate.
1. **Derive identity from a registry reference.** Registry hosts, repository paths, aliases, and tags are not a constrained opaque identity and can change through mirroring or republishing. This also fails to cover locally compiled modules.
1. **Accept arbitrary metadata, full module names, or full SemVer strings.** These would be easier to interpret or more flexible, but introduce free-form text without addressing the risk of accidentally carrying identifiable data in deployment metadata. This proposal starts with an opaque, bounded identifier, a closed object, and numeric release core.

## Rollout plan

1. Implement the settled CLI and JSON-RPC input contracts, shared validator, and emission path, initially behind an experimental gate. Cover standalone Bicep build, source-based publication, raw JSON publication validation, `bicep/compile` JSON-RPC, consumer validation, and valid metadata preservation together.
1. Pilot publisher and consumer workflows, including AVM modules, without claiming that template metadata is being collected or surfaced.
1. Stabilize the standalone Bicep input/output contract and document its minimum version. Do not remove any existing AVM telemetry mechanism merely because this field is available.
1. Deliver Azure CLI and Azure PowerShell named-parameter support as follow-up parity work. It does not block standalone Bicep adoption or the initial GA of this feature.

## Unresolved questions

1. What are the required opt-out semantics before any future telemetry collection is enabled? Compile-time identity is static and cannot be conditioned on a deployment parameter such as AVM's existing `enableTelemetry`.
1. What preservation behavior is required across older Bicep versions, decompilation/recompilation, template specs, and alternative module artifact formats? The initial inline-template contract must not imply these paths are already covered.

## Out of scope

- Telemetry collection, transmission, storage, querying, reporting, or its eventual presentation.
- Changes to existing AVM deployment telemetry, opt-out behavior, or a migration away from that mechanism.
- Publisher authentication, signed provenance, online catalog lookups, and enforcement of module identifier ownership.
- Support for module types other than `avm` in the initial release.
- Changes to ARM resource tags, deployment resources used for existing telemetry, or resource provider behavior.
- A generic mechanism for copying arbitrary repository metadata into ARM templates.

## Input mechanisms considered

These are alternatives considered during design; they are not part of the normative input contract.

1. **CLI flags (chosen for CLI invocation).** Named flags are explicit, discoverable, and scoped to one invocation entry point. The equivalent `bicep/compile` JSON-RPC tuple is required for compiler callers; it is not a competing discovery mechanism.
1. **Environment variables.** They could flow through an inherited process environment, but wrapper propagation is unverified and ambient state risks identity leakage between builds.
1. **`bicepconfig.json`.** Shared compiler configuration is available, but parent-directory discovery and inheritance could apply identity to helper modules; release version changes would also require mutating a configuration file.
1. **A dedicated metadata file.** It is unnecessary for the current single-module publishing workflow and adds file-selection, precedence, and dependency-tracking complexity. It can be proposed separately if a concrete batch-publishing scenario requires it.
