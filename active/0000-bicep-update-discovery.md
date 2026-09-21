---
REP Number: "Unassigned"
Author: andyleejordan (Andy Jordan)
Start Date: 2026-09-18
Feature Status: "Proposed; not shipped"
---

# Bicep update discovery and Dependabot integration

## Summary

Add a `bicep outdated` command that reports available newer module and resource API versions, using Bicep's existing language and version-discovery code. Discovery is read-only. Dependabot would consume versioned JSON output to select and propose updates under the user's configured rules. The design should also accommodate extensions and potentially compiler constraints later.

**Open:** whether Bicep should also apply explicitly requested version edits using shared quick-fix machinery, through a separate operation or an explicit write mode. This would let Dependabot delegate file editing to Bicep without delegating its update policy.

Start with public modules. Whether v1 covers AVM only or a broader set of public OCI modules remains open, as does which side performs registry lookups for Dependabot.

The first Dependabot integration updates modules only. Resource API reporting is included in the CLI, but resource API PRs would require a separate, explicitly enabled Dependabot ecosystem.

## Terms and definitions

- **Module artifact**: A reusable Bicep module published in a registry. A dependency on it can occur in a module declaration, a compile-time import, or a Bicep parameters file's `using` declaration.
- **Artifact identity**: The resolved source and repository of a dependency, together with its original tag or digest. Alias spelling is not identity.
- **Project compiler constraint**: The accepted compiler version or range declared in `bicepconfig.json`. It is separate from the version of the Bicep CLI that Dependabot runs for analysis.
- **Candidate provider**: Code that obtains available versions and metadata for a dependency kind. Modules and resource APIs need not use the same provider or protocol.
- **Authoritative source**: The source of available versions for a dependency. For modules, this is the registry/repository selected by the project's effective configuration, including an intentionally curated mirror. Resource API discovery will identify its type-data source separately.
- **Update policy**: Rules selecting which candidates should become update proposals. Dependabot policy belongs in Dependabot configuration, not Bicep configuration.
- **Credential proxy**: Dependabot's mechanism for keeping configured registry credentials outside the updater while authenticating its registry traffic.

## Motivation

Bicep users can pin reusable modules, but discovering updates and maintaining those pins requires additional tooling. The existing `use-recent-module-versions` linter demonstrates part of the capability for public modules, but a diagnostic and quick fix are not a complete automation interface.

