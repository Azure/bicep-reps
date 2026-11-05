---
REP Number: "0031"
Author: jaredfholgate (Jared Holgate)
Start Date: 2026-11-05
Feature Status: "Not implemented (draft proposal)"
---

# Module reference metadata

## Summary

Introduce an optional module reference input that Bicep emits as a reserved `metadata._module` object in compiled ARM JSON:

```json
{
  "_module": {
    "reference": "br:exampleregistry.azurecr.io/bicep/modules/container-app:0.1.2"
  }
}
```

The CLI input is `--module-ref`; the equivalent `bicep/compile` JSON-RPC input is an optional `module` object with one `reference` property. A normal build with no input emits no `_module` object and remains unchanged. A source-based `bicep publish` defaults the compiler input to its resolved `--target` value when `--module-ref` is omitted.

Existing compiled JSON without `_module` remains unchanged. When `_module` is present, publication and consumption validate its closed canonical schema before preserving it in the resulting template.

This is an initial design draft. Proposed input names and implementation details below are not existing Bicep capabilities.

## Terms and definitions

| Term | Meaning |
| --- | --- |
| Module reference | A fully qualified Bicep OCI artifact reference, including the `br:` scheme and either a tag or digest. It is not an ARM resource ID. |
| Publisher build | Compilation of a module before its compiled template is distributed or published to a registry. |
| Consumer build | Compilation of a template that references an already-published module. |
| Reserved metadata | A compiler-managed metadata property with a leading underscore, separate from ordinary Bicep metadata declarations. |

## Motivation

A registry module's reference is used to restore the module during compilation. In the inline deployment model, its template is embedded in the consumer's compiled ARM template; the original registry reference is not otherwise a stable identity field in that output.

Embedding a validated module reference in compiled ARM JSON carries it through build, artifact, and deployment workflows to ARM, where it is available to read during deployment. This avoids relying on source-only metadata, CLI state, or an external sidecar that may be absent from the deployment payload. This REP defines availability only: it does not define an ARM reader, collection, interpretation, transmission, storage, or surfacing mechanism.

Embedding static module identity requires no feature-specific deployment opt-out. Any future ARM collection or handling follows applicable ARM platform behavior and policies, not a policy defined here. A deployment-command flag can be considered separately if it is later needed.

The intended outcome is a bounded, machine-readable metadata contract, not an instruction to transmit telemetry or a claim that the module's publisher has been authenticated.

## Detailed design

### Client side changes

#### Existing implementation and required change points

