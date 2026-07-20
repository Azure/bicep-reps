---
REP Number: "0022"
Author: shenglol (Shenglong Li)
Start Date: 2026-07-20
Feature Status: Private Preview
Bicep Issue Number(s): "[#19883](https://github.com/Azure/bicep/issues/19883)"
---

# Artifact aliases and redirects

## Summary

This proposal unifies the artifact-reference experience for modules and extensions. It also supports monorepo setups through local artifact resolution, allowing modules, extensions, and data files in the same repository to be consumed without brittle relative paths or publishing every change to a registry.

This proposal introduces:

- `artifacts.aliases`, which provides reusable aliases for artifact locations.
- `artifacts.redirects`, which redirects artifact references to relative local files for development and testing, with optional named capture substitution.
- `mar` and `avm`, predefined OCI artifact aliases for Microsoft Artifact Registry and Azure Verified Modules whose OCI locations can be overridden.
- `local`, a first-class artifact scheme for resolving artifacts from the local filesystem.
- `oci`, the canonical scheme for registry-backed artifacts.

The primary scenarios are a consistent artifact-reference experience across modules and extensions and local artifact development in monorepos. This proposal preserves existing `br` references and `moduleAliases` configuration without warnings or required migration.

## Terms and definitions

- **Artifact**: A Bicep module, extension package, Template Spec, local data file, or future package type that Bicep can reference or load.
- **Artifact reference**: The source text that identifies an artifact, either through a fully qualified scheme or an alias.
- **Artifact scheme**: A prefix such as `oci:`, `ts:`, or `local:` that identifies how Bicep resolves an artifact.
- **Artifact alias**: A logical name that maps to an artifact location and is referenced with the `::` separator.
- **Artifact redirect**: A development-time override that maps an artifact reference or pattern to one relative local file.
- **MAR**: Microsoft Artifact Registry, exposed through the predefined `mar` alias.
- **AVM**: Azure Verified Modules, exposed through the predefined `avm` alias.
- **OCI**: Open Container Initiative. In this proposal, `oci:` identifies registry-backed artifacts distributed using OCI protocols.
- **Template Spec**: An Azure Resource Manager template artifact referenced with the `ts:` scheme.

## Motivation

Bicep does not currently offer one consistent user experience for referencing artifacts across modules and extensions. This proposal makes the same artifact schemes and artifact-location aliases available to both artifact types. Named extensions continue to be configured in the existing `extensions` map because that map binds an extension declaration name to an artifact reference.

### Existing module alias experience

Today, a `moduleAliases.br` entry in `bicepconfig.json` can assign a short name to a Bicep Registry location. For example, the `avm` alias defines a registry and module path:

```json
{
  "moduleAliases": {
    "br": {
      "avm": {
        "registry": "mcr.microsoft.com",
        "modulePath": "bicep/avm"
      }
    }
  }
}
```

A module uses the alias through the `br/<alias>:<module>` syntax:

```bicep
module storage 'br/avm:res/storage/storage-account:0.32.1'
```

This resolves to:

```text
br:mcr.microsoft.com/bicep/avm/res/storage/storage-account:0.32.1
```

This proposal continues to support this configuration and reference syntax without warnings. It builds on the existing alias experience while making artifact references consistent for modules and extensions and adding local artifact support.

