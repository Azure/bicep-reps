---
REP Number: "0000"
Author: levimatheri (Levi Muriuki)
Start Date: 2026-08-23
Feature Status: Private Preview
Bicep Issue Number(s): "[#5890](https://github.com/Azure/bicep/issues/5890), [#12919](https://github.com/Azure/bicep/issues/12919), [#8586](https://github.com/Azure/bicep/issues/8586), [#6970](https://github.com/Azure/bicep/issues/6970), [#4293](https://github.com/Azure/bicep/issues/4293)"
---

# OCI Template Links for Bicep Deployments

## Summary

This REP proposes a new way to submit large Bicep deployments to Azure Resource Manager (ARM). Instead of embedding the entire compiled template graph in the deployment request, the Bicep CLI packages the entry-point template and its nested deployment templates as a single OCI (Open Container Initiative) artifact and publishes it to a container registry. The user then submits an *OCI reference* to ARM, which resolves, validates, and processes the deployment from the artifact. This avoids the current 4 MB inline template limit for large module compositions while preserving the familiar Bicep authoring and deployment workflow.

## Terms and definitions

- **OCI (Open Container Initiative)**: An open standard, originally created for container images, for packaging and distributing content-addressed artifacts. Container registries (such as Azure Container Registry) implement the OCI distribution spec.
- **AVM (Azure Verified Modules)**: A Microsoft initiative that provides reusable infrastructure-as-code modules following defined quality and security standards. Bicep AVM modules are distributed through the public Bicep registry as OCI artifacts.
- **Artifact**: An OCI package, identified by a repository and tag (or digest), that bundles one or more files. Bicep already publishes modules as OCI artifacts (`bicep publish`); this proposal adds a layered format for deployable template bundles.
- **Manifest**: A small JSON document that describes an artifact's `config` blob and `layers`. Each is identified by a digest, media type, and size.
- **Image index**: An OCI document that groups multiple manifests under a single tag, allowing one artifact reference to expose different manifests to different consumers (for example, one per CPU architecture in a multi-platform container image). The primary design does not use an image index: each tag resolves directly to one manifest. [Compatibility with older Bicep CLI versions](#compatibility-with-older-bicep-cli-versions) discusses a dual-manifest alternative that would use one.
- **Layer**: An individual file stored inside an artifact (for example, a compiled ARM JSON template), addressed by its digest.
- **Digest**: A content hash (for example `sha256:44136fa3...`) that uniquely and immutably identifies a blob's content. Any layer can be referenced directly by digest instead of by tag.
- **Entry point**: The root ARM JSON template within an artifact — the template the Deployments service begins evaluating when the artifact is deployed.
- **Deployment closure**: The entry point and all nested deployment templates reachable from it. A layered artifact contains this complete set without external template dependencies.

## Motivation

ARM limits the template payload submitted with a deployment to 4 MB. Bicep compositions with many nested modules can exceed this limit even when no individual module is large. The current alternatives are:

- Upload the compiled templates to blob storage and reference them through `templateLink.uri` (linked templates). The caller must stage the complete deployment closure and maintain either individual links or relative paths from a common base URI. Private storage also requires SAS management.
  - **SAS tokens push users toward storage access keys.** User delegation SAS, backed by revocable RBAC grants, is safer but more cumbersome to configure than an access-key-based SAS. This friction can lead customers to enable broader, harder-to-revoke storage account access keys simply to generate SAS tokens more easily.
  - **Loosely linked templates make migrations fragile.** Moving a deployment to another cloud or environment means finding and re-uploading every blob in its closure; missing one breaks the deployment. Static tooling cannot always discover that closure because links can depend on runtime evaluation and short-circuiting.
  - **Loosely linked templates create reliability and substitution risks.** The root template owner may not control every linked blob. For example, public quickstarts that reference `raw.githubusercontent.com` regularly break when content moves or disappears. More seriously, linked content can change accidentally or maliciously while nested deployments continue to run with the root deployment's identity and permissions.
- Publish a Template Spec.
  - **Template Specs are capped at 2 MB.** This is tighter than ARM's own 4 MB inline limit, so a Template Spec can't solve the problem for large module compositions.
  - **Template Spec versions are mutable and scoped to a subscription and region.** This makes it harder to move a deployment safely across tenants, regions, or clouds.


Bicep already uses OCI artifacts to publish and consume reusable modules. This REP extends that mechanism to whole deployments: the CLI compiles a Bicep entry point and its nested modules, bundles the resulting templates into one content-addressed artifact, and pushes it to a registry. The deployment request carries only a small artifact reference, and the Deployments service fetches the templates from the registry. This approach:

- Removes the practical size ceiling for Bicep module compositions.
- Aligns with the existing Bicep module ecosystem by reusing the same OCI registry infrastructure, tooling, and customer mental model.
- Packages the complete deployment closure as one artifact, making it safer and easier to copy between environments.
- Provides human-readable versioning through tags and content integrity through immutable digests, guaranteeing the exact bytes ARM evaluates.
- Uses registry-native governance, replication, retention, and access controls rather than introducing another storage system. For ACR, private access uses `Container Registry Repository Reader` with a repository-scoped condition on ABAC-enabled registries, or registry-scoped `AcrPull` on non-ABAC registries, without requiring SAS tokens or storage account access keys.

## Detailed design

### End-to-end flow

#### Publishing

1. The user authors a normal Bicep file with local modules, registry modules, or both. No Bicep language changes are required.
1. The user invokes a Bicep command to compile and publish their files as an OCI artifact.
1. Bicep compiles the entry point and all required nested deployment templates, then packages them as layers in one OCI artifact.
1. Bicep publishes the artifact to a registry such as Azure Container Registry.
1. The user then invokes tooling to submit a deployment request with the OCI reference as the `templateLink`.
1. The Deployments service downloads the OCI artifact, validates its manifest and layers, and deploys it.

#### Consuming

##### Authoring experience

Referencing an OCI module in a Bicep file:

1. The user references a multi-layered registry module using the existing `br:` syntax.
1. Bicep restores and validates the multi-layer OCI artifact, then surfaces its parameters, outputs, and exports as usual.
1. If restoration or validation fails, Bicep reports an actionable diagnostic.
1. Navigation flows, e.g. inspecting backing source file for a registry module in VSCode should work with the new format too.
1. The user should then be able to deploy using whichever format they'd like (OCI or not), whether or not they are consuming multi-layer Bicep registry modules.

##### Deployment experience

Deploying an OCI module:

1. The user deploys an OCI module by supplying an OCI reference to a tool e.g. AzCLI.
1. The tool invokes the Deployments service by passing along the OCI reference.
1. The Deployments service resolves and validates the OCI artifact, then deploys it. 

In the future, this would allow such experiences like deploying an AVM module directly from the Azure Portal.


### CLI surface changes

```sh
# Compile the entry point and nested modules as separate OCI layers rather than
# inlining them into one file, and publish the result as a deployable artifact.
bicep publish main.bicep --target br:<registry>/<repo>:<tag> --artifact-format layered

# Deploy an already-published artifact.
bicep deploy --resource-group <rg> --template-oci-reference <registry>/<repo>:<tag>
az deployment group create --resource-group <rg> --template-oci-reference br:<registry>/<repo>:<tag>
New-AzResourceGroupDeployment -ResourceGroupName <rg> -TemplateOciReference br:<registry>/<repo>:<tag>
```

`bicep deploy` and `az deployment group create`/`New-AzResourceGroupDeployment` gain a `--template-oci-reference` parameter (mutually exclusive with `--template-file`/`--template-uri`/`--template-spec`). Publishing and deploying remain two distinct, explicit steps — neither command publishes on the caller's behalf. See [Artifact format](#artifact-format) for what `--artifact-format layered` actually produces.

### Artifact format

`bicep publish --artifact-format layered` publishes one manifest whose `config.mediaType` is `application/vnd.ms.bicep.module.config.v2+json`. It uses the same `artifactType` as the existing single-file format because both are Bicep module artifacts with different internal layouts:

```
artifactType: application/vnd.ms.bicep.module.artifact

config.mediaType:
  application/vnd.ms.bicep.module.config.v1+json  # existing single-file format
  application/vnd.ms.bicep.module.config.v2+json  # layered entry point + nested templates
```

Suppose `main.bicep` deploys one nested module, `modules/network.bicep`. Running `bicep publish main.bicep --target br:myregistry.azurecr.io/bicep/deployments/my-app:v1.2.0 --artifact-format layered` produces the following artifact. Because each tag has one manifest, the tag resolves directly to it without an image index.

**Manifest** (what `myregistry.azurecr.io/bicep/deployments/my-app:v1.2.0` resolves to):

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "artifactType": "application/vnd.ms.bicep.module.artifact",
  "config": {
    "mediaType": "application/vnd.ms.bicep.module.config.v2+json",
    "digest": "sha256:1f2a3bfa9c1e4d5b6a7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a",
    "size": 58
  },
  "layers": [
    {
      "mediaType": "application/vnd.ms.bicep.module.layer.v1+json",
      "digest": "sha256:aaa111e6b6e7cb3a5c7e6d4b3a2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c",
      "size": 842,
      "annotations": { "org.opencontainers.image.title": "main.json" }
    },
    {
      "mediaType": "application/vnd.ms.bicep.module.layer.v1+json",
      "digest": "sha256:bbb222e6b6e7cb3a5c7e6d4b3a2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c",
      "size": 391,
      "annotations": { "org.opencontainers.image.title": "modules/network.json" }
    }
  ]
}
```

**Config blob** (fetched separately, by `config.digest`):

```json
{
  "entryPointDigest": "sha256:aaa111e6b6e7cb3a5c7e6d4b3a2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c"
}
```

**`main.json` layer** (the entry point, `sha256:aaa111...`) — its nested deployment references the second layer purely by digest:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.Resources/deployments",
      "apiVersion": "2022-09-01",
      "name": "networkModule",
      "properties": {
        "mode": "Incremental",
        "templateLink": {
          "digest": "sha256:bbb222e6b6e7cb3a5c7e6d4b3a2c1d0e9f8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c"
        },
        "parameters": {}
      }
    }
  ]
}
```