The concrete request is [dependabot/dependabot-core#15952](https://github.com/dependabot/dependabot-core/issues/15952), whose initial scope is public AVM module updates. This proposal also considers broader public OCI module support.

We want the work needed for Dependabot to also provide a useful native command. Keeping language knowledge in Bicep avoids another parser that must track reference syntax, aliases, configuration inheritance, and source locations. Keeping the integration at the CLI boundary avoids asking consumers to depend on a supported public compiler-library API.

### Feasibility

A local prototype used Bicep's parser and Dependabot's version selection to update a public AVM module from `0.18.0` to `0.33.1`, preserving source formatting and resource API versions. It exercised the registered Dependabot runner with mounted source; packaged-image and hosted-service validation remain release work.

## Detailed design

### Client side changes

#### Scope and ecosystem enrollment

Dependabot does not require a separate ecosystem for every package kind. Its [Terraform integration](https://github.com/dependabot/dependabot-core/blob/main/terraform/lib/dependabot/terraform/file_parser.rb) updates both modules and providers from the same files under one ecosystem. We choose separate ecosystems because enabling module updates must not also enable resource API updates, now or when resource support is added.

These are different kinds of dependencies with different update expectations. An OCI module is a published package identified by its registry, repository, and tag or digest. A resource API reference selects a service contract for an Azure resource type, not a downloadable module package. Its date-based versions are not SemVer: newer does not establish compatibility or a need to upgrade. The Bicep team describes the risk of deployment failures or behavioral changes without a corresponding benefit in [Azure/bicep#8324](https://github.com/Azure/bicep/issues/8324#issuecomment-1577531181). Users must explicitly enable resource API updates and test the changes; enabling a linter warning is not that opt-in.

Dependabot's separate [Cargo](https://github.com/dependabot/dependabot-core/tree/main/cargo) and [Rust toolchain](https://github.com/dependabot/dependabot-core/tree/main/rust_toolchain) integrations illustrate separate enrollment within one language. Unlike those integrations, Bicep modules and resource APIs share source files and can share parsing code. Terraform providers are plugin packages, closer to Bicep extensions than to individual resource API versions.

This contribution uses existing Dependabot configuration behavior. `allow.dependency-type` describes direct/transitive or production/development relationships, not Bicep-specific categories. We are not adding new category values or activation rules.

The working names are `bicep-modules` for module updates and `bicep-resources` for a future resource API integration. **Open:** final identifiers need agreement with the Dependabot maintainers.

```yaml
version: 2
updates:
  - package-ecosystem: "bicep-modules"
    directory: "/infra"
    schedule:
      interval: "weekly"
```

The entry exposes only module dependencies. Existing `allow`, `ignore`, and grouping rules operate within that inventory. A broad rule such as `dependency-type: all` cannot activate resource API updates.

A future resource API integration would require its own entry:

```yaml
  - package-ecosystem: "bicep-resources"
    directory: "/infra"
    schedule:
      interval: "monthly"
```

This also allows independent schedules.

**Why not separate them with `allow.dependency-name`?** Dependabot's existing [name filter](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#dependency-name-allow) can distinguish modules and resource APIs if their dependency names are unambiguous. For example, a single ecosystem could assign names with `module:` and `resource-api:` prefixes and let users allow `module:*`. Those prefixes are illustrative, not a proposed naming contract. This would work as a filter without adding a new configuration key.

It would not, however, make resource API updates independently opt-in by default. Under normal Dependabot behavior, omitting `allow` permits updates to explicitly declared dependencies, and `dependency-name: "*"` matches both categories. If a module-only ecosystem later began exposing resource APIs, existing jobs with either configuration could start proposing API changes without a new enrollment decision. A carefully configured module-only allowlist would remain safe; the problem is requiring that allowlist to make the default safe.

Separate ecosystems preserve explicit enrollment without changing the meaning of `allow`. Users can still use name filters within each ecosystem.

The CLI should allow additional dependency kinds without broadening existing module-only requests. Dependabot enrollment for extensions and compiler constraints remains open.

#### Which files are checked?

`bicep outdated` needs to support both Dependabot's directory-based configuration and a Bicep user asking about a particular deployment file.

For Dependabot, the configured directories and exclusions determine which source files are eligible. **Recommendation:** check all supported Bicep source files within that scope, rather than requiring a designated `main.bicep` and following only its references. A repository can contain several independently deployed files, examples, and tests. Whether those are included should follow the user's Dependabot configuration, not whether another file happens to reference them.

For a Bicep user, **recommendation:** `bicep outdated main.bicep` checks that file and the local Bicep files it references, recursively, without scanning unrelated files alongside it. Published module internals are not traversed. This answers a different question from a directory scan: which versions are used by this deployment?

**Open:** how should Dependabot tell `outdated` which files to check? **Recommendation:** Dependabot chooses the files using its existing directory and exclusion settings, then passes that list to Bicep. Bicep checks only the listed files. The CLI syntax for accepting that list remains to be decided; Bicep does not need to read `dependabot.yml`.

Bicep may still need to read other files to understand a selected file. For example, checking `/infra/main.bicep` may require reading a parent `bicepconfig.json`. Reading that configuration does not make it a target for version updates. Likewise, if a selected file references a Bicep file that Dependabot excluded, that reference must not cause the excluded file's dependencies to be checked or changed.

For the user-facing command, we still need to decide how far to follow local references: whether to include files outside the starting directory, what to report when a referenced file is missing, and whether a `.bicepparam` input should also check the template named by its `using` declaration.

#### Which module references can be checked?

We propose starting with public registry modules pinned to an exact semantic version, such as:

```bicep
module storage 'br/public:avm/res/storage/storage-account:0.6.0' = {
  // ...
}
```

For this declaration, `outdated` would identify the module's registry, repository, and current version, then look for newer versions in that same repository. Whether the first release supports only AVM or other public OCI modules is still open.

Modules can also be referenced by `import ... from` in a Bicep file or `using` in a `.bicepparam` file. **Open:** should the first release check these references as well as `module` declarations? We should recognize them even if checking their versions is deferred, so the output can say they were not checked. Separately, we need to decide which of these references Dependabot can update; being able to report a newer version does not by itself provide editing support.

**Resolve aliases using the project's configuration.** We propose reusing Bicep's own reference and configuration handling rather than interpreting aliases separately. For example, `br/public:...` usually points to MCR, but a project can redefine `public` to point to a private registry. In that case, a public-only check should report that it cannot check the reference, not look for a replacement on MCR. Each source file should use its applicable configuration and inherited settings, following [REP 0023](https://github.com/Azure/bicep-reps/blob/main/final/0023-bicep-configuration-inheritance.md). Aliases and local redirects should follow Bicep's supported behavior, including the design in [REP 0022](https://github.com/Azure/bicep-reps/blob/main/active/0022-artifact-aliases-and-redirects.md).

**Keep each use of a module distinct.** If two declarations use the same module at different versions, report both versions and their file locations. Keep the original reference spelling as well as its resolved registry and repository, so a later edit can preserve aliases. Do not inspect or edit dependencies inside the published module.

**Say when a reference could not be checked.** The first release would not check private registries, Template Specs, or digest-only references for newer versions. Tags that cannot be compared as versions and references redirected to local files also need an explicit result, rather than being labeled up to date. Broken configuration or an alias that cannot be resolved should produce an error explaining the problem. Documentation and tests should distinguish references that can be checked, references that can be edited, and those not yet supported.

#### Native command and JSON boundary

`bicep outdated` is the shared entrypoint for update discovery, not a module-only command. Separate Dependabot ecosystems select the dependency kind they need; they do not require separate Bicep commands. Discovering a newer resource API version does not imply that it should be adopted or that it is compatible.

Follow existing Bicep CLI conventions: a positional input path and named options with values for selections. For example, [`generate-params`](https://github.com/Azure/bicep/blob/main/src/Bicep.Cli/Commands/GenerateParametersFileCommand.cs) uses `--output-format` and `--include-params`. Reuse `--output-format` and follow the same selection pattern with `--kind`, rather than separate `--modules` and `--resources` switches.

Proposed user experience:

```sh
bicep outdated main.bicep
bicep outdated main.bicep --kind module --output-format json --schema-version 1
```

Resource API reporting uses the same command:

```sh
bicep outdated main.bicep --kind resource-api --output-format json --schema-version 1
```

Explicit kind selection keeps a module-only request module-only as the CLI gains capabilities. Requesting an unsupported kind should produce an error.

Human output summarizes available newer versions; JSON provides enough version information for Dependabot to apply its rules. **Open:** the selector syntax, the default when no kind is specified, and whether one call can request multiple kinds.

Ordinary `bicep outdated` leaves project files unchanged; any editing capability requires an explicit request. Registry metadata caching should be documented.

Dependabot already consumes JSON from native tools and helpers. Bicep should define its own public schema rather than expose Dependabot's internal helper protocol.

Schema v1 must define:

| Area | Required meaning |
| --- | --- |
| Execution | Schema identifier, Bicep CLI version, requested dependency kinds and file scope, completion state |
| Configuration | Source files read, configuration files applied to each source, and where the compiler constraint was declared |
| Inventory | Dependency kind, stable identity, original reference, declared version or constraint, and kind-specific source details |
| Occurrences | Repository-relative source path and unambiguous edit location |
| Candidates | Available versions, their version scheme, available metadata and where it came from; module candidates also retain original tag spelling |
| Diagnostics | Stable code, affected dependency/file, severity, unsupported or failed operation |

**Open:** field names and JSON types, how source positions are counted, whether range ends are inclusive, text encoding, result ordering, and how to reject edits if the file has changed since analysis. These need agreement before Dependabot uses the locations to edit files. The JSON exposes purpose-built records, not internal compiler syntax-tree objects.

The shared result format carries kind-specific records: module identity includes a registry and repository, while resource API identity identifies a resource type and its API version. Resource APIs must not need placeholder registry fields or be converted to SemVer to fit the schema.

Future extension records can describe their resolved artifact source and version without treating them as modules. Potential compiler records need to preserve the declared constraint or range, its configuration-file location, and candidate compiler releases. A constraint is not a single installed version: reporting a newer release that satisfies it differs from reporting a release that would require changing it. The shared format must allow that distinction without deciding compiler-update policy now.

Keep common fields for kind, identity, declared version or constraint, source occurrences, candidates, and diagnostics, with kind-specific details where needed. Adding extensions or compiler reporting should not require a second command or inventing module-shaped identities. Their selectors, candidate sources, and comparison rules will be defined with those features.

Compatibility tests must show that adding a dependency kind does not change results for existing module-only requests. An unknown kind must never be interpreted as a module by a consumer.

Machine mode writes one JSON document to stdout; progress goes to stderr. Outdated dependencies are a successful analysis result. Errors and unsupported cases must not be encoded as empty successful candidate lists.

**Open:** exit codes and how callers handle partial results. These behaviors need agreement and protocol tests before the interface is declared stable.

Additive schema changes must not break existing consumers. Breaking changes require explicit schema negotiation/versioning. No public .NET API compatibility promise is introduced.

#### Separate analysis from version lookup

The design separates three responsibilities:

1. Bicep discovers references and defines their language/version semantics.
2. A candidate provider retrieves versions from the appropriate source for that dependency kind.
3. The consumer chooses updates according to its policy.

The standalone command needs a Bicep-owned module candidate provider. Resource API discovery reuses Bicep's existing API-version provider rather than OCI registry access. Future extension and compiler discovery can supply their own version sources and comparison rules through the same separation of responsibilities. **Open:** which module provider the Dependabot adapter uses:

| Approach | Benefit | Cost or risk |
| --- | --- | --- |
| Bicep performs lookups | Shared native command and automation path; existing registry knowledge | Azure SDK/ORAS behavior must work with Dependabot's proxy and noninteractive environment |
| Dependabot performs lookups | Reuse ecosystem registry experience and proxy-compatible request paths | Requires structured offline discovery and a way to retain shared Bicep version analysis |

If Dependabot performs lookup, the CLI contract must allow analysis without network access and, if needed, accept externally supplied candidate data. Exact commands and data exchange are not settled. Whichever side performs lookup, version ordering needs one defined meaning. Any comparison code required on both sides must follow the same rules and conformance tests.

Existing public module metadata and direct registry queries are both candidates for v1. The public index's freshness, coverage, and metadata guarantees must be evaluated rather than treated as equivalent to a live registry listing.

Lookup targets the known module repository rather than requiring a registry-wide catalog. Original tag spelling is retained separately from the parsed version. OCI transport reuse does not imply adopting container image tag-selection rules.

#### Open question: should Bicep apply version edits?

The resource API linter already creates a precise text replacement for its quick fix. We could reuse that machinery to let Bicep apply version edits, or have the Dependabot adapter apply edits using locations reported by Bicep.

Native editing has precedent in established Dependabot ecosystems:

| Ecosystem | Existing behavior |
| --- | --- |
| [NuGet](https://github.com/dependabot/dependabot-core/blob/main/nuget/helpers/lib/NuGetUpdater/NuGetUpdater.Core/Updater/FileWriters/XmlFileWriter.cs) | A C# updater edits package references in project and shared MSBuild files. This is the closest precedent for native source-reference edits, though it is Dependabot helper code rather than a general-purpose public CLI. |
| [Cargo](https://github.com/dependabot/dependabot-core/blob/main/cargo/lib/dependabot/cargo/file_updater/lockfile_updater.rb) | Dependabot invokes `cargo update` to rewrite `Cargo.lock`; manifest editing remains separate. |
| [npm](https://github.com/dependabot/dependabot-core/blob/main/npm_and_yarn/lib/dependabot/npm_and_yarn/file_updater/npm_lockfile_updater.rb) | Dependabot invokes `npm install --package-lock-only` to update the lockfile. This is native lockfile editing, not evidence that all manifest edits are native. |

The existing Bicep quick fix is not yet the required automation contract. It selects the first version in the linter's acceptable-version list and is attached to a diagnostic. Dependabot needs to request an exact target for particular occurrences, including when the current reference produces no warning.

If Bicep owns editing, Dependabot would pass selected targets to a separate command or explicit write mode, then collect the changed files from its temporary workspace. Automatic target selection for CLI users would need its own policy.

A native editing interface would need to preserve formatting, comments, alias spelling, and per-occurrence versions; reject stale or ambiguous inputs; constrain edits to requested files and references; and report changed files and failures. Multi-file failure behavior and previewing proposed edits also need definition. Tests should cover Unicode, repeated references, and unchanged surrounding text.

**Open:** whether native editing belongs in the first increment, which reference kinds it supports, and whether it uses a separate command or an explicit write mode. Module editing would need its own coverage alongside resource API quick fixes.

#### Comparison with Dependabot's Terraform integration

Terraform provides a useful example of splitting these responsibilities:

| Existing Terraform behavior | Implication for this proposal |
| --- | --- |
| A native `hcl2json` helper parses source; Ruby reads its JSON output. | Native parsing and a JSON boundary fit established Dependabot practice. |
| A Ruby registry client retrieves module and provider versions. | Registry lookup need not live in Bicep simply because parsing does. Terraform's authentication path does not by itself prove OCI credential-proxy compatibility. |
| Modules and provider packages share one ecosystem. | Separate package kinds alone do not require separate ecosystems; our deciding requirement is explicit resource API opt-in. |
| Module update checking does not perform a separate project compatibility check. | A proposed update is not proof that the project builds or deploys successfully. |
| Provider lockfile maintenance invokes `terraform providers lock` and sometimes `terraform init`. | Native dependency operations are distinct from project validation. |
| The updater handles multiple requirements for the same module. | Each Bicep reference needs its own current version and edit location. |

The relevant implementations are the [parser](https://github.com/dependabot/dependabot-core/blob/main/terraform/lib/dependabot/terraform/file_parser.rb), [registry client](https://github.com/dependabot/dependabot-core/blob/main/terraform/lib/dependabot/terraform/registry_client.rb), [update checker](https://github.com/dependabot/dependabot-core/blob/main/terraform/lib/dependabot/terraform/update_checker.rb), and [file updater](https://github.com/dependabot/dependabot-core/blob/main/terraform/lib/dependabot/terraform/file_updater.rb).

#### Private registries: v2, with v1 design constraints

Private registries are planned for v2. The initial design should keep version lookup separate from source analysis so authenticated providers can be added later.

V2 requires testing repository-scoped read permissions, pagination, authentication challenges, proxy routing and certificate trust, and artifact metadata access. Successful local Azure login or successful module restore does not prove version discovery works through Dependabot.

Credentials must not be embedded in JSON analysis or persisted into project files. Dependabot operation must not require interactive login or real registry secrets inside the updater.

A curated internal feed can intentionally lag the public source. Only versions available from the configured source are candidates. Authentication failure must not trigger public fallback, and a private alias must not be redirected to the public index based on its name.

#### Version policy and validation boundary

Module version interpretation should reuse or reconcile Bicep's existing comparison logic. The public-module linter uses strict SemVer, while the compiler-pinning parser has a different accepted grammar. They are not automatically interchangeable.

What `bicep outdated` reports as available is separate from what Dependabot proposes under its configured rules. Availability is not a recommendation to adopt a version, especially for resource APIs. The JSON must retain enough information for Dependabot to apply ignores, prerelease rules, and cooldowns; a single "latest" answer is insufficient.

**Open:** which available versions the CLI displays by default, and separately how the Dependabot adapter maps module versions to existing update rules. Scenarios must cover major/minor/patch changes for `0.x`, prerelease transitions, non-SemVer tags, withdrawn versions, and missing publication dates. A SemVer label is not proof that a module change is safe.

Cooldown support is part of Dependabot's ecosystem contribution process. **Open:** where reliable publication dates come from and what happens when they are unavailable. This needs agreement before release. Tag ordering is not a release date, and a timestamp that changes when an artifact is modified is not necessarily its first publication time.

The integration would update source references without requiring project compilation or deployment validation. Users review the PR and validate through their own CI.

#### Compiler pinning

Dependabot would invoke a Bicep CLI version that supports the JSON interface, independently of the project's compiler pin.

We propose allowing read-only discovery to inspect projects whose compiler constraint excludes that CLI version, without changing the constraint. Builds and deployments would continue to enforce the pin defined in [Azure/bicep-reps#11](https://github.com/Azure/bicep-reps/pull/11).

This needs a dedicated analysis path, not compilation with diagnostics suppressed. Unsupported syntax should produce an error, and available versions should not be presented as compatible with the project's pinned compiler.

#### Reuse of existing Bicep behavior

We propose reusing Bicep's parser, artifact handling, configuration resolution, public module metadata, registry transports, and applicable comparison logic. Extracting shared code must preserve the existing linter's behavior.

Current metadata helpers include convenience paths that convert failures into empty results. An automation path must expose errors explicitly rather than copying that behavior.

#### Resource API discovery in v1

Bicep already has a [`use-recent-api-versions` linter](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/linter-rule-use-recent-api-versions). It identifies old API versions, lists acceptable alternatives, and offers a quick fix for resource declarations. Its [implementation](https://github.com/Azure/bicep/blob/2cef422d196e59ca959a411d3bd036035f37c7df/src/Bicep.Core/Analyzers/Linter/Rules/UseRecentApiVersionRule.cs) obtains known versions through `IApiVersionProvider`, separates stable and preview versions, and returns both all known versions and a policy-filtered set. This existing discovery capability brings resource API reporting into v1; no new Azure discovery service is proposed.

Reuse that version lookup and date/preview handling, not just emitted linter warnings. The rule is off by default and normally tolerates versions up to 730 days old, with a 90-day grace period for new versions. Consequently, a missing warning does not mean no newer version is known. `outdated` must be able to report newer known versions even when the linter is disabled or the current version passes its age policy.

The linter's recommendations remain distinct from available-version reporting. Its settings do not enable Dependabot PRs, and extracting shared code must preserve existing linter behavior. **Open:** stable/preview display defaults, whether to expose the linter's filtered set as separately labeled context, and coverage of API versions in `reference()` and `list*()` calls, which the linter also analyzes.

The existing [`AzApiVersionProvider`](https://github.com/Azure/bicep/blob/2cef422d196e59ca959a411d3bd036035f37c7df/src/Bicep.Core/Analyzers/Linter/ApiVersions/AzApiVersionProvider.cs) reads Bicep's resource type catalog, not live subscription availability. Report that source and its freshness limits. A version known to Bicep is not proof that it is available in a user's target region or subscription, or compatible with their deployment. Unknown resource types and unresolved API expressions must not appear as successfully checked, up-to-date dependencies.

The current linter uses a semantic model. Shared discovery may need semantic analysis, but it must retain the read-only compiler-pin boundary and must not turn successful project compilation into an update-availability gate.

### Server side changes

No Bicep registry protocol or Azure service change is proposed. On the Dependabot side, adding an ecosystem directory is not sufficient: registration, packaging, and hosted-service enablement are separate steps requiring Dependabot maintainer coordination.

### Microsoft.Resources/deployments API changes

None.

### Examples

For a reference such as:

```bicep
module storage 'br/public:avm/res/storage/storage-account:0.6.0' = {
  name: 'storage'
  params: {
    name: 'example'
  }
}
```

The inventory records the resolved registry/repository, original tag `0.6.0`, and source occurrence. The candidate provider supplies versions from that source. Dependabot applies its configured rules and proposes a narrow tag edit. It does not update resource API strings inside the source or the compiled registry artifact.

If an equivalent alias resolves to an internal mirror, public-only v1 reports that source as unsupported. It must not query the public AVM index and recommend a version unavailable in the mirror.

If the project pins an older compiler, `bicep outdated` still reports supported references and leaves the pin unchanged. An available module update is not a claim that it works with that compiler.

### Scenarios for the remaining decisions

Source inspection of these repositories identified useful design and test cases:

| Observed scenario | Decision or invariant |
| --- | --- |
| [Two versions of the same module in one file](https://github.com/Azure-Samples/azure-search-openai-demo/blob/3f4a21f03ae3d565aca37cc300e3d38b0c7b582a/infra/main.bicep#L784-L868) | Preserve per-occurrence versions; no accidental downgrades. Updating does not automatically imply convergence to one version. |
| [Separately deployed application](https://github.com/Azure-Samples/Agentic-AI-Investment-Analysis-Sample/blob/7ca6abaeb5eb98f2690b9f92d619ffb3d25dea4d/api-app/infra/bicep/main.bicep#L92) | Check independently deployed files rather than assuming one `main.bicep`. |
| [Workload referencing a versioned vendored tree](https://github.com/Azure/avdaccelerator/blob/4ab656e10a8ee7eac861c235675c86e4d1abd7f4/workload/bicep/brownfield/addAvdAgents/deploy.bicep#L47-L50) | Honor Dependabot directory/exclusion controls; do not assume copied versioned trees should all be edited. |
| [Remote compile-time imports](https://github.com/microsoft/Build-your-own-copilot-Solution-Accelerator/blob/3e90f31bbcf687f523e980f8914a41b2255f24fa/infra/modules/ai-services.bicep#L67-L95) | Decide import update coverage explicitly. This retired repository is a possible fixed regression example, not a rollout partner. |
| Custom aliases and inherited configuration | Add targeted fixtures; the sampled built-in aliases do not prove coverage of custom alias/inheritance behavior. |

Check licenses before copying test fixtures, and use maintained repositories for rollout trials.

## Tradeoffs

- A supported JSON contract needs compatibility management, even though it avoids a supported compiler-library API. Purpose-built records limit exposure to internal compiler changes.
- Separate module and resource ecosystems add onboarding and documentation work, but preserve explicit enrollment using existing Dependabot semantics.
- Separate candidate-provider boundaries add an interface to design, but avoid committing private-registry support to one authentication stack before testing it.
- Public-only v1 gives limited coverage. Explicit unsupported-source reporting prevents that limitation from looking like a complete update check.
- Read-only analysis with a newer executable can identify updates without proving compatibility with the pinned compiler. Clear result semantics and project CI keep that limitation visible.

## Alternatives

### One ecosystem with new `dependency-type` values

Rejected under this contribution's constraints. Existing values are not arbitrary artifact categories; adding `module` and `resource-api` plus independent defaults would require shared Dependabot configuration/behavior work.

### Configure Dependabot category opt-in in `bicepconfig.json`

Rejected. Project configuration supplies source semantics and compiler constraints. Dependabot enrollment belongs in `dependabot.yml`. Enabling a linter diagnostic is not consent to dependency update PRs.

### One ecosystem with name-based allowlists or exclusions

Workable for users who maintain category-specific filters, but absent or broad filters would enable resource API updates once they entered the inventory. See **Scope and ecosystem enrollment** for why we require a separate opt-in.

### Implement all parsing and selection in Ruby

This follows parts of existing ecosystems but duplicates Bicep language/configuration knowledge and does not provide a native `outdated` command.

### Depend on the public Bicep Core package

Not selected as the supported integration boundary. Bicep's published compiler package does not promise a stable supported API. The JSON CLI contract permits internal code reuse without extending that promise.

### Delegate all registry access to Bicep immediately

Not settled. It offers reuse but must be compared against Dependabot-owned registry access before defining a contract that excludes the latter.

## Rollout plan

1. Agree on initial coverage, CLI and JSON design, registry lookup ownership, and editing support with Bicep and Dependabot maintainers.
2. Implement shared Bicep analysis, module lookup, and resource API reporting. Test file selection, JSON compatibility, compiler-pin handling, and room for future extension and compiler-constraint records.
3. Implement the Dependabot module ecosystem against the CLI interface. If native editing is included, pass exact targets and collect changed files. Test formatting preservation and per-reference edits.
4. Complete packaged-image validation, hosted-service onboarding, and preview requirements. Document unsupported cases and release metadata limitations.
5. Evaluate private registries for v2 using credential-proxy and curated-feed scenarios.

No ARM feature flag is required. Bicep CLI preview mechanics, first supported release, and Dependabot beta gates remain to be agreed. Separate golden fixtures should cover the protocol, reference updates, configuration inheritance, aliases/mirrors, unsupported cases, and deterministic behavior.

## Unresolved questions

### Before design approval

- Which public OCI repositories are supported in v1: AVM only, all indexed public Bicep modules, or a broader set? What is the authoritative candidate source and freshness contract?
- Does Dependabot fetch registry data itself or invoke Bicep's provider? What offline inventory/candidate-input surface is needed?
- What are the final command options, dependency-kind selector, schema envelope, and ecosystem identifier? What does a request without an explicit kind report?
- How does resource API output distinguish known newer versions from the linter's acceptable-version set, and which resource declarations and function calls can be checked?
- What minimal kind-specific fields allow future extension and compiler-constraint reporting without assuming every dependency is an OCI module or an exact version?
- Should Bicep or the Dependabot adapter apply version edits? If Bicep exposes editing, which kinds should v1 support, and through a separate command or an explicit write mode?
- What are the CLI defaults and how do they differ from Dependabot's configured rules? Resolve with scenarios, not assumptions about SemVer safety.
- How should Dependabot pass its selected file list to the CLI? Recommend checking only those files, without adding files they reference.
- When a user supplies a single file, should the command follow references outside its directory, and should a `.bicepparam` input include its template? What should it report for missing files, local redirects, and mock aliases?
- Which reference forms can the CLI check in v1, which can Dependabot edit, and how are different starting versions of one module handled without assuming convergence?

### Before stabilization

- Exact source-location/encoding contract, stale-edit guard, duplicate occurrences, deterministic ordering, and contract-version migration.
- Partial-result and exit-code semantics, unsupported-source reporting, and mapping to existing Dependabot error types.
- Publication-date provenance, cooldown behavior, and metadata/changelog coverage.
- Supported Bicep syntax and releases, CLI packaging, timeouts, cache freshness, and repository boundary enforcement.
- Evidence that read-only analysis never modifies compiler pins or silently replaces the configured source.

## Out of scope

- Dependabot resource API update PRs. Resource API availability reporting through the existing Bicep version lookup is in scope; the separate update policy and opt-in ecosystem will be developed later.
- Implicit file changes during discovery. Explicit native editing remains an open design question; creating update PRs remains the responsibility of a consumer such as Dependabot.
- Implementing extension or compiler-constraint discovery and updates, and choosing their eventual ecosystem boundaries. Accommodating those future kinds in the command and JSON design is in scope.
- Private registry implementation in v1; preserving the interfaces and identity required for v2 is in scope.
- Template Spec updates, module version-range updates, digest refreshes, and automatic dependency upgrades inside published artifacts.
- Updating arbitrary version-looking deployment values such as container images, VM images, script runtimes, or module parameters.
- Project compilation, tests, deployment validation, or automatic PR merging.
- Publishing a supported public Bicep .NET analysis API.

## References

- [Bicep REP process](https://github.com/Azure/bicep-reps/blob/main/README.md) and [current template](https://github.com/Azure/bicep-reps/blob/main/0000-rep-template.md).
- [Dependabot ecosystem contribution guide](https://github.com/dependabot/dependabot-core/blob/main/NEW_ECOSYSTEMS.md) and [existing dependency-type semantics](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#dependency-type-allow).
- [Module documentation command REP](https://github.com/Azure/bicep-reps/pull/25), a related example of Bicep-owned analysis with CLI consumers.
- [Marcin's external module reference proposal](https://github.com/Azure/bicep/issues/3186) and [package layout proposal](https://github.com/Azure/bicep/issues/3266), historical rationale for explicit versions and decoupling module consumption from its publishing compiler.
- [Resource API-version guidance](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/best-practices#api-version).

_Drafted by Copilot with GPT-6 Astra._