For monorepos, [Azure/bicep#19883](https://github.com/Azure/bicep/issues/19883) describes how relative references to shared artifacts become brittle when files move, while publishing every change to a registry inhibits the end-to-end development workflow that a monorepo is intended to provide. Local artifact resolution gives modules, extensions, and data files stable artifact-style references without requiring a publish step. This is the local file registry scenario from the issue; the `local` scheme in this proposal resolves files directly and does not operate an OCI registry.

The proposed model has the following goals.

### Unify module and extension artifact-reference UX

Modules and extensions can use the same fully qualified artifact schemes and `<alias>::<artifact>` location references. An extension can use an artifact reference directly in its declaration or be referenced by a name defined in the `extensions` configuration. Users do not need different source-reference syntax for each artifact type.

### Support multiple artifact sources

The design supports:

- OCI registries.
- Template Specs.
- Local filesystem artifacts.
- Future artifact types.

### Preserve fully qualified references

Users can always reference artifacts directly without configuring an alias.

```bicep
module storage 'oci:mcr.microsoft.com/bicep/avm/res/storage/storage-account:0.32.1'

module app 'ts:subscription/resourceGroup/templateSpec:v1'

module local 'local:modules/storage/main.bicep'
```

### Provide convenient aliases

Aliases avoid repeating long artifact locations.

```bicep
module publicStorage 'mar::avm/res/storage/storage-account:0.32.1'

module storage 'avm::res/storage/storage-account:0.32.1'
```

These references can replace a fully qualified location such as:

```bicep
module storage 'oci:mcr.microsoft.com/bicep/avm/res/storage/storage-account:0.32.1'
```

### Support local development without changing artifact identity

Developers can redirect a published artifact reference to an unpublished local module or extension. The source keeps its production artifact identity while a development configuration selects the local implementation.

### Maintain backward compatibility

Existing `br` references and `moduleAliases` configuration continue to work without modification or warnings under this proposal. Projects can adopt `oci` and `artifacts.aliases` incrementally without being required to migrate existing source or configuration.

### Non-goals

This proposal does not replace ordinary relative file references, make aliases an artifact allowlist, or make local artifacts immutable. It does not change configuration discovery or merging, registry authentication, OCI distribution, Template Spec APIs, or the ARM deployment contract.

## Detailed design

### Artifact reference syntax

Artifact references are either fully qualified references or alias references.

#### Fully qualified references

Fully qualified references use a scheme:

```text
<scheme>:<artifact>
```

Supported schemes are:

- `oci`, the canonical registry artifact scheme.
- `ts`, the Template Spec scheme.
- `local`, the local filesystem artifact scheme.
- `br`, the backward-compatible registry module scheme.

Examples:

```bicep
module storage 'oci:mcr.microsoft.com/bicep/avm/res/storage/storage-account:0.32.1'

module app 'ts:<subscriptionId>/<resourceGroup>/<templateSpec>:v1'

module storage 'local:modules/storage/main.bicep'
```

Fully qualified references do not require configuration. `br` remains supported, but `oci` is the recommended spelling for new registry-backed references.

#### Predefined MAR and AVM aliases

`mar` and `avm` are predefined aliases for artifacts hosted in Microsoft Artifact Registry. Like every artifact alias, they use `::` and are not qualified by a scheme.

```text
mar::<artifact>
avm::<artifact>
```

Examples:

```bicep
module storageFromMar 'mar::avm/res/storage/storage-account:0.32.1'

module storageFromAvm 'avm::res/storage/storage-account:0.32.1'
```

These resolve as follows:

```text
mar::avm/res/storage/storage-account:0.32.1
  -> oci:mcr.microsoft.com/bicep/avm/res/storage/storage-account:0.32.1

avm::res/storage/storage-account:0.32.1
  -> oci:mcr.microsoft.com/bicep/avm/res/storage/storage-account:0.32.1
```

By default, `mar` maps to registry `mcr.microsoft.com` with repository prefix `bicep`, while `avm` maps to the same registry with repository prefix `bicep/avm`.

Both aliases are OCI defaults whose locations can be overridden by entries with the same names in `artifacts.aliases`. An override must have `type: "oci"`; configuration validation fails if an override attempts to change either alias to `ts`, `local`, or a future alias type. This supports private OCI mirrors without allowing a source reference associated with MAR or AVM to change artifact source type. A predefined alias remains a convenience rather than a guarantee of artifact provenance because its registry and repository prefix can change through configuration.

The default mappings are independent of the configured Azure cloud profile. `mar` and `avm` resolve to the same `mcr.microsoft.com` locations in Azure public, government, China, custom, and disconnected cloud profiles. This avoids making one source reference resolve differently when only the Azure Resource Manager cloud changes. Environments that require a sovereign endpoint, private mirror, or disconnected registry can override either alias explicitly in `artifacts.aliases`.

#### Artifact alias references

Aliases use `::` to distinguish them from artifact schemes.

```text
<alias>::<artifact>
```

Examples:

```bicep
module storage 'avm::res/storage/storage-account:0.32.1'

module network 'specs::network/vnet:v2'
```

The alias is resolved through `artifacts.aliases`. For example, `avm::res/storage/storage-account:0.32.1` is an alias reference, while `oci:mcr.microsoft.com/bicep/avm/res/storage/storage-account:0.32.1` is fully qualified.

The new syntax does not support `<scheme>/<alias>:<artifact>`. Scheme-qualified alias forms such as `oci/mar:` are invalid. The slash-based `br/<alias>:` form remains supported only for backward compatibility.

Artifact alias names use the same grammar as existing module alias names so aliases can be migrated without renaming them. A name must contain one or more ASCII letters, digits, hyphens, or underscores and is matched case-sensitively. There is no additional alias-specific length limit beyond the limits applied to the configuration document. Characters that participate in artifact-reference syntax, including `/` and `:`, whitespace, and non-ASCII characters are not allowed. `mar` and `avm` follow the same name grammar, but their alias type is fixed to `oci`.

#### Canonical artifact identity

Alias expansion produces the same fully qualified reference that the corresponding artifact scheme accepts. The existing artifact handlers remain responsible for scheme-specific validation and normalization; this proposal does not introduce a second OCI or Template Spec grammar.

- An OCI identity uses the `oci:` scheme and the registry, repository, and version accepted by the existing registry artifact handler. A legacy `br:` reference is canonicalized by changing the scheme to `oci:` after applying the existing `br` parsing and normalization rules.
- A Template Spec identity uses `ts:<subscriptionId>/<resourceGroup>/<templateSpec>:<version>` after expanding either a legacy or new alias.
- A local identity is the normalized filesystem path described under **Local artifact scheme**.

Redirect matching uses the normalized identity returned by the artifact handler. It does not treat different tags as equivalent, resolve a tag to a digest, or perform a registry request merely to canonicalize a reference.

### `artifacts.aliases`

The top-level `artifacts` object groups artifact resolution configuration and leaves room for future artifact policies. Its `aliases` property maps logical names to artifact locations. Aliases are shortcuts, not an allowlist. Fully qualified references remain valid regardless of which aliases are configured.

```json
{
  "artifacts": {
    "aliases": {
      "company": {
        "type": "oci",
        "registry": "contoso.azurecr.io",
        "repositoryPrefix": "bicep/modules"
      },
      "specs": {
        "type": "ts",
        "subscriptionId": "...",
        "resourceGroup": "shared-specs-rg"
      }
    }
  }
}
```

Each alias has a `type` discriminator. The remaining properties are validated according to that type.

#### OCI aliases

An OCI alias defines a registry and an optional repository prefix.

```json
{
  "artifacts": {
    "aliases": {
      "company": {
        "type": "oci",
        "registry": "contoso.azurecr.io",
        "repositoryPrefix": "bicep/modules"
      }
    }
  }
}
```

The reference:

```bicep
module storage 'company::storage/storage-account:1.0.0'
```

resolves to:

```text
oci:contoso.azurecr.io/bicep/modules/storage/storage-account:1.0.0
```

An OCI alias can also set `repositoryPrefix` to the complete repository path:

```json
{
  "artifacts": {
    "aliases": {
      "storage": {
        "type": "oci",
        "registry": "mcr.microsoft.com",
        "repositoryPrefix": "bicep/avm/res/storage/storage-account"
      }
    }
  }
}
```

Because the alias supplies the full repository, the artifact portion of the reference contains only the tag delimiter and tag:

```bicep
module storage 'storage:::0.32.1'
```

This resolves to:

```text
oci:mcr.microsoft.com/bicep/avm/res/storage/storage-account:0.32.1
```

The three consecutive colons are not a separate operator. The first two form the `::` alias separator, and the third begins the OCI tag.
This empty artifact-path form is valid only when the OCI alias supplies the complete repository path and the remaining text is a valid version suffix for that repository. Other alias types require a non-empty artifact path.

#### Template Spec aliases

A Template Spec alias defines the Azure scope containing a set of Template Specs.

```json
{
  "artifacts": {
    "aliases": {
      "specs": {
        "type": "ts",
        "subscriptionId": "<subscriptionId>",
        "resourceGroup": "shared-specs-rg"
      }
    }
  }
}
```

The reference:

```bicep
module app 'specs::webapp:v1'
```

resolves to:

```text
ts:<subscriptionId>/shared-specs-rg/webapp:v1
```

#### Local aliases

A local alias establishes a stable artifact root relative to the configuration file.

##### Full monorepo example

Consider a team that keeps reusable modules organized by resource provider under `<root>/modules`, shared assets under `<root>/assets`, and deployable solutions under `<root>/solutions`:

```text
<root>/
├── bicepconfig.json
├── modules/
│   ├── compute/
│   │   └── virtual-machine.bicep
│   ├── network/
│   │   └── virtual-network.bicep
│   └── web/
│       └── web-app.bicep
├── assets/
│   ├── legal-notice.txt
│   ├── environments.json
│   ├── deployment.yaml
│   └── certificates/
│       └── service.pfx
└── solutions/
    ├── storefront/
    │   └── main.bicep
    └── internal-api/
        └── regions/
            └── westus/
                └── main.bicep
```

The root `bicepconfig.json` defines one local alias for the shared module library and one for shared assets:

```json
{
  "artifacts": {
    "aliases": {
      "modules": {
        "type": "local",
        "path": "./modules"
      },
      "assets": {
        "type": "local",
        "path": "./assets"
      }
    }
  }
}
```

`solutions/storefront/main.bicep` can consume modules from different resource-provider folders and load shared data through these aliases:

```bicep
module network 'modules::network/virtual-network.bicep' = {
  name: 'network'
}

module compute 'modules::compute/virtual-machine.bicep' = {
  name: 'compute'
}

module web 'modules::web/web-app.bicep' = {
  name: 'web'
}

var legalNotice = loadTextContent('assets::legal-notice.txt')
var environments = loadJsonContent('assets::environments.json')
var deployment = loadYamlContent('assets::deployment.yaml')
```

The more deeply nested `solutions/internal-api/regions/westus/main.bicep` uses the same references. It does not need paths such as `../../../../modules/network/virtual-network.bicep`, and moving the consuming solution to a different directory does not change its module or asset references.

All artifact portions include the complete filename rather than identifying a folder with a conventional entry point. Each alias root is resolved relative to the `bicepconfig.json` that declares it, so every consuming solution covered by that configuration has the same view of the shared module library and assets.

### Local artifact scheme

`local` resolves artifacts from the local filesystem. It is a filesystem artifact source, not a registry. It does not publish, pull, authenticate, or maintain a registry cache. A locally hosted OCI registry still uses `oci:`, for example:

```bicep
module storage 'oci:localhost:5000/bicep/modules/storage:v1'
```

A fully qualified `local:` path is resolved relative to the Bicep or Bicep parameter file containing the reference. A `local` alias path is resolved relative to the `bicepconfig.json` that declares the alias, as described above. Redirect targets are also relative to their declaring configuration file. Bicep normalizes `.` and `..` segments and uses the same platform-specific path and case handling as ordinary relative Bicep file references. These references are not resolved relative to the process working directory.

A local artifact reference identifies the artifact file, including its filename and extension:

```bicep
module storage 'local:modules/storage/main.bicep'
```

Compiled module JSON can also be referenced directly:

```bicep
module storage 'local:modules/storage/main.json'
```

An extension can reference a local package directly:

```bicep
extension 'local:extensions/foo.tgz' as foo
```

Alternatively, the extension can be assigned a name in `bicepconfig.json`:

```json
{
  "extensions": {
    "foo": "local:extensions/foo.tgz"
  }
}
```

and referenced by that name:

```bicep
extension foo
```

The `extensions` entry maps the declaration name `foo` to a fully qualified artifact reference. It is distinct from `artifacts.aliases`, which defines reusable artifact-location aliases. The expected artifact type is known from the declaration. Module references must end in `.bicep` or `.json`, and local extension references must end in `.tgz`. Bicep loads the referenced file directly and does not probe its containing directory for a conventional filename.

When a local `.bicep` module contains further references, its ordinary relative references and fully qualified `local:` references are anchored to that module file. The closest applicable `bicepconfig.json` is selected for each source file using existing configuration discovery rules. Artifact aliases and redirects therefore apply to transitive references as well as references in the entrypoint. A redirect is applied at most once to each reference; after it selects a local file, references inside that file are resolved normally. Existing module-cycle detection continues to report cycles involving local or redirected modules.

A local extension package is an explicitly trusted project input. Bicep applies the same package-format, manifest, extension compatibility, and archive-safety validation that it applies after restoring an OCI extension package. Invalid archives, unsafe archive paths, and incompatible manifests fail before the extension is loaded. A local package is not required to have a registry digest or signature because local development must allow the package to change in place. Teams that require immutable or signed extensions should use an OCI reference rather than `local:`. If a redirect replaces a remote extension identity with a local package, the redirect disclosure described below makes that substitution visible.

#### Data file loading

Local artifact references can be passed to compile-time file-loading functions. The reference includes the complete filename and is resolved through the same `local:` scheme or local alias as module and extension files.

```bicep
var notice = loadTextContent('local:assets/notice.txt')
var settings = loadJsonContent('local:assets/settings.json')
var deployment = loadYamlContent('assets::deployment.yaml')
var certificate = loadFileAsBase64('assets::certificates/service.pfx')
```

This behavior applies to `loadTextContent`, `loadJsonContent`, `loadYamlContent`, and `loadFileAsBase64`. Additional compile-time file-loading functions can adopt artifact references when their contracts explicitly do so. Ordinary relative paths remain supported unchanged. The loader continues to enforce its existing content, encoding, and file-size requirements after the artifact reference resolves to a file.

### `artifacts.redirects`

The `artifacts.redirects` configuration maps an artifact reference pattern to a relative local file. Named values captured by the reference pattern can be substituted into the target path.

A redirect allows a local implementation to stand in for a published artifact identity without requiring the local directory structure to mirror published versions. Redirects are intended for local development, testing unpublished modules, and testing local forks.

```json
{
  "artifacts": {
    "redirects": {
      "avm::res/storage/storage-account:{tag}": "./modules/storage/main.bicep"
    }
  }
}
```

The reference:

```bicep
module storage 'avm::res/storage/storage-account:0.32.1'
```

resolves directly to:

```text
./modules/storage/main.bicep
```

The original version does not need to exist locally.

#### Redirect target rules

A redirect target must:

- Start with `./` or `../`.
- Resolve to exactly one local artifact file.
- Be resolved relative to the configuration file in which the redirect is declared.
- Refer to a `.bicep` or `.json` file for a module import.
- Refer to a `.tgz` file for an extension import.
- Satisfy the existing filename, content, encoding, and size requirements of the function for a compile-time data-file load.

A redirect target must not be:

- An absolute filesystem path.
- A directory.
- An artifact reference such as `oci:`, `ts:`, `local:`, or an alias reference.
- Another redirect pattern.

Resolving paths relative to the declaring configuration file keeps redirects portable and makes their filesystem location unambiguous. A target can intentionally use `../` to select a sibling of the configuration directory. Path normalization uses the same rules as local artifact references; redirects do not introduce a separate filesystem sandbox.

Restricting targets to local files prevents redirect chains and cycles. Redirect matching is performed once; the selected file is loaded directly.

#### Capture syntax and validation

A redirect key can contain named captures:

- `{name}` captures exactly one artifact-reference segment and does not cross `/`, `:`, or `::` delimiters.
- `{...name}` captures one or more `/`-separated segments and does not cross `:` or `::` delimiters. A key can contain at most one multi-segment capture.

Capture names must start with a letter or underscore, contain only letters, digits, and underscores, and be unique within a key. A target substitutes a captured value with `{name}`, regardless of whether the key declared it as `{name}` or `{...name}`. A target can use each capture, use a capture more than once, or omit it. Every placeholder used by the target must be declared by the key.

Redirect patterns are templates over artifact references, not regular expressions or filesystem globs. Syntax such as `*`, `?`, `**`, character classes, and regular-expression groups is not supported. After substitution, the target must identify one explicit file.

A single-segment capture must not be empty, `.`, or `..`, and must not introduce a path separator. Each segment in a multi-segment capture is subject to the same validation. The substituted path is validated using the same relative-path and file-type rules as a target without captures. Captured values cannot add `.` or `..` segments, change the target root, or otherwise introduce traversal beyond the path already written in the target.

```json
{
  "artifacts": {
    "redirects": {
      "avm::res/storage/storage-account:{tag}": "./overrides/storage/{tag}.bicep",
      "oci:mcr.microsoft.com/bicep/extensions/microsoftgraph/v1.0:{tag}": "./extensions/{tag}.tgz"
    }
  }
}
```

This produces deterministic mappings:

```text
avm::res/storage/storage-account:0.31.1
  -> ./overrides/storage/0.31.1.bicep

avm::res/storage/storage-account:0.32.1
  -> ./overrides/storage/0.32.1.bicep

oci:mcr.microsoft.com/bicep/extensions/microsoftgraph/v1.0:1.0.0
  -> ./extensions/1.0.0.tgz
```

If the substituted file does not exist, resolution fails. Bicep does not search for another matching file.

#### Template Spec redirects

A Template Spec reference can redirect to a local implementation:

```json
{
  "artifacts": {
    "redirects": {
      "specs::network/vnet:{tag}": "./modules/network/main.json"
    }
  }
}
```

Redirecting to another remote artifact is prohibited. A remote redirect could cause a trusted-looking reference to restore content from an unexpected host, obscuring artifact provenance and introducing supply-chain or credential-disclosure risks. An alias can select a test registry when the source uses an alias.

#### Named capture and substitution

The target can omit a capture to map every matching version to one shared local file:

```json
{
  "artifacts": {
    "redirects": {
      "avm::res/compute/virtual-machine:{tag}": "./modules/virtual-machine/main.bicep"
    }
  }
}
```

This matches references such as:

```text
avm::res/compute/virtual-machine:0.21.0
avm::res/compute/virtual-machine:0.22.2
avm::res/compute/virtual-machine:dev
```

and redirects all of them to:

```text
./modules/virtual-machine/main.bicep
```

The target can include `{tag}` to map each version to a corresponding local file:

```json
{
  "artifacts": {
    "redirects": {
      "avm::res/compute/virtual-machine:{tag}": "./modules/virtual-machine/{tag}.bicep"
    }
  }
}
```

For example:

```text
avm::res/compute/virtual-machine:0.22.2
  -> ./modules/virtual-machine/0.22.2.bicep

avm::res/compute/virtual-machine:dev
  -> ./modules/virtual-machine/dev.bicep
```

Multi-segment captures allow one redirect to cover many modules under a registry prefix. This supports a local checkout containing modules whose repository paths have different depths:

```json
{
  "artifacts": {
    "redirects": {
      "oci:mcr.microsoft.com/bicep/avm/res/{...path}:{tag}": "./avm/res/{path}/main.bicep"
    }
  }
}
```

For example:

```text
oci:mcr.microsoft.com/bicep/avm/res/network/virtual-network:0.7.2
  -> ./avm/res/network/virtual-network/main.bicep

oci:mcr.microsoft.com/bicep/avm/res/compute/virtual-machine:0.21.0
  -> ./avm/res/compute/virtual-machine/main.bicep
```

Here, `path` captures the repository path below `res`, while `tag` captures the version. The target intentionally omits `tag` because the local checkout has one active implementation of each module.

#### Matching precedence

Redirects can target either the spelling used in source or the canonical artifact identity. Bicep first parses the source reference without loading the artifact. It then applies redirect matching in this order:

1. An exact match against the source spelling.
2. A capture pattern against the source spelling, choosing the pattern with the most literal characters.
3. If the source is an alias, expand it without restoring the artifact.
4. An exact match against the canonical, fully expanded identity.
5. A capture pattern against the canonical identity, choosing the pattern with the most literal characters.
6. No redirect.

For registry artifacts, the canonical identity uses the `oci:` spelling even when the source used `br`, `br/public`, or an alias. Source-spelling matches take precedence so a project can override one intentional spelling without affecting another alias that expands to the same location. Canonical matches allow one redirect to cover legacy, fully qualified, and alias references that identify the same artifact.

For Template Specs, the canonical identity uses the fully qualified `ts:` spelling even when the source uses `ts/<alias>:` or a new artifact alias. For local artifacts, the canonical identity is the normalized local path. A source-spelling redirect remains preferable for portable local references because an absolute normalized path is machine-specific.

If equally specific capture patterns match the same reference in the same matching phase, configuration validation fails rather than depending on declaration order.

```json
{
  "artifacts": {
    "redirects": {
      "avm::res/storage/storage-account:0.32.1": "./overrides/storage-0.32.1.bicep",
      "avm::res/storage/storage-account:{tag}": "./overrides/storage.bicep"
    }
  }
}
```

The exact reference `avm::res/storage/storage-account:0.32.1` uses `./overrides/storage-0.32.1.bicep`.

#### Build disclosure and cache behavior

When a redirect is applied, Bicep emits the configurable `artifact-redirect` analyzer diagnostic once per redirected artifact during a compilation. The diagnostic identifies both the source artifact reference and the resolved local path, making the substitution visible in ordinary builds. Its default level is `warning`.

The rule uses the existing analyzer configuration and supports the standard `error`, `warning`, `info`, and `off` levels:

```json
{
  "analyzers": {
    "core": {
      "rules": {
        "artifact-redirect": {
          "level": "error"
        }
      }
    }
  }
}
```

Setting the level to `error` prohibits redirects: Bicep can resolve the redirect to produce useful diagnostics, but the compilation fails and does not emit a successful output artifact. Setting the level to `info` retains non-warning disclosure, while `off` suppresses the diagnostic without changing redirect resolution. An individual occurrence can use the standard suppression directive:

```bicep
#disable-next-line artifact-redirect
module storage 'avm::res/storage/storage-account:0.32.1'
```

Applied redirects remain visible in verbose CLI output even when the analyzer diagnostic is suppressed. Diagnostic suppression affects presentation and build policy, not resolution or provenance metadata.

Redirected content bypasses the registry restore cache and must never be stored under the source artifact's remote identity. The local file remains a dependency of the compilation so changes invalidate incremental build results. This proposal does not introduce a lock file. If Bicep later introduces an artifact lock file or resolution manifest, redirected entries should record the source identity, resolved local path, declaring configuration file, and local content digest rather than presenting the source as a successfully restored remote artifact.

### Incremental adoption from `br` to `oci`

This proposal makes `oci` the canonical scheme for registry-backed artifacts. The name `br` was originally inspired by Terraform Registry terminology, but it is too generic as Bicep expands beyond one registry model. It does not identify whether an artifact comes from MAR, a private Azure Container Registry, or a locally hosted registry.

`oci` identifies the underlying artifact format and distribution protocol. The same scheme therefore applies consistently to MAR, private registries, locally hosted OCI registries, modules, extensions, and future OCI artifact types. The `local` scheme remains distinct because it resolves files directly and is not a registry.

#### Public registry alias

The existing predefined public registry reference:

```bicep
module storage 'br/public:avm/res/storage/storage-account:0.32.1'
```

has the following canonical form:

```bicep
module storage 'avm::res/storage/storage-account:0.32.1'
```

For AVM artifacts, `avm::` is the recommended spelling for new references. Other artifacts under the existing public registry can use `mar::<artifact>`. The name `mar` identifies Microsoft Artifact Registry rather than implying that every artifact in the registry is public or part of AVM.

#### Fully qualified references

An existing fully qualified `br` reference:

```bicep
module storage 'br:mcr.microsoft.com/bicep/avm/res/storage/storage-account:0.32.1'
```

has the following canonical `oci` form:

```bicep
module storage 'oci:mcr.microsoft.com/bicep/avm/res/storage/storage-account:0.32.1'
```

Both forms identify the same registry artifact.

#### Independent alias namespaces

Legacy module aliases and artifact aliases are selected by source syntax and are resolved independently:

- `br/public:<artifact>` always uses the existing predefined `public` alias.
- `br/<alias>:<artifact>` resolves `<alias>` only from `moduleAliases.br`.
- `ts/<alias>:<artifact>` resolves `<alias>` only from `moduleAliases.ts`.
- `<alias>::<artifact>` resolves `<alias>` only from `artifacts.aliases` or the predefined artifact aliases.
- A fully qualified `br:<registry>/<repository>:<tag>` or `ts:<subscriptionId>/<resourceGroup>/<templateSpec>:<version>` reference does not use either alias map.

Defining an alias only in `artifacts.aliases` does not make `br/<alias>:` or `ts/<alias>:` valid. Bicep does not fall back between either legacy alias map and `artifacts.aliases`. If a legacy alias reference has no corresponding entry, resolution fails with the existing unknown-module-alias diagnostic.

The same name can exist in both maps, including with different targets. This is unambiguous because the reference syntax selects the map. Bicep does not warn when both maps contain the same name or when their values differ.

#### Migrating a configured alias

To migrate incrementally, a project can define equivalent aliases in both maps:

```json
{
  "moduleAliases": {
    "br": {
      "company": {
        "registry": "contoso.azurecr.io",
        "modulePath": "bicep/modules"
      }
    }
  },
  "artifacts": {
    "aliases": {
      "company": {
        "type": "oci",
        "registry": "contoso.azurecr.io",
        "repositoryPrefix": "bicep/modules"
      }
    }
  }
}
```

Existing and migrated references can then coexist:

```bicep
module existing 'br/company:storage/storage-account:1.0.0'

module migrated 'company::storage/storage-account:1.0.0'
```

Each reference can be migrated independently. The `moduleAliases.br.company` entry can be removed after no `br/company:` references remain, but removal is optional because `moduleAliases` remains supported. Bicep does not emit migration or deprecation warnings for `br` references or `moduleAliases` configuration.

A Template Spec alias migrates in the same way: `moduleAliases.ts.specs` and an `artifacts.aliases.specs` entry with `type: "ts"` can coexist, while `ts/specs:webapp:v1` and `specs::webapp:v1` continue to select their respective maps.

#### Overriding a predefined alias

An `artifacts.aliases` entry can override the OCI location of a predefined alias. For example, an organization can resolve `avm::` references through a private mirror:

```json
{
  "artifacts": {
    "aliases": {
      "avm": {
        "type": "oci",
        "registry": "contoso.azurecr.io",
        "repositoryPrefix": "bicep/avm"
      }
    }
  }
}
```

The alias is used through the new `<alias>::<artifact>` syntax:

```bicep
module storage 'avm::res/storage/storage-account:0.32.1'
```

With this configuration, the reference resolves to:

```text
oci:contoso.azurecr.io/bicep/avm/res/storage/storage-account:0.32.1
```

Changing `avm` or `mar` to another alias type is invalid:

```json
{
  "artifacts": {
    "aliases": {
      "avm": {
        "type": "local",
        "path": "./avm"
      }
    }
  }
}
```

The configuration schema and compiler report this as an invalid predefined-alias override.

#### Backward compatibility guarantees

- Existing `br/public:` references continue to resolve unchanged.
- Existing fully qualified `br:` references continue to resolve unchanged.
- Existing `br/<alias>:` references continue to use `moduleAliases.br` configuration.
- Existing `ts/<alias>:` references continue to use `moduleAliases.ts` configuration.
- `artifacts.aliases` entries are never used to resolve `br/<alias>:` or `ts/<alias>:` references.
- Existing `moduleAliases` configuration remains supported.
- Old and new reference forms can coexist in the same project.
- Adopting a Bicep version that supports this proposal does not require source or configuration migration.
- This proposal does not deprecate `br` references or `moduleAliases` and does not emit migration or deprecation warnings for either form.

#### Future deprecation of `moduleAliases`

Deprecating `moduleAliases` requires a separate REP. Before marking `moduleAliases` as deprecated in the configuration schema, Bicep should provide CLI commands that:

- Detect stale configuration and source features, including `moduleAliases` and `br/<alias>:` references.
- Report what would change without modifying files.
- Convert `moduleAliases` entries to equivalent `artifacts.aliases` entries.
- Rewrite affected source references while preserving fully qualified artifact identity.
- Support repository-wide validation so an old alias is not removed while references to it remain.

Only after this tooling is published and users have an adoption period should a future REP consider marking `moduleAliases` as deprecated in the configuration schema. That REP must define the deprecation timeline, editor and CLI diagnostics, treatment of `br/<alias>:`, and any eventual removal criteria. The predefined `br/public:` alias is unaffected by the absence or deprecation of user-defined `moduleAliases`.

### Client side changes

The Bicep CLI, compiler, language server, configuration schema, and artifact dispatcher require changes to:

- Parse and validate alias references that use `::`.
- Recognize `oci:` and `local:` schemes.
- Resolve and validate typed `artifacts.aliases` entries.
- Keep `moduleAliases.br`, `moduleAliases.ts`, and `artifacts.aliases` resolution independent, without cross-map fallback or coexistence warnings.
- Provide the predefined `mar` and `avm` aliases, allow their OCI locations to be overridden, and reject overrides that change their type.
- Apply `artifacts.redirects` before loading an artifact.
- Match redirects against source spelling before canonical identity, without restoring the artifact during alias expansion.
- Implement the configurable `artifact-redirect` analyzer rule with a default level of `warning` and disclose applied redirects in verbose CLI output.
- Keep redirected local content out of the registry restore cache.
- Resolve artifact references from both direct extension declarations and values in the existing `extensions` configuration map.
- Load fully qualified local module and extension files.
- Resolve local artifact references passed to compile-time file-loading functions, including `loadTextContent`, `loadJsonContent`, `loadYamlContent`, and `loadFileAsBase64`.
- Preserve existing `br` and `moduleAliases` behavior.
- Produce diagnostics for invalid aliases, ambiguous redirects, invalid capture patterns, invalid local targets, and missing local artifacts.
- Ensure editor navigation, completion, restore, build, and publish-related experiences understand the new reference forms where applicable.

Resolution follows one client-side pipeline: parse the source reference, try source-spelling redirects, expand an alias when present, canonicalize through the scheme's existing artifact handler, try canonical-identity redirects, and then restore or load the selected artifact. The compiler and language server use the same resolver so build, diagnostics, completion, and navigation do not disagree about the selected artifact.

### Server side changes

No server-side changes are required. Alias expansion, redirects, local artifact loading, and compatibility handling occur in Bicep tooling before template deployment.

### Microsoft.Resources/deployments API changes

No changes to the `Microsoft.Resources/deployments` API or generated ARM template contract are required.

### End-to-end examples

The predefined `avm` alias shortens a production reference without requiring configuration:

```bicep
module storage 'avm::res/storage/storage-account:0.32.1'
```

A development configuration can redirect the same source reference to a local implementation:

```json
{
  "artifacts": {
    "redirects": {
      "avm::res/storage/storage-account:{tag}": "./modules/storage/main.bicep"
    }
  }
}
```

The Bicep source does not change between production and development.

## Tradeoffs

- Supporting both legacy and new artifact-reference models expands the language and configuration surface area. Bicep must maintain both models because this proposal does not require migration or deprecate the existing forms.
- As with existing `moduleAliases`, artifact aliases trade source-level location visibility for shorter references and location changes without source edits. This proposal extends that existing indirection to additional artifact types. Predefined `mar` and `avm` aliases can also resolve through a configured OCI mirror, although their artifact type cannot be changed.
- Local artifacts and redirects improve development workflows at the cost of registry-backed immutability and reproducibility. Redirect disclosure, direct dependency tracking, package validation, and exclusion from the remote cache reduce the risk of mistaking local content for a restored artifact.

## Alternatives

### Extend `moduleAliases`

`moduleAliases` could be extended with more artifact types. This would preserve the existing configuration shape, but the name and nested `br`/`ts` structure remain module-specific and do not provide a clean model for extensions or future artifacts. It would also retain the slash-based alias syntax that this proposal separates from schemes.

### Require fully qualified references

Bicep could support only fully qualified `oci:`, `ts:`, and `local:` references. This would minimize configuration, but users would repeatedly encode registry hosts, Azure scopes, and repository prefixes in source. It would also make moving an artifact location costly across a large codebase.

### Use scheme-qualified aliases

Aliases could use forms such as `oci/mar:` or extend `br/<alias>:`. This makes the source type explicit, but it leaks the alias target type into every reference and prevents an alias from abstracting where an artifact is hosted. The proposal instead uses `::` to distinguish aliases from schemes.

### Redirect to remote artifacts

Redirect targets could accept another artifact reference. This would support remote test registries, but it would introduce redirect chains and cycles and could silently restore a trusted-looking identity from an unexpected host. Aliases provide the remote-location scenario without obscuring provenance.

A future proposal could add `artifacts.allowUnsafeRemoteArtifactRedirects` as an explicit opt-in for remote targets. This proposal does not define or enable that setting; all redirect targets remain restricted to relative local files. The nested `artifacts` object allows this and other artifact-resolution policies to be added without introducing more top-level configuration properties.

### Allow general globbing and fallback targets

Redirects could support filesystem globs, character classes, regular expressions, or ordered arrays of fallback targets like some source-resolution systems. This would be more flexible, but resolution would become dependent on filesystem contents, complex matching rules, or target order. Named captures and one explicit target keep the result deterministic.

### Use relative module paths for all local development

Consumers can replace artifact references with relative paths while developing. This requires source changes, can produce accidental commits, and does not test the same artifact identity used by production. Redirects preserve the source reference and move the development override into configuration.

## Rollout plan

This additive client-side feature does not require an experimental feature flag or backend rollout. It will ship in a regular Bicep release after implementation, documentation, and automated tests are complete. Tests will cover parsing and configuration validation, artifact resolution, redirects and captures, cross-platform local paths, language-server behavior, and backward compatibility for existing `br`, `ts`, and `moduleAliases` references. Post-release bugs will be handled through the normal Bicep issue and servicing process.

## Unresolved questions

No unresolved question blocks the functional design. Diagnostic codes and final wording can be finalized during implementation without changing artifact identity or resolution behavior. Any change to syntax, precedence, path anchoring, predefined-alias type restrictions, or backward compatibility requires an update to this REP before stabilization.

## Out of scope

- Deprecating or removing `moduleAliases`, `br/<alias>:`, or `ts/<alias>:` references. A separate REP must define migration tooling, schema deprecation, diagnostics, timelines, and removal criteria. The predefined `br/public:` alias remains supported independently of user-defined `moduleAliases`.
- Defining environment-specific configuration discovery, selection, inheritance, or merging. That behavior requires a separate REP.
- Defining inheritance for extension configuration or declarations. Extension inheritance is related because inherited extension declarations would make it easier to redirect extensions to local packages consistently across a repository without repeating configuration, but it will be addressed separately.
- Redirecting one remote artifact reference to another remote artifact. A future proposal may introduce the explicit `artifacts.allowUnsafeRemoteArtifactRedirects` opt-in, but this setting is not part of the current design.
- Defining a general-purpose source import alias or filesystem globbing system.
- Publishing, authenticating, or maintaining a registry cache for `local:` artifacts.
- Changing OCI distribution protocols, registry authentication, Template Spec APIs, or the ARM deployment contract.
- Defining new artifact package formats beyond the existing Bicep module JSON and extension `.tgz` formats.