**`modules/network.json` layer** (`sha256:bbb222...`) — a normal compiled ARM template, with no special structure of its own:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.Network/virtualNetworks",
      "apiVersion": "2023-09-01",
      "name": "vnet1",
      "location": "eastus",
      "properties": { "addressSpace": { "addressPrefixes": ["10.0.0.0/16"] } }
    }
  ]
}
```

### Authentication and registry support

**CI/CD considerations:**

- Publishing becomes a build step that requires `Container Registry Repository Writer` on [ABAC-enabled registries](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-rbac-abac-repository-permissions?tabs=azure-portal), or `AcrPush` on non-ABAC registries, in the pipeline's service connection, in addition to its existing deployment permissions.

**Authentication expectations:**

- Publishing uses the existing `bicep publish` authentication model for module registries, such as the caller's `az login` context.
- Consuming the artifact at deploy time does not require credentials for anonymously readable repositories. For private repositories, the Deployments service exchanges the caller's Entra ID token for an ACR token carrying only the caller's pull permissions; no separate secret or SAS management is needed.
- This token-exchange design is intended to support both user and service-principal callers. Service-principal delegation depends on ACR implementing stronger claim validation and is required before general availability.

**Registry support:**

- Initially, the Deployments service resolves `ociReference` only from Azure Container Registry. The OBO-based authentication described above is ACR-specific (`AcrPull`), but the artifact format is standard OCI and contains nothing ACR-specific.
- We will initially support ACR registries which are network-accessible to the Deployments service.
  - The preferred future path for network-restricted registries is for ACR to support [Network Security Perimeter (NSP)](https://learn.microsoft.com/en-us/azure/private-link/network-security-perimeter-concepts), allowing customers to configure inbound access from the `AzureResourceManager` service tag.
- Other OCI-compliant registry providers are out of scope for this proposal. See [Non-ACR OCI registries](#non-acr-oci-registries) for possible future authentication models.

### Module consumption of layered artifacts

A compatible Bicep CLI recognizes the layered format described in [Artifact format](#artifact-format) and treats it as a layered module rather than a single flattened template:

- **Restore** caches every layer on disk, not just the entry point.
- **Compile** builds the module's semantic model from the entry-point layer, whose parameters, outputs, and exports are already declared there — no flattening needed for type-checking.
- **Build** flattens the layers depending on the output format: `bicep build` (or `bicep publish` without `--artifact-format layered`) recursively inlines each nested `templateLink.digest` reference into a single template, reproducing today's flattened output; `bicep publish --artifact-format layered` instead copies the referenced artifact's transitive layers byte-for-byte into the new manifest, deduplicated by digest.

An artifact using the existing single-file format (`config.mediaType` of `application/vnd.ms.bicep.module.config.v1+json`) is unaffected — Bicep resolves it exactly as it does today.

#### Compatibility with older Bicep CLI versions

A Bicep CLI version released before layered-artifact support cannot restore or consume an artifact published with `--artifact-format layered`. Four options are under consideration:

| Option | Old clients can consume the artifact | Extra registry storage | Additional Bicep engineering beyond the core proposal |
| --- | --- | --- | --- |
| [1. Layered manifest only](#option-1-publish-only-the-layered-manifest) | No — fails with an "upgrade required" diagnostic | No | No |
| [2. OCI image index](#option-2-oci-image-index) | No — image-index resolution doesn't exist in Bicep yet | Yes | Yes — image-index resolution and manifest selection |
| [3. Single compatible manifest](#option-3-single-compatible-manifest) | Yes — reads the legacy layer, ignores the rest | Yes | No |
| [4. Separate tag](#option-4-publish-the-layered-format-under-a-separate-tag) | Yes — existing tag is untouched | Yes (new tag) | No |

##### Option 1: Publish only the layered manifest

`--artifact-format layered` publishes the new manifest on the module's normal tag, with no bridge to the existing format.

- Pros:
  - Simplest to implement, no additional Bicep engineering beyond the core proposal.
  - No duplicated storage — only the new artifact format is stored.
- Cons:
  - Older Bicep CLI versions fail outright when referencing the artifact, surfacing an "upgrade required" diagnostic, until they upgrade.

Because this introduces a compatibility break for consumers of layered artifacts, it would be best to ship it before Bicep 1.0, while the artifact-version contract can still evolve, rather than introduce this kind of break after 1.0 ships under a message of stability and maturity.

##### Option 2: OCI image index

Publish an [OCI image index](https://oras.land/docs/concepts/artifact/#harnessing-image-indexes) referencing both the existing single-file manifest and the new layered manifest under one tag.

- Pros:
  - Once index support and a Bicep-specific manifest-selection algorithm ship, one tag can expose both formats to index-aware clients.
  - Matches how the [OCI spec distinguishes a manifest from an index](https://github.com/opencontainers/image-spec/blob/main/manifest.md): each manifest still describes one specific representation, and the index is the sanctioned mechanism for exposing variants from one reference.
- Cons:
  - Requires implementing image-index resolution and manifest selection, **which doesn't exist in Bicep today**, therefore this does not help clients released before that support ships.
  - Uses additional registry storage because each artifact retains both the flattened legacy manifest and the layered representation manifest.

This also changes the [server-side resolution logic](#server-side-changes): the Deployments engine would need to resolve an image index and select the layered manifest from it, rather than assuming `ociReference` resolves directly to a manifest.

##### Option 3: Single compatible manifest

Keep the legacy template layer and add the new layered representation as additional layers. Old Bicep ignores the unfamiliar layers; new Bicep prefers them.

- Pros:
  - One tag and one manifest work for both old and new clients without a new manifest-selection mechanism.
  - Existing clients continue to consume the legacy template layer, while new clients can preserve the layered representation.
- Cons:
  - Repurposes manifest layers as an ad hoc variant-selection mechanism rather than the OCI image index that exists for that purpose; per the [OCI spec](https://github.com/opencontainers/image-spec/blob/main/manifest.md#image-manifest), each manifest is meant to describe one specific representation, not two representations disambiguated by which layers a client happens to recognize.
  - Uses additional registry storage because each artifact retains both the flattened legacy template and the layered representation.

This also changes the [server-side resolution logic](#server-side-changes): the manifest would need to keep the legacy `config.mediaType` of v1 for the new layered representation too, with an added `entryPointDigest` field that the server reads to find the entry-point layer. This works because existing Bicep versions already ignore the contents of the config blob.

##### Option 4: Publish the layered format under a separate tag

The customer publishes the layered format under a second, separate tag (e.g. `v1.2.0-layered`), in addition to the existing single-file tag they already publish. This is a manual step on the customer's side — Bicep does not publish both tags from a single `bicep publish` invocation.

- Pros:
  - Works today with no new Bicep engineering; existing clients and tags are unaffected.
- Cons:
  - Requires the customer to run and remember an extra publish step per version (manually or via their own tooling/automation).
  - Consumers must know which tag to reference for which experience.

### Server side changes

When the Deployments engine encounters a `templateLink.ociReference`:

- If a `digest` is also supplied, the engine downloads that layer directly and parses it as the template, without resolving the manifest.
- If no `digest` is supplied, the engine fetches the manifest for the given `ociReference` and requires it to have:
  - A `config.mediaType` value of `application/vnd.ms.bicep.module.config.v2+json`.
  - A non-empty `config` containing an `entryPointDigest`. The engine downloads that layer and parses it as the root template.
- The engine validates every layer download against the manifest-declared `size` and `digest`. It aborts downloads that exceed the declared size and rejects completed downloads whose size or content hash does not match. Each layer must be smaller than 4 MB.

#### Nested deployment validation

To prevent external or mutable dependencies:

- A nested deployment inside an artifact-resolved template should reference another layer in the same artifact using **only** a `digest` key in its `templateLink`.
- A nested deployment may alternatively use an inline `template`.
- A nested deployment's `templateLink` must **not** specify `uri`, `id`, `relativePath`, `queryString`, or `ociReference`. Only `digest` is allowed, so an artifact cannot reach outside itself.
- The engine tracks the originating `ociReference` across nested inline templates, so these restrictions apply recursively.
- Allowing nested `ociReference` links (e.g., to a different repository) is a possible future extension, not part of this proposal.

### Microsoft.Resources/deployments API changes

New `ociReference` and `digest` properties are added to the `templateLink` object on a deployment resource, **gated by a server-side feature flag**. At the root, `digest` is valid only when paired with `ociReference`; within a template already resolved from an OCI artifact, a digest-only `templateLink` identifies a layer in that same artifact.

```json
// ✅ ociReference alone — resolves the entry point via the artifact's manifest
{ "templateLink": { "ociReference": "myregistry.azurecr.io/bicep/deployments/my-app:v1.2.0" } }

