---
REP Number: "Unassigned"
Author: andyleejordan (Andy Jordan)
Start Date: 2026-09-18
Feature Status: Private Preview
Bicep Issue Number(s): "[#8324](https://github.com/Azure/bicep/issues/8324)"
---

# Bicep update discovery and Dependabot integration

## Summary

Add two Bicep CLI operations. One reports newer versions of a project's dependencies, as a human-readable table or versioned JSON. The other moves one named dependency to an exact version everywhere the project references it. Bicep owns reading source and configuration, following local references, looking up versions, and editing files. What to call these operations, and whether they belong under a new command group, is an [open question](#command-shape-and-naming-open) for the Bicep Community Call.

Dependabot uses these operations to support Bicep through separate ecosystems, one per kind of dependency. The initial implementation covers two:

| Ecosystem | Updates |
| --- | --- |
| `bicep-artifacts` | Registry module and extension versions referenced from Bicep files and `bicepconfig.json` |
| `bicep-cli` | The `bicep.version` pin in `bicepconfig.json` |

The Dependabot team has agreed to separate ecosystems and prefers to ship a single `bicep` updater image that serves all of them. Resource API version updates, the subject of [#8324](https://github.com/Azure/bicep/issues/8324), are [deferred](#resource-api-versions-deferred) until we've researched how often and how severely an API version change breaks or changes a deployment.

## Terms and definitions

- **Artifact**: A module or extension published to an OCI registry and referenced by Bicep, following the `artifacts` terminology of [REP 0022](https://github.com/Azure/bicep-reps/blob/main/active/0022-artifact-aliases-and-redirects.md).
- **Dependency kind**: One of artifacts, the Bicep CLI, or (deferred) resources. Each kind has its own declaration site, version scheme, and source of versions.
- **Ecosystem**: A Dependabot `package-ecosystem` value that users enable in `dependabot.yml`. Each ecosystem has its own schedule, `allow`/`ignore` rules, grouping, and cooldown.
- **Updater image**: The container image Dependabot runs for an update job. One `bicep` image handles every Bicep ecosystem.
- **Entrypoint**: A Bicep or Bicep parameters file selected for analysis. Files it references can live outside its directory.
- **Occurrence**: One versioned reference in a source or configuration file. The same dependency can have several occurrences at different versions.
- **Effective configuration**: The configuration Bicep resolves for a source file from its nearest `bicepconfig.json` (or `--config`), any `extends` chain, and the built-in defaults, as described in [REP 0023](https://github.com/Azure/bicep-reps/blob/main/final/0023-bicep-configuration-inheritance.md) and [REP 0029](https://github.com/Azure/bicep-reps/blob/main/active/0029-cli-config-option.md).
- **Available version**: A version that exists at the dependency's source. Discovery reports what's available; it doesn't say whether updating is safe.

## Motivation

[dependabot/dependabot-core#15952](https://github.com/dependabot/dependabot-core/issues/15952) asks for Dependabot updates to Azure Verified Modules (AVM), and [#8324](https://github.com/Azure/bicep/issues/8324) has asked for resource API version updates since 2022. Bicep already has the [`use-recent-module-versions`](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/linter-rule-use-recent-module-versions) and [`use-recent-api-versions`](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/linter-rule-use-recent-api-versions) linter rules, but a warning with a "use latest" quick fix isn't an update tool. There's no way to list what's outdated across a project, or to move a dependency to a specific version.

Doing this outside Bicep means reimplementing Bicep. An artifact reference can use an alias, a mock, or a redirect. Configuration can be inherited through `extends` from a file in another directory, or replaced with `--config`. A local module can resolve a different configuration than the file that references it. A Ruby parser and registry client in Dependabot would have to duplicate all of that and keep up as the language changes. We built a proof of concept that way, using a C# helper over `Azure.Bicep.Core`. It updated a public AVM module tag end to end, but it relied on an unsupported library API and couldn't follow configuration inheritance. So Bicep should own discovery and editing, and the Dependabot updater should stay a thin adapter.

### Three kinds of dependency

A Bicep project depends on three different things:

| | Artifacts | Resources | Bicep CLI |
| --- | --- | --- | --- |
| Declared in | `module` and `extension` references in `.bicep` files; `extensions` in `bicepconfig.json` | `resource` declarations in `.bicep` files | `bicep.version` in `bicepconfig.json` |
| Version scheme | Usually SemVer tags | Dates, optionally with a suffix such as `-preview` | SemVer |
| Update cadence | Frequent | Rare and deliberate | Frequent |
| Source of versions | OCI registries, mostly the Microsoft Artifact Registry | Types built into each Bicep release | The Bicep CDN, which mirrors GitHub releases |
| Tracks | Published infrastructure code, such as AVM, and extension types | Azure Resource Manager (ARM) resource provider API contracts | The compiler |

Modules and extensions share one kind because they share everything that matters here: both are OCI artifacts with SemVer tags, resolved through the same aliases and registry code, and REP 0022 already treats them uniformly.

Artifacts and the CLI map to existing Dependabot patterns: Rust has separate [`cargo`](https://github.com/dependabot/dependabot-core/tree/main/cargo) and [`rust_toolchain`](https://github.com/dependabot/dependabot-core/tree/main/rust_toolchain) ecosystems, split along the same line between packages and the compiler.

Resources have no direct analog. They resemble versioned type packages, but languages that version types as packages (TypeScript's [`@types`](https://github.com/DefinitelyTyped/DefinitelyTyped) packages, Python's [typeshed](https://github.com/python/typeshed) stubs) do so because types were added after the fact. A Bicep resource's API version selects a resource provider's API contract, and its types are built into Bicep rather than downloaded. Changing that version can change deployment behavior, which is why the Bicep team has [cautioned against automated resource API updates](https://github.com/Azure/bicep/issues/8324#issuecomment-1577531181) and why `use-recent-api-versions` is off by default. If resource updates are ever offered, they must be a deliberate opt-in, never a side effect of enabling artifact updates.

Dependabot configuration has no ecosystem-specific options. With one Bicep ecosystem, users would need some Bicep-side setting to choose which kinds to update, which would put Dependabot configuration into `bicepconfig.json`. Separate ecosystems let each kind use Dependabot's native schedule, filters, grouping, and cooldown, and leave room to add resources later as their own opt-in.

### Goals

1. A native discovery experience that's useful without Dependabot.
2. An exact, explicit update; neither operation silently picks "latest".
3. One source of truth for references, configuration, version sources, and edits, so the Dependabot updater stays thin.
4. A versioned JSON contract between Bicep and Dependabot.
5. Reuse the version lookup and comparison behind Bicep's existing linter rules, rather than building parallel ones.

## Detailed design

### Client side changes

#### Command shape and naming (open)

The two operations need names, and the design review didn't settle them. Bicep isn't a package manager, and a top-level `bicep update` could easily be read as updating the Bicep CLI itself, the way [`az bicep upgrade`](https://learn.microsoft.com/en-us/cli/azure/bicep#az-bicep-upgrade) does. We'll take these options to the Bicep Community Call:

**Option A: a command group.** Both operations live under a dependency- or artifact-oriented group, such as `bicep dependencies outdated` and `bicep dependencies update`. This follows [`helm dependency list|update`](https://helm.sh/docs/helm/helm_dependency/) and [`dotnet package list|add`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-list). It reads unambiguously and leaves room for related operations later. The group's name is itself a question: "artifacts" matches the ecosystem but doesn't describe the CLI pin, "packages" suggests a package manager, and "dependencies" is long.

**Option B: top-level commands.** `bicep outdated` and `bicep update`, following [`npm outdated`](https://docs.npmjs.com/cli/v11/commands/npm-outdated/), [`npm update`](https://docs.npmjs.com/cli/v11/commands/npm-update/), and [`cargo update`](https://doc.rust-lang.org/cargo/commands/cargo-update.html). They're short and familiar to anyone who uses those tools, at the cost of the ambiguity above.

The verbs are open too. `outdated` and `update` are familiar package-management terms, but that doesn't make them the clearest terms for the Bicep CLI; for example, `list` might read better under a group.

The rest of this REP writes the discovery and update operations as `bicep outdated` and `bicep update` for readability only; every command line below is illustrative. Under any naming, their arguments, output, and JSON contract are the same.

#### Discovery operation

```text
bicep outdated <file> [--kind artifacts|cli] [--exclude <glob>]... [--config <path>] [--output-format human|json]
bicep outdated --pattern <glob> [--kind ...] [--exclude <glob>]... [--config <path>] [--output-format human|json]
```

The input is one or more entrypoints, following the `--pattern` convention of `build`, `lint`, `format`, and `restore`. `--kind` restricts the report to one kind; without it, every supported kind is reported. Automation should always pass `--kind`. `--exclude` (repeatable) names files that are read to follow the graph but never reported or edited, which is how Dependabot's `exclude-paths` reaches Bicep. `--config` behaves as in [REP 0029](https://github.com/Azure/bicep-reps/blob/main/active/0029-cli-config-option.md). The command makes no changes.

Human output groups occurrences by kind and shows only those with a newer version:

```text
$ bicep outdated infra/main.bicep
Artifacts
  Source                  Dependency                                                   Current        Latest
  shared/storage.bicep:1  br/public:avm/res/storage/storage-account                    0.6.0          0.33.1
  bicepconfig.json:3      br:mcr.microsoft.com/bicep/extensions/microsoftgraph/v1.0    0.1.8-preview  1.0.0

Bicep CLI
  Source                  Dependency                                                   Current        Latest
  build/bicepconfig.json  bicep                                                        0.50.0         0.51.0
```

**Dependency** is written the way it's written in source; add `:` or `@` and a version to pass it to `bicep update`. **Latest** is the newest stable version at the dependency's [source](#version-sources); when the current version is itself a prerelease, prereleases are considered too, so a preview extension can move to a newer preview or to a stable release. The table states what's available; it doesn't recommend updating. When nothing is newer, the command prints `No newer versions found.` rather than an empty table.

This shape follows [`npm outdated`](https://docs.npmjs.com/cli/v11/commands/npm-outdated/) and [`dotnet package list --outdated`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-list), which also offers a versioned JSON format for tools. npm's extra "Wanted" column is the newest version within the range in `package.json`; Bicep artifacts are exact pins, so there's no equivalent.

With `--output-format json`, the command writes an inventory to stdout. Dependencies are grouped by canonical identity, with every occurrence and every known version, so a caller can apply its own policy:

```json
{
  "schemaVersion": 1,
  "bicep": {
    "running": "0.51.0",
    "constraint": "0.50.0",
    "declaredIn": ["build/bicepconfig.json"],
    "satisfied": false,
    "resolved": "0.50.0",
    "downloadUrl": "https://downloads.bicep.azure.com/v0.50.0/bicep-linux-x64"
  },
  "dependencies": [
    {
      "kind": "artifacts",
      "type": "module",
      "name": "br:mcr.microsoft.com/bicep/avm/res/storage/storage-account",
      "versions": [
        { "version": "0.6.0", "publishedAt": "2024-02-09T08:45:33Z" },
        { "version": "0.6.1", "publishedAt": "2024-02-16T10:03:40Z" },
        { "version": "0.33.1", "publishedAt": "2026-09-16T08:17:53Z" }
      ],
      "occurrences": [
        {
          "location": { "file": "shared/storage.bicep", "line": 1, "column": 11 },
          "reference": "br/public:avm/res/storage/storage-account:0.6.0",
          "version": "0.6.0",
          "status": "checked"
        }
      ]
    },
    {
      "kind": "artifacts",
      "type": "extension",
      "name": "br:mcr.microsoft.com/bicep/extensions/microsoftgraph/v1.0",
      "versions": [
        { "version": "0.1.8-preview", "publishedAt": "2025-01-22T21:20:13Z" },
        { "version": "0.1.9-preview", "publishedAt": "2025-01-28T18:43:54Z" },
        { "version": "0.2.0-preview", "publishedAt": "2025-03-11T20:43:08Z" },
        { "version": "1.0.0", "publishedAt": "2026-01-08T19:57:28Z" }
      ],
      "occurrences": [
        {
          "location": { "file": "bicepconfig.json", "line": 3, "column": 25 },
          "reference": "br:mcr.microsoft.com/bicep/extensions/microsoftgraph/v1.0:0.1.8-preview",
          "version": "0.1.8-preview",
          "status": "checked"
        }
      ]
    },
    {
      "kind": "cli",
      "name": "bicep",
      "versions": [
        { "version": "0.50.0", "publishedAt": "2027-01-12T18:00:00Z" },
        { "version": "0.51.0", "publishedAt": "2027-02-09T18:00:00Z" }
      ],
      "occurrences": [
        {
          "location": { "file": "build/bicepconfig.json", "line": 1, "column": 25 },
          "reference": "0.50.0",
          "version": "0.50.0",
          "status": "checked"
        }
      ]
    }
  ],
  "errors": []
}
```

- `bicep` describes the running CLI and the effective `bicep.version` constraint across all reached files, used by the [bootstrap](#bicep-cli-bootstrap) below. `downloadUrl` is for the running platform, so the caller doesn't need to know the CDN's layout.
- `name` is the canonical identity: for artifacts, the fully qualified reference with aliases resolved through each occurrence's own effective configuration. It's the form automation should pass to `update`. `type` distinguishes modules from extensions.
- `versions` lists every version known at the source, with `publishedAt` where the source provides it, which Dependabot needs for [cooldowns](#release-dates-for-cooldowns). Dependabot uses this list to apply its own filters, as it does with other registries.
- Paths are relative to the current directory, and positions are 1-based. An occurrence declared in configuration, such as a CLI pin or an extension, is in the file that declares it, which can differ for each source file.
- An occurrence that can't be checked has `status: "unsupported"` with a `reason` (for example, `"non-comparable-tag"` or `"template-spec"`) instead of disappearing. A lookup failure has `status: "error"` and a matching `errors` entry with a location, a stable `code` such as `"registry-auth-failed"`, and a message; it makes the exit code nonzero. An empty `dependencies` array is a success.
- Diagnostics go to stderr. Neither stream contains credentials.

Artifact tags and dates in these examples are real, though the module's `versions` list is abbreviated. Bicep CLI versions are hypothetical future releases that include these operations. The JSON is a versioned CLI contract with its own `schemaVersion`, not an export of compiler objects. Bicep commits to keeping a schema version stable once it ships outside experimental, and adds new kinds or fields under the same version only when older callers can ignore them. The `bicep` block is part of every schema version, because a newer CLI uses it to hand off to an older one.

#### Update operation

```text
bicep update <file> <dependency> [--from <version>] [--exclude <glob>]... [--config <path>] [--output-format human|json]
bicep update --pattern <glob> <dependency> [--from <version>] [--exclude <glob>]... [--config <path>] [--output-format human|json]
```

`<dependency>` names a dependency and its target version in Bicep's own syntax:

```sh
bicep update infra/main.bicep 'br/public:avm/res/storage/storage-account:0.8.0'
bicep update infra/main.bicep 'br:mcr.microsoft.com/bicep/extensions/microsoftgraph/v1.0:1.0.0' --from 0.1.8-preview
bicep update infra/main.bicep 'bicep@0.51.0'
```

Artifacts use the `:tag` form of an artifact reference, and the CLI uses the reserved name `bicep`. That name can't be confused with an artifact, which always has a scheme or alias prefix.

`update` changes every non-excluded occurrence reachable from the entrypoints whose identity matches and whose current version is older than the target. That includes artifacts declared in configuration, so updating an extension edits both `extension` declarations in source and `extensions` entries in `bicepconfig.json`. Occurrences already at or past the target are left alone, so with `0.6.0` and `0.8.0` occurrences, updating to `0.8.0` moves only the first. An artifact given as an alias is resolved through the entrypoint's effective configuration, and each occurrence is compared through its own, so `br/public:...` and the equivalent `br:mcr.microsoft.com/...` both match and each keeps its own spelling. If an alias resolves to different registries in different parts of the graph, `update` rejects it as ambiguous and the caller passes the canonical `name` from `outdated` instead. `--from` narrows the change to occurrences currently at that version; Dependabot always passes it, so a pull request changes exactly the requirements it describes. For a CLI pin, the edit goes to the configuration file that declares `bicep.version`, which may be a base configuration reached through `extends`.

Only the version text changes; aliases, formatting, comments, and other references are preserved. Before writing anything, `update` checks that:

- at least one occurrence would change;
- the target exists at the dependency's source: a registry tag or a CDN release;
- the target compares with the current versions as SemVer, so a non-comparable tag is never "upgraded"; and
- a CLI pin is an exact version, not a range.

If any check fails, it exits nonzero and changes nothing; otherwise all edits are written together. It prints `Updated br/public:avm/res/storage/storage-account 0.6.0 -> 0.8.0 in shared/storage.bicep`, or with `--output-format json`, `{"schemaVersion":1,"changedFiles":["shared/storage.bicep"]}`.

`update` applies no selection policy of its own. Its caller has already chosen an exact version, whether a person reading `outdated` or Dependabot applying its own filters. This mirrors [`cargo update <spec> --precise <version>`](https://doc.rust-lang.org/cargo/commands/cargo-update.html), [`go get <module>@<version>`](https://go.dev/ref/mod#go-get), [`npm install <package>@<version>`](https://docs.npmjs.com/cli/v11/commands/npm-install/), and [`dotnet package add <package> --version <version>`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-add). It's also Dependabot's own model: one dependency moves to one version across every file that requires it.

#### Following the source and configuration graph

`bicep outdated main.bicep` follows the local references it reaches, including local targets selected through aliases or redirects. For each reached file, Bicep resolves its own effective configuration, including an `extends` chain that can leave the entrypoint's directory. Bicep also tracks which file declares each setting, which is how `update` knows to edit a base configuration rather than add an override to the leaf.

Traversal covers local source and configuration only, not the contents of published artifacts. A reference to a file outside the repository can work locally, but a hosted Dependabot job can't read it; the job must report that it couldn't analyze the graph rather than return a partial inventory.

#### Version sources

**Artifacts.** Bicep's registry code lists tags in the resolved repository, using the configured registry and credentials. Never assume a public index: an alias named `public` can point to a mirror. Tags that compare as SemVer are candidates; other tags make the occurrence unsupported. This is the comparison [`use-recent-module-versions`](https://github.com/Azure/bicep/pull/14309) already uses, but that rule reads a cached index of public modules, so we'll reuse its comparison with live tags rather than its lookup. A reference that a mock or [redirect](https://github.com/Azure/bicep-reps/blob/main/active/0022-artifact-aliases-and-redirects.md) resolves to a local file gets no registry versions; Bicep follows the local file instead. Template Specs and digest references are reported as unsupported.

Private registries rely on Dependabot's existing registry credentials and proxy. That needs an end-to-end test in the packaged updater before we claim support. An authentication failure is an error, never a fallback to a public registry.

**Bicep CLI.** Versions come from the CDN's release list at `https://downloads.bicep.azure.com/releases`, which mirrors GitHub's release objects including `published_at`. This is the same host that `BicepInstaller` in `Azure.Bicep.RpcClient` already uses.

#### Release dates for cooldowns

Dependabot's [`cooldown`](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#cooldown--) waits until a version has been published for a set number of days, so it needs each version's release date. When a date is missing, Dependabot logs that cooldown wasn't applied and proposes the update anyway, so implementation must verify that `publishedAt` is actually populated:

- **CLI releases**: the CDN's release list includes `published_at`.
- **Artifacts**: `bicep publish` and `bicep publish-extension` both write an `org.opencontainers.image.created` annotation to the artifact's manifest. Reading it costs one manifest request per newer tag, so implementation should measure that against large repositories. Artifacts published by other tools may not have the annotation; their versions are reported without `publishedAt`.

#### Bicep CLI bootstrap

The updater always starts with the latest Bicep CLI from the CDN, then switches to the project's pinned version when there is one. This happens in the updater's file parser step, described under [Dependabot integration](#dependabot-integration):

1. Run the latest CLI's `bicep outdated --kind <kind> --output-format json` on the entrypoints. Its `bicep` block reports the effective [`bicep.version`](https://github.com/Azure/bicep-reps/blob/main/active/0000-bicep-version-pinning.md) constraint across every reached file, whether the running CLI satisfies it, and the newest CDN release that does. Resolving the pin reads configuration only, and `outdated` reports an unsatisfied pin in this block rather than failing on it, so the latest CLI can always get this far.
2. If the running CLI satisfies the constraint, or there's no constraint, use its results.
3. Otherwise, if the pinned release has these operations, download `downloadUrl` and rerun with that version. The download logic is the same as `BicepInstaller`'s, and the `outdated` output is how the CLI shares it with an updater that isn't written in .NET.
4. If the pinned release is too old to have these operations, see [pins older than these operations](#pins-older-than-these-operations-open).
5. If reached files declare constraints no single release satisfies, the job fails with that conflict rather than guessing.

Running the pinned version means the project is read by the compiler it was written for, and that `CompilerVersionValidator` doesn't reject the project for running the wrong compiler. Unlike the entrypoint-only lookup that the version pinning REP describes for AzCLI, step 1 considers every reached file. The `bicep-cli` ecosystem itself always uses the latest CLI, since its job is to find newer releases.

#### Pins older than these operations (open)

A repository can pin a Bicep release that predates these operations, so step 3 has no pinned CLI to switch to. The design review leaned toward letting Dependabot fall back to the latest CLI with a warning, rather than failing the job, so that the dependency pull request can still be generated. This needs confirming before the REP is approved.

The case for falling back: the projects with the oldest pins are often the ones that most need updates, and a failed job leaves them none. In the initial scope, a newer CLI sees the same artifact tags and CDN releases a pinned one would, because neither depends on the compiler's version; that wouldn't hold for resources, whose versions come from each release's type catalog. The risk is that a newer CLI reads the project differently from the pinned compiler, for example by resolving configuration under newer rules. The warning would appear in the job log, and in the pull request if Dependabot supports surfacing it there. The alternative is to fail with a clear error naming the minimum supported pin.

#### Resource API versions (deferred)

Resource API version updates aren't part of the initial implementation. Before enabling a `bicep-resources` ecosystem, we need more research on how often an API version change breaks a deployment or changes its behavior, and how severe those changes are. That research should also inform the policy question this REP previously left open: whether to propose only the versions [`use-recent-api-versions`](https://github.com/Azure/bicep/pull/7612) would suggest (with its [730-day maximum age](https://github.com/Azure/bicep/pull/10344) and [90-day grace period](https://github.com/Azure/bicep/pull/19205)), or every newer version once someone has opted in.

The design leaves room for resources without committing to them. They'd be a new `--kind` and a new ecosystem served by the same updater image, with versions from the type catalog of the running Bicep release. Two findings carry forward:

- **Preview forms.** ARM API versions aren't marked prerelease only by `-preview`. Bicep's `AzureResourceApiVersion` parser already recognizes `-preview`, `-alpha`, `-beta`, `-rc`, and `-privatepreview`, case-insensitively. The shared comparison code extracted now must keep that behavior, and any future Dependabot version class for resources must treat every one of those as a prerelease and compare versions as dates, never as SemVer.
- **Release dates.** An API version's date is its name, not when it became usable; a version can appear in Bicep's type catalog months later. Cooldowns would need the date of the first Bicep release whose catalog included it.

### Dependabot integration

No new `dependabot.yml` keys or job fields are needed. Each kind is enabled on its own:

```yaml
version: 2
updates:
  - package-ecosystem: "bicep-artifacts"
    directory: "/infra"
    schedule:
      interval: "weekly"
  - package-ecosystem: "bicep-cli"
    directory: "/infra"
    schedule:
      interval: "monthly"
```

A single `bicep` updater image handles every Bicep `package-ecosystem` value and routes them internally, sharing the CLI bootstrap, process runner, and JSON adapter. Following [Dependabot's new ecosystem guide](https://github.com/dependabot/dependabot-core/blob/main/NEW_ECOSYSTEMS.md):

- **File fetcher** clones the repository, as several existing ecosystems do, so Bicep can follow references and `extends` chains without the updater parsing them first. The entrypoints are the `.bicep` and `.bicepparam` files directly in `directory`; `directories` adds more. They select where analysis starts, not the edge of the graph.
- **File parser** downloads the latest CLI from the CDN and runs the [bootstrap](#bicep-cli-bootstrap): `bicep outdated --kind <kind>` with `--exclude` for each `exclude-paths` entry, then a rerun with the pinned CLI if the `bicep` block calls for one. It records which CLI produced the final results and maps each dependency to a Dependabot dependency with one requirement per occurrence, carrying `publishedAt` into Dependabot's release metadata. `bicep-cli` jobs skip the switch and always use the latest CLI.
- **Update checker** selects a target version from `versions`, as below.
- **File updater** runs `bicep update` with the same CLI the parser settled on, passing the canonical `name`, the target, `--from`, and the same exclusions, and returns the changed files.

| Ecosystem | Version selection |
| --- | --- |
| `bicep-artifacts` | All `versions`, filtered by `allow`/`ignore` (including SemVer update types), prerelease handling, and cooldown where `publishedAt` is known. With no filters, the newest version, matching the module linter. If the newest is ignored, an older permitted version can still be proposed. |
| `bicep-cli` | CDN releases for exact pins, with cooldown from `published_at`. Range constraints are reported but not edited. |

Files reached from an entrypoint are eligible for updates even if they're outside `directory`, unless excluded with `exclude-paths`; excluded files are still read for analysis. This matches how the Terraform and Cargo ecosystems treat reachable local files. A dependency reached from several entrypoints is proposed once. A reference that leaves the repository fails the job, as described under [graph traversal](#following-the-source-and-configuration-graph).

Before the Bicep operations are merged and published in a release, the integration is validated with a dependabot-core pull request and a matching [smoke test](https://github.com/dependabot/smoke-tests) pull request, run against a prerelease build of Bicep. The tests must show that release dates reach Dependabot and that cooldowns hold back recent versions in both ecosystems. That way the JSON contract, command behavior, and cooldown metadata are proven against a real updater while they can still change.

### Server side changes

None. No new Bicep registry protocol or Azure service is needed. Hosted Dependabot enablement is coordinated with the Dependabot team separately from merging the updater.

### Microsoft.Resources/deployments API changes

None.

### Examples

Consider this repository:

```text
bicepconfig.json          { "extends": "./build/bicepconfig.json" }
build/bicepconfig.json    { "bicep": { "version": "0.50.0" } }
infra/main.bicep          module storage '../shared/storage.bicep' = { ... }
shared/storage.bicep
```

`shared/storage.bicep` contains:

```bicep
module sa 'br/public:avm/res/storage/storage-account:0.6.0' = {
  name: 'sa'
}

resource legacy 'Microsoft.Storage/storageAccounts@2025-01-01' existing = {
  name: 'legacy'
}
```

With the `dependabot.yml` above:

1. The `bicep-artifacts` job starts with the latest CLI, finds the `0.50.0` pin through the root configuration's `extends`, downloads `0.50.0`, and reruns `outdated` with it. If the pin were older than the first release with these operations, this step would follow whichever [old-pin behavior](#pins-older-than-these-operations-open) is confirmed.
2. `outdated` follows `infra/main.bicep` to `shared/storage.bicep`, outside `/infra`, and reports every newer tag through `0.33.1`, each with its publication date.
3. Dependabot selects `0.33.1` and runs `bicep update infra/main.bicep 'br:mcr.microsoft.com/bicep/avm/res/storage/storage-account:0.33.1' --from 0.6.0`. Only the tag in `shared/storage.bicep` changes, and it keeps its `br/public:` spelling.
4. The `bicep-cli` job proposes `bicep@0.51.0`, editing `build/bicepconfig.json`, where the pin is declared.
5. The resource API version is untouched; resource updates aren't part of this proposal.

Without Dependabot, a person can run `bicep outdated infra/main.bicep` to see the same table and `bicep update` to apply a version they choose.

## Tradeoffs

- **Separate ecosystems mean separate opt-ins.** Users enable and document each one. This is what keeps compiler or (eventually) resource changes from appearing in an existing artifact job, and the single updater image keeps the implementation cost close to one ecosystem.
- **One edit can reach outside `directory`.** Following the graph can update a shared file used by several deployments. Pull requests show every changed file, and `exclude-paths` keeps files read-only.
- **The JSON output becomes a compatibility commitment.** Bicep must version and maintain it. In exchange, Dependabot doesn't need a Ruby Bicep parser or an unsupported compiler library API.
- **The bootstrap costs a download per job and depends on the CDN.** It's needed for the analysis to run on the project's pinned compiler, and very old pins still need a [defined behavior](#pins-older-than-these-operations-open).
- **Deferring resources leaves [#8324](https://github.com/Azure/bicep/issues/8324) open.** Its long-standing request goes unanswered for now, in exchange for not automating changes whose deployment risk we haven't measured.

## Alternatives

**One ecosystem with Bicep-side settings.** Dependabot has no ecosystem-specific configuration, so selecting kinds would need a Dependabot section in `bicepconfig.json`. That puts one tool's configuration in another's file, and the kinds would still differ in version scheme, source, and cadence underneath one name.

**Separate ecosystems for modules and extensions.** Both are OCI artifacts with SemVer tags, resolved through the same aliases and registry code, and REP 0022 already treats them as one concept. Splitting them would double the opt-ins without any difference in how updates work.

**One updater image per ecosystem.** Earlier drafts assumed each ecosystem would be its own updater. The Dependabot team prefers one image because each updater carries significant packaging and onboarding overhead, and the ecosystems would share nearly all of it.

**Parse and edit Bicep in the updater.** This is how the proof of concept worked. It duplicates alias, redirect, mock, and configuration resolution, and depends on compiler internals with no compatibility promise. It also leaves people without a native discovery command.

**Occurrence-scoped `update`.** An earlier draft had `update` take a file, line, column, and expected reference. That's exact, but awkward to type, requires rediscovery after every edit, and doesn't match how Dependabot or other package managers model an update. `--from` provides the narrowing that matters.

**Return edits from `outdated`.** `outdated` could return text edits for each candidate and let the updater apply them. This removes a command but leaves people without an update command, and puts file writes back in Ruby.

**Have Bicep switch versions itself.** Go's [`toolchain` directive](https://go.dev/doc/toolchain) and rustup's [`rust-toolchain.toml`](https://rust-lang.github.io/rustup/overrides.html#the-toolchain-file) download and run the pinned compiler automatically. Bicep could do the same for every command, which would help CI and AzCLI as well as Dependabot. That's a broader change to the version pinning design and isn't needed here. If Bicep gains it later, steps 2 and 3 of the bootstrap collapse into Bicep.

## Rollout plan

1. Settle the command shape and names after the Bicep Community Call.
2. Extract the linters' version lookup and comparison into shared code without changing linter results. This preserves the existing handling of every ARM API version suffix, so a future resource design can build on it; no resource support ships in this rollout.
3. Implement the operations for artifacts and CLI pins, marked experimental in CLI help until the JSON schema stabilizes. Test unsupported references, lookup errors, inherited pins and extension declarations, release dates, and stale `--from` values.
4. Add the `bicep` updater image to dependabot-core with the `bicep-artifacts` and `bicep-cli` ecosystems, run against a prerelease Bicep build. Open the matching smoke test pull request, including cooldown cases for both ecosystems, and validate end to end against public AVM consumers, such as [Azure/alz-bicep-accelerator](https://github.com/Azure/alz-bicep-accelerator) and [Azure-Samples/azure-search-openai-demo](https://github.com/Azure-Samples/azure-search-openai-demo), then against a private registry before claiming support.
5. Merge and release the Bicep operations, then merge the updater and enable it in hosted Dependabot with the Dependabot team.
6. Research resource API version changes separately; any `bicep-resources` ecosystem gets its own design review.

No ARM feature flag or deployment API change is needed. Discovery doesn't deploy anything, and a proposed version isn't proof that a deployment will succeed with it.

## Unresolved questions

To resolve through this REP before merging:

- **Command shape and naming**: A command group (Option A) or top-level commands (Option B), and whether `outdated` and `update` are the right verbs. Pending feedback from the Bicep Community Call; see [command shape and naming](#command-shape-and-naming-open).
- **Old pins**: Confirm that Dependabot falls back to the latest CLI with a warning, as the design review leaned, rather than failing; see [pins older than these operations](#pins-older-than-these-operations-open).
- **Reachable files**: Confirm with Dependabot maintainers that updates to reachable files outside `directory` are acceptable for hosted jobs.

To resolve through implementation before stabilization:

- **Release dates**: Verify that `publishedAt` reaches Dependabot's cooldown logic for both ecosystems, as a gate in the integration test pull request, and measure the cost of reading artifact manifests on large repositories.
- **Initial reference coverage**: Artifact versions appear in `module` declarations, compile-time `import ... from 'br:...'`, `.bicepparam` `using 'br:...'`, `extension` declarations, and `extensions` entries in `bicepconfig.json`. The first preview needn't handle all of them; anything not yet handled is reported as `unsupported`, not skipped.

## Out of scope

- Resource API version updates, [deferred](#resource-api-versions-deferred) to a future investigation and design review.
- Dependencies inside published artifacts, Template Spec updates, and digest refreshes.
- Editing `bicep.version` ranges, and range constraints for artifacts.
- Automatic compilation, deployment validation, or merging.
- Bicep automatically switching to a pinned version for all commands.

_Drafted by Copilot with GPT-6 Sol and Claude Opus 5.5._