The current [build command](https://github.com/Azure/bicep/blob/main/src/Bicep.Cli/Commands/BuildCommand.cs) resolves a file pattern and creates one compilation for each selected entry point. The proposed `--module-ref` input therefore flows through the command arguments and into one compilation. `--pattern` is rejected when `--module-ref` is present, even if it happens to select one file.

The current [template writer](https://github.com/Azure/bicep/blob/main/src/Bicep.Core/Emit/TemplateWriter.cs) emits `metadata._generator` before authored metadata. The [metadata symbol validator](https://github.com/Azure/bicep/blob/main/src/Bicep.Core/Semantics/MetadataSymbol.cs) already reserves every metadata identifier beginning with `_`. `_module` uses that reserved namespace and requires a compiler input and emitter path; it cannot be produced as an ordinary Bicep metadata declaration.

Bicep already resolves publish targets through its [artifact reference resolver](https://github.com/Azure/bicep/blob/main/src/Bicep.Core/Registry/ArtifactReferenceResolver.cs). That resolver expands configured `br/<alias>:` aliases, then parses the fully qualified OCI body. The emitted identity is the resolver's fully qualified `br:` reference, never an alias spelling.

The OCI parser accepts a registry, lower-case repository path, and exactly one tag or digest. Tagged references take the form `br:registry/repository:tag`; digest references take the form `br:registry/repository@sha256:<64 lowercase hexadecimal characters>`. Validation reuses the [existing OCI address parser](https://github.com/Azure/bicep/blob/main/src/Bicep.Core/Registry/Oci/OciArtifactAddressComponents.cs), rather than a new regular expression or generic URI validator.

Restored ARM JSON modules use an [ARM template writer](https://github.com/Azure/bicep/blob/main/src/Bicep.Core/Emit/ArmTemplateWriter.cs) that writes the parsed template object unchanged. The proposal adds validation before that write: when `_module` is present in a compiled JSON artifact, validate it on raw JSON publication and at consumer restore boundaries, including embedded inline nested templates. A missing object is ignored. A valid object is retained without normalization or rewrite; an invalid object fails the operation.

The [CLI JSON-RPC protocol](https://github.com/Azure/bicep/blob/main/src/Bicep.Cli/Rpc/ICliJsonRpcProtocol.cs) currently exposes `bicep/compile` with a path and optional configuration path, plus `bicep/compileParams`; it has no JSON-RPC publish operation. The initial implementation extends `bicep/compile` for `.bicep` entry points with the optional structured `module.reference` input. It uses the shared validator and emitter, applies only to that entry point, and never propagates to referenced modules. `bicep/compileParams` produces deployment parameters rather than a module template and does not receive this input. This requires protocol and compiler coverage, not an editor UI change.

#### Wrapper integration

The current [Azure CLI Bicep commands](https://learn.microsoft.com/en-us/cli/azure/bicep#az-bicep-publish) expose `az bicep build` and `az bicep publish`, but do not expose the proposed reference input. Follow-up Azure CLI work adds named `--module-ref` options to both commands, forwards them to Bicep, updates help and tests, and enforces the minimum Bicep version with consistent unsupported-binary errors.

The current [Publish-AzBicepModule](https://learn.microsoft.com/en-us/powershell/module/az.resources/publish-azbicepmodule) cmdlet likewise does not expose a module reference parameter. Follow-up Azure PowerShell work adds `-ModuleRef`, forwards it to Bicep, and covers help, tests, and minimum-version behavior. No public Azure PowerShell build cmdlet is identified; callers building from PowerShell continue to use standalone `bicep build`.

Shared Bicep core owns resolution, validation, and emission; wrappers only provide argument plumbing, help, tests, and capability handling. This requires separate Azure/azure-cli and Azure/azure-powershell changes. Wrapper parity is not an initial rollout or GA prerequisite: MAR invokes standalone `bicep publish` directly, so day-one AVM adoption uses standalone Bicep build and publish. Existing wrapper commands continue to work without the new input; callers that need it use standalone Bicep until the wrapper updates ship.

#### Output contract

When an effective module reference is present and valid, Bicep emits exactly this object at the root of the template being compiled:

```json
{
  "_module": {
    "reference": "br:exampleregistry.azurecr.io/bicep/modules/container-app:0.1.2"
  }
}
```

The object is a property of the template's existing `metadata` object, alongside `_generator`, `name`, and `description`. It is not part of `_generator`, a resource's `tags`, or the deployment resource's own metadata.

Each template has at most one `_module` object describing that template's module. Dependencies retain their own objects; there is no flattened list on the parent. A normal build with no `--module-ref` emits no object, inferred default, or other new metadata. Source-based `publish` is the exception: because it already requires `--target`, an omitted `--module-ref` uses that resolved target as the effective reference for the source compilation.

The name is reserved for the compiler. Ordinary Bicep `metadata` declarations cannot author or override `_module`. This REP does not relax the existing reserved-name restriction.

#### Input option: explicit CLI value

Proposed build syntax:

```powershell
bicep build .\main.bicep `
  --module-ref br:exampleregistry.azurecr.io/bicep/modules/container-app:0.1.2 `
  --outfile .\main.json
```

`--module-ref` is optional and may occur once. Build without it preserves current behavior. A provided value is a literal Bicep OCI reference; it is not a Bicep expression, environment-variable reference, file content, or deployment parameter. It applies only to the selected entry point and never relabels local or registry dependencies.

The option accepts fully qualified `br:` references and configured `br/<alias>:` references accepted by the shared resolver. The compiler emits the resolved fully qualified `br:` form. Resolution validates syntax and configured aliases only; it does not fetch from, authenticate to, or otherwise contact a registry. Existing restore and publish authentication behavior is unchanged.

#### Input option: JSON-RPC compile request

The optional `module` object on a `bicep/compile` request contains exactly one `reference` property. It has the same resolution, validation, and canonicalization as the CLI input. An absent object emits no `_module` and preserves the request's existing behavior. It is valid only when the request path is a `.bicep` entry point; it is not inherited by helper or dependency modules.

#### Validation and canonicalization

The shared validator applies to CLI and JSON-RPC inputs. The existing OCI parser is the source of truth for accepted syntax, including tag and digest forms, registry/repository/tag component limits, and configured alias expansion. No separate arbitrary overall reference length is introduced.

The parser limits the registry and repository components to 255 characters each and a tag to 128 characters. The limits are measured by the existing parser as string characters, not a new byte cap; a tagged, fully qualified `br:` reference is therefore bounded by its validated components at 643 characters. Digest references use the parser's fixed `sha256:<64 lowercase hexadecimal characters>` format. Values are never silently truncated.

The object schema is closed: exactly one `reference` property is required, with no additional properties. The canonical emitted form is a fully qualified `br:` reference with no alias. Registry comparison is case-insensitive; repository, tag, and digest comparisons retain the existing parser's behavior. For a `_module` object already present in compiled JSON, `reference` must already be in this canonical fully qualified form. Existing JSON is never normalized or rewritten, so aliases or other noncanonical values are rejected rather than changed.

Diagnostics identify the input source or artifact boundary and violated rule without echoing the rejected reference or complete metadata object. An invalid input or present `_module` object produces a non-zero exit code and no successful template emission or publication; an existing output file must not be presented as the result of that failed build.

An OCI reference can contain organization- or publisher-identifying registry, repository, and tag text. Parser validation reduces malformed input but does not prove a reference is non-sensitive or establish publisher authenticity. Any future collector needs its own trust and validation boundary.

#### Compilation, publishing, and consumption

When `--module-ref` is supplied for build or source publish, Bicep resolves it and emits the effective reference through the normal template emitter before final template hashing.

The current [publish command](https://github.com/Azure/bicep/blob/main/src/Bicep.Cli/Commands/PublishCommand.cs) resolves `--target` before source compilation, publishes raw ARM JSON directly, and rejects `--with-source` for JSON input. Source-based `bicep publish` retains `--module-ref` and the existing `--with-source` option:

```powershell
# Proposed --module-ref; current --target and --with-source options.
bicep publish .\main.bicep --target br:exampleregistry.azurecr.io/bicep/modules/container-app:0.1.2 --module-ref br:exampleregistry.azurecr.io/bicep/modules/container-app:0.1.2 --with-source
```

If `--module-ref` is omitted for source publish, Bicep resolves `--target` through the same resolver and uses the resulting fully qualified `br:` reference during compilation. An explicit `--module-ref` may differ from `--target`; this REP does not require them to match. The effective reference is compilation metadata, not an authored source declaration. `--with-source` continues to archive Bicep source; this REP does not define an archive format.

The feature must not be implemented as a post-build text replacement. The current [template writer](https://github.com/Azure/bicep/blob/main/src/Bicep.Core/Emit/TemplateWriter.cs) computes `metadata._generator.templateHash` from the generated template before inserting the hash value itself. Emitting `_module` before that calculation gives each generated artifact a hash that matches its own content. Adding or changing `_module` therefore produces a different valid template hash; it does not leave a hash valid for prior content.

`bicep publish` also currently accepts a pre-compiled ARM JSON input. The following two-step alternative uses the proposed reference input only on `build`, then uses the current JSON publication capability without `--module-ref` or `--with-source`:

```powershell
# Proposed --module-ref; not supported by Bicep today.
bicep build .\main.bicep --module-ref br:exampleregistry.azurecr.io/bicep/modules/container-app:0.1.2 --outfile .\main.json

# Current pre-compiled JSON publication capability.
bicep publish .\main.json --target br:exampleregistry.azurecr.io/bicep/modules/container-app:0.1.2
```

Current Bicep rejects `--with-source` for a JSON input. Supplying `--module-ref` for a pre-compiled JSON publish is rejected because there is no compilation phase in which to inject it. Under this REP, the JSON publication path validates a present `_module` object but never injects, relabels, or rewrites it.

During a consumer build, the existing ARM JSON writer preserves each restored module's top-level metadata when it is embedded in `Microsoft.Resources/deployments.properties.template`. A consumer validates every present `_module` object in restored registry templates and embedded inline nested templates against the closed canonical schema. Valid values are retained unchanged; missing objects remain untouched; invalid, unknown-field, or noncanonical objects fail the build without being normalized or rewritten. Future schema additions require consumers to upgrade Bicep before they can consume artifacts that use them.

External templates with no `_module` object remain supported without inferred module identification, validation failure, or any other action. Reusing the same module more than once preserves its identity on each occurrence. A parent has no module identification object unless its own build input or source publish target supplies one.

There are no new deployment parameters, resources, resource tags, or runtime expressions.

#### Validation coverage required for implementation

| Area | Required cases |
| --- | --- |
| Reference resolution | Fully qualified tagged and digest references; configured aliases resolving to a fully qualified emitted value; invalid syntax; parser component limits; no registry call for validation. |
| CLI input semantics | Build with and without `--module-ref`; repeated option; `--pattern` rejection with the option; source publish explicit override and target default; pre-compiled JSON rejection of the option. |
| JSON-RPC input semantics | `bicep/compile` `.bicep` entry point with and without `module.reference`, shared diagnostics and emission, and no propagation to dependencies. |
| Emission | Metadata beside `_generator`; absent normal-build input leaves baseline output unchanged; source-publish target default; deterministic output and normal template-hash behavior. |
| Publication and consumption | Source publication with `--with-source` and explicit or target-default reference; raw JSON publication and consumer acceptance of missing `_module`; valid canonical metadata preservation; rejection of invalid, aliased, or noncanonical metadata at registry and inline nested-template boundaries; multiple uses, transitive nested templates, and no identity propagation from parents. |
| Compatibility and diagnostics | Supported ARM language versions, array- and object-shaped resource collections, older templates without the field, errors that never echo rejected values, and upgrade requirements for future schema additions. |

#### Initial compatibility scope

Initial delivery covers compiler build, source-based `bicep publish`, the `bicep/compile` JSON-RPC input, raw JSON publication validation, and validation/preservation of existing registry and inline compiled JSON templates. Decompile/recompile behavior, template specs, and alternative module artifact formats are deferred and are not initial delivery gates. Older Bicep binaries do not gain the new input; consumers must upgrade Bicep before accepting artifacts that use a later `_module` schema.

### Server side changes (if applicable)

No server-side change is required to carry descriptive metadata in the compiled template. This REP does not define a server-side interpretation of `_module`.

#### Telemetry surfacing

### Microsoft.Resources/deployments API changes (if applicable)

None proposed for carrying the object in existing template metadata.

### Examples

These are proposed output excerpts, not complete deployable templates. The reference is illustrative and not asserted to identify the Container App module.

#### Container App module compiled by its publisher

The existing [Container App compiled template](https://github.com/Azure/bicep-registry-modules/blob/main/avm/res/app/container-app/main.json) contains descriptive metadata. With a validated module reference, its top-level metadata would have this shape:

```json
{
  "metadata": {
    "_generator": {
      "name": "bicep",
      "version": "<compiler-version>",
      "templateHash": "<generated-template-hash>"
    },
    "_module": {
      "reference": "br:exampleregistry.azurecr.io/bicep/modules/container-app:0.1.2"
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
              "reference": "br:exampleregistry.azurecr.io/bicep/modules/container-app:0.1.2"
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

1. **Registry reference and privacy.** A reference carries a directly interpretable module location and version, but registry, repository, and tag text can identify organizations or contain publisher-chosen text. OCI syntax validation limits malformed input but is not a privacy, provenance, or safety guarantee.
1. **Target default versus explicit override.** Defaulting source publish to `--target` avoids repeating the reference in the common path. An explicit `--module-ref` supports a distinct identity and is deliberately not required to match the publication target.
1. **Artifact and compatibility costs.** Each module occurrence adds a small object and changes the compiled artifact. Present malformed or future-schema `_module` objects fail publication or consumption, so consumers must upgrade before accepting future schema additions.

## Alternatives

1. **Ordinary Bicep metadata declarations.** No new input mechanism would be needed, but release automation would have to modify source and the reserved schema would not be enforced as a compiler-managed contract. Descriptive user metadata remains available for other purposes.
1. **Put module metadata inside `_generator`.** That conflates module identity with compiler identity. A sibling `_module` object keeps their lifecycles and meanings separate.
1. **Require a matching publication target.** Requiring `--module-ref` to equal `--target` would prevent intentional overrides and adds a policy the initial design does not need.

## Rollout plan

1. Implement the settled CLI and JSON-RPC input contracts, shared OCI resolution and validation, and emission path, initially behind an experimental gate. Cover standalone Bicep build, source-based publication with target default and `--with-source`, raw JSON publication validation, `bicep/compile` JSON-RPC, consumer validation, and valid metadata preservation together.
1. Pilot publisher and consumer workflows, including AVM modules, without claiming that template metadata is being collected or surfaced.
1. Stabilize the standalone Bicep input/output contract and document its minimum version.
1. Coordinate AVM adoption and migration separately: static module reference metadata makes the legacy active telemetry mechanism and `enableTelemetry` parameter redundant, so they are intended to be retired rather than retained indefinitely. This compiler REP neither removes those AVM parameters or resources nor requires their immediate removal before future ARM availability.
1. Deliver Azure CLI and Azure PowerShell named-parameter support as follow-up parity work. It does not block standalone Bicep adoption or the initial GA of this feature.

## Unresolved questions

None for the initial scope; additional preservation paths are deferred.

## Out of scope

- Telemetry collection, transmission, storage, querying, reporting, or its eventual presentation.
- Implementing the separately coordinated AVM migration that retires legacy active telemetry and its `enableTelemetry` parameter.
- Publisher authentication, signed provenance, online catalog lookups, and enforcement of module reference ownership.
- Support for non-OCI module reference schemes in the initial release.
- Changes to ARM resource tags, deployment resources, or parameters used by legacy AVM active telemetry, or to resource provider behavior.
- A generic mechanism for copying arbitrary repository metadata into ARM templates.

## Input mechanisms considered

These are alternatives considered during design; they are not part of the normative input contract.

1. **CLI flags (chosen for CLI invocation).** `--module-ref` is explicit, discoverable, and scoped to one invocation entry point. The equivalent `bicep/compile` JSON-RPC input is required for compiler callers; it is not a competing discovery mechanism.
1. **Environment variables.** They could flow through an inherited process environment, but wrapper propagation is unverified and ambient state risks reference leakage between builds.
1. **`bicepconfig.json`.** Shared compiler configuration is available, but parent-directory discovery and inheritance could apply a module reference to helper modules or retain stale publication targets.
1. **A dedicated metadata file.** It is unnecessary for the current single-module publishing workflow and adds file-selection, precedence, and dependency-tracking complexity. It can be proposed separately if a concrete batch-publishing scenario requires it.