// ✅ ociReference + digest — pins to one specific layer, skipping manifest resolution
{ "templateLink": { "ociReference": "myregistry.azurecr.io/bicep/deployments/my-app:v1.2.0", "digest": "sha256:bbb222..." } }

// ❌ ociReference paired with any other template source — mutually exclusive
{ "templateLink": { "ociReference": "myregistry.azurecr.io/bicep/deployments/my-app:v1.2.0", "uri": "https://..." } }
```

When `ociReference` is present, `uri`, `id`, `relativePath`, `queryString`, and `contentVersion` are disallowed. The optional `digest` is the only additional template-source property permitted. `contentVersion` is not used because OCI tags and digests provide the artifact's versioning and content-selection mechanisms.

Validation failures, including unsupported key combinations, oversized or malformed layers, hash mismatches, and invalid manifest media types, surface as standard deployment errors through the existing CLI, portal, and PowerShell paths.

## Drawbacks

- Requires the user to have (or create) a container registry as a prerequisite for large deployments, which is a new dependency for anyone who previously only relied on inline templates.
- Implementing OCI authentication for Deployments-to-ACR calls requires additional engineering effort, though a viable approach (exchanging the caller's Entra ID token for a pull-scoped ACR token) is already identified.
- Layered artifacts are incompatible with Bicep CLI versions that predate layered-manifest support; consumers must upgrade before referencing a layered artifact, unless we adopt one of the mitigations in [Compatibility with older Bicep CLI versions](#compatibility-with-older-bicep-cli-versions).

## Alternatives

1. **Raise the ARM request-body limit.** The 4 MB limit is a longstanding dependency of ARM PUT request processing and is unlikely to change. Raising it would also only move the boundary: sufficiently large deployments would eventually require another increase, without gaining self-containment or digest-based integrity.
1. **Store a zip or another archive in blob storage.** This would bundle the templates but retain blob storage's signed-URL and credential-management concerns. ARM would also need to download the full archive for each nested deployment unless it added caching or random-access semantics. OCI already provides content addressing, selective layer downloads, and registry distribution.
1. **Have the Deployments service run its own registry.** The Deployments service could operate a shared, Microsoft-managed registry that pools artifacts across all customer deployments. This raises new concerns: Bicep would need to publish directly to that registry (with its own access-token security questions), unreferenced artifacts would need active cleanup to prevent unbounded storage growth, and the Deployments service would take on a new operational dependency on a registry it must build and run itself. This is more complex than relying on registries customers already use, without the customer-side ownership and governance benefits of a self-managed registry.
1. **Model artifacts as ARM resources.** Introduce a new control-plane API that represents a Bicep artifact as an ARM resource, with a child resource type for each chunk, and have the Deployments service reassemble the chunks at deploy time. This risks reinventing capabilities OCI registries already provide, such as immutability and content-addressing, and raises the same abuse and cleanup concerns as a service-run registry, without the benefit of reusing the existing Bicep module ecosystem.

## Rollout plan

This feature ships behind two independent gates, both of which must be enabled for an end-to-end deployment to succeed:

- **Bicep CLI**: `--artifact-format layered` and `--template-oci-reference` are explicit opt-in flags, following the precedent of other experimental CLI functionality (e.g. `publish-extension`). Without passing `--artifact-format layered`, `bicep publish` continues to publish only the existing single-file format.
- **Deployments service**: the `ociReference` and `digest` properties on `templateLink` are gated behind a server-side feature flag on the `Microsoft.Resources/deployments` API; requests using `ociReference` are rejected unless the flag is enabled for the calling subscription.

The feature flag is initially enabled only for a small set of customers, to gather feedback and stabilize the feature before wider exposure.

## Open questions

1. Is requiring a container registry as a prerequisite for large deployments acceptable for most customers running into the 4 MB limit?
1. Older Bicep versions compatibility: which of the four options in [Compatibility with older Bicep CLI versions](#compatibility-with-older-bicep-cli-versions) should we go with? Any other options?

## Out of scope

### Non-ACR OCI registries

Initially, publishing, authentication, and Deployments service resolution target Azure Container Registry (ACR). Alternative registries are not part of this proposal. The Entra token-exchange flow does not generalize to registries that do not trust Entra, but possible future authentication models include:

- **Anonymous pull for public repositories.** This is the simplest increment but supports only content the customer is willing to make public.
- **Key Vault-backed registry credentials.** The caller could store a pull-scoped registry token in Key Vault and reference it from `templateLink`:

    ```json
    {
      "templateLink": {
        "ociReference": "ghcr.io/contoso/my-app:v1.2.0",
        "registryCredential": {
          "username": "contoso-deploy-bot",
          "passwordSecret": {
            "keyVault": { "id": "/subscriptions/.../vaults/contoso-kv" },
            "secretName": "ghcr-pull-token"
          }
        }
      }
    }
    ```

  `Microsoft.Resources/deployments` already resolves Key Vault references for secure parameters at deployment time, gated on the vault's `enabledForTemplateDeployment` setting and the caller's `Microsoft.KeyVault/vaults/deploy/action` permission. However, the service would have to resolve this credential before fetching the template rather than during parameter evaluation.
- **OIDC federation.** Registries that can federate with Entra would require no secret at rest. This is the cleanest end state, but few registries support it today.

### Helper functions for retrieving arbitrary application assets from an artifact's layers

Examples:

- **`downloadLayer`** — Accepts a digest in the format `<digestType>:<digestValue>` and returns the layer as a string.
- **`downloadLayerAsBase64`** — Returns the same layer content encoded as Base64.
- **`uriForLayer`** — Accepts a digest in the same format and returns a signed URL for the layer.

Because these functions would need to interact asynchronously with the registry, a dedicated top-level property that disallows expressions (e.g. a `variablesFromParentOciArtifact` block naming layers by digest) may be preferable to functions — it would let downloads happen before preprocessing, guarantee each layer loads only once, and make size/count limits easy to enforce.