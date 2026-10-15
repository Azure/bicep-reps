---
REP Number: "Unassigned"
Author: andyleejordan (Andy Jordan)
Start Date: 2026-09-18
Feature Status: Private Preview
Bicep Issue Number(s): "[#8324](https://github.com/Azure/bicep/issues/8324)"
---

# Bicep update discovery and Dependabot integration

## Summary

Add two Bicep CLI commands. `bicep outdated` reports newer versions of a project's dependencies, as a human-readable table or versioned JSON. `bicep update` moves one named dependency to an exact version everywhere the project references it. Bicep owns reading source and configuration, following local references, looking up versions, and editing files.

Dependabot uses these commands to support Bicep through three ecosystems, one per kind of dependency:

| Ecosystem | Updates |
| --- | --- |
| `bicep-modules` | Registry module versions referenced from Bicep files |
| `bicep-resources` | Resource API versions in Bicep files |
| `bicep-cli` | The `bicep.version` pin in `bicepconfig.json` |

The Dependabot team has agreed to this split and prefers to ship a single `bicep` updater image that serves all three. One design question remains open for this review: when should the resource ecosystem propose an API version update? It could follow the same rules as the `use-recent-api-versions` linter rule, or propose every newer version once someone has opted in.

## Terms and definitions

- **Dependency kind**: One of modules, resources, or the Bicep CLI. Each kind has its own declaration site, version scheme, and source of versions.
- **Ecosystem**: A Dependabot `package-ecosystem` value that users enable in `dependabot.yml`. Each ecosystem has its own schedule, `allow`/`ignore` rules, grouping, and cooldown.
- **Updater image**: The container image Dependabot runs for an update job. One `bicep` image handles all three Bicep ecosystems.
- **Entrypoint**: A Bicep or Bicep parameters file selected for analysis. Files it references can live outside its directory.
- **Occurrence**: One versioned reference in a source or configuration file. The same dependency can have several occurrences at different versions.
- **Effective configuration**: The configuration Bicep resolves for a source file from its nearest `bicepconfig.json` (or `--config`), any `extends` chain, and the built-in defaults, as described in [REP 0023](https://github.com/Azure/bicep-reps/blob/main/final/0023-bicep-configuration-inheritance.md) and [REP 0029](https://github.com/Azure/bicep-reps/blob/main/active/0029-cli-config-option.md).
- **Available version**: A version that exists at the dependency's source. `outdated` reports what's available; it doesn't say whether updating is safe.

## Motivation

[dependabot/dependabot-core#15952](https://github.com/dependabot/dependabot-core/issues/15952) asks for Dependabot updates to Azure Verified Modules (AVM), and [#8324](https://github.com/Azure/bicep/issues/8324) has asked for resource API version updates since 2022. Bicep already has the [`use-recent-module-versions`](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/linter-rule-use-recent-module-versions) and [`use-recent-api-versions`](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/linter-rule-use-recent-api-versions) linter rules, but a warning with a "use latest" quick fix isn't an update tool. There's no way to list what's outdated across a project, or to move a dependency to a specific version.

Doing this outside Bicep means reimplementing Bicep. A module reference can use an alias, a mock, or a redirect. Configuration can be inherited through `extends` from a file in another directory, or replaced with `--config`. A local module can resolve a different configuration than the file that references it. A Ruby parser and registry client in Dependabot would have to duplicate all of that and keep up as the language changes. We built a proof of concept that way, using a C# helper over `Azure.Bicep.Core`. It updated a public AVM module tag end to end, but it relied on an unsupported library API and couldn't follow configuration inheritance. So Bicep should own discovery and editing, and the Dependabot updater should stay a thin adapter.

### Three kinds of dependency

A Bicep project depends on three different things:

| | Modules | Resources | Bicep CLI |
| --- | --- | --- | --- |
| Declared in | `module` references in `.bicep` files | `resource` declarations in `.bicep` files | `bicep.version` in `bicepconfig.json` |
| Version scheme | Usually SemVer tags | Dates, optionally `-preview` | SemVer |
| Update cadence | Frequent | Rare and deliberate | Frequent |
| Source of versions | OCI registries, mostly the Microsoft Artifact Registry | Types built into each Bicep release | The Bicep CDN, which mirrors GitHub releases |
| Tracks | Published infrastructure code, such as AVM | Azure Resource Manager (ARM) resource provider API contracts | The compiler |

Modules and the CLI map to existing Dependabot patterns: Rust has separate [`cargo`](https://github.com/dependabot/dependabot-core/tree/main/cargo) and [`rust_toolchain`](https://github.com/dependabot/dependabot-core/tree/main/rust_toolchain) ecosystems, split along the same line between packages and the compiler.

Resources have no direct analog. They resemble versioned type packages, but languages that version types as packages (TypeScript's [`@types`](https://github.com/DefinitelyTyped/DefinitelyTyped) packages, Python's [typeshed](https://github.com/python/typeshed) stubs) do so because types were added after the fact. A Bicep resource's API version selects a resource provider's API contract, and its types are built into Bicep rather than downloaded. Changing that version can change deployment behavior, which is why the Bicep team has [cautioned against automated resource API updates](https://github.com/Azure/bicep/issues/8324#issuecomment-1577531181) and why `use-recent-api-versions` is off by default. Resource updates can still be useful, but only as a deliberate opt-in, never as a side effect of enabling module updates.

Dependabot configuration has no ecosystem-specific options. With one Bicep ecosystem, users would need some Bicep-side setting to choose which kinds to update, which would put Dependabot configuration into `bicepconfig.json`. Separate ecosystems let each kind use Dependabot's native schedule, filters, grouping, and cooldown, and keep the resource opt-in explicit.

### Goals

1. A native `outdated` experience that's useful without Dependabot.
2. An exact, explicit `update`; neither command silently picks "latest".
3. One source of truth for references, configuration, version sources, and edits, so the Dependabot updater stays thin.
4. Reuse the version lookup and comparison behind Bicep's existing linter rules, rather than building parallel ones.

## Detailed design

### Client side changes

#### `bicep outdated`

```text
bicep outdated <file> [--kind modules|resources|cli] [--exclude <glob>]... [--config <path>] [--output-format human|json]
bicep outdated --pattern <glob> [--kind ...] [--exclude <glob>]... [--config <path>] [--output-format human|json]
```

The input is one or more entrypoints, following the `--pattern` convention of `build`, `lint`, `format`, and `restore`. `--kind` restricts the report to one kind; without it, all three are reported. Automation should always pass `--kind`. `--exclude` (repeatable) names files that are read to follow the graph but never reported or edited, which is how Dependabot's `exclude-paths` reaches Bicep. `--config` behaves as in [REP 0029](https://github.com/Azure/bicep-reps/blob/main/active/0029-cli-config-option.md). The command makes no changes.

Human output groups occurrences by kind and shows only those with a newer version:

```text
$ bicep outdated infra/main.bicep
Modules
  Source                  Dependency                                  Current     Latest
  shared/storage.bicep:1  br/public:avm/res/storage/storage-account   0.6.0       0.8.0

Resources
  Source                  Dependency                                  Current     Latest
  infra/main.bicep:12     Microsoft.KeyVault/vaults                   2022-07-01  2025-05-01
  shared/storage.bicep:5  Microsoft.Storage/storageAccounts           2025-01-01  2025-06-01

Bicep CLI
  Source                  Dependency                                  Current     Latest
  build/bicepconfig.json  bicep                                       0.47.16     0.48.1
```

**Dependency** is written the way it's written in source; add `:` or `@` and a version to pass it to `bicep update`. **Latest** is the newest stable version at the dependency's [source](#version-sources), or the newest prerelease (including `-preview` API versions) when the current version is itself a prerelease. The table states what's available; it doesn't recommend updating. When nothing is newer, the command prints `No newer versions found.` rather than an empty table.

This shape follows [`npm outdated`](https://docs.npmjs.com/cli/v11/commands/npm-outdated/) and [`dotnet package list --outdated`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-list), which also offers a versioned JSON format for tools. npm's extra "Wanted" column is the newest version within the range in `package.json`; Bicep modules and resources are exact pins, so there's no equivalent.

With `--output-format json`, the command writes an inventory to stdout. Dependencies are grouped by canonical identity, with every occurrence and every known version, so a caller can apply its own policy:

```json
{
  "schemaVersion": 1,
  "bicep": {
    "running": "0.48.1",
    "constraint": "0.47.16",
    "declaredIn": ["build/bicepconfig.json"],
    "satisfied": false,
    "resolved": "0.47.16",
    "downloadUrl": "https://downloads.bicep.azure.com/v0.47.16/bicep-linux-x64"
  },
  "dependencies": [
    {
      "kind": "modules",
      "name": "br:mcr.microsoft.com/bicep/avm/res/storage/storage-account",
      "versions": [
        { "version": "0.6.0" },
        { "version": "0.7.0" },
        { "version": "0.8.0" }
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
      "kind": "resources",
      "name": "Microsoft.Storage/storageAccounts",
      "versions": [{ "version": "2025-01-01" }, { "version": "2025-06-01" }],
      "occurrences": [
        {
          "location": { "file": "shared/storage.bicep", "line": 5, "column": 17 },
          "reference": "Microsoft.Storage/storageAccounts@2025-01-01",
          "version": "2025-01-01",
          "status": "checked"
        }
      ]
    },
    {
      "kind": "cli",
      "name": "bicep",
      "versions": [
        { "version": "0.47.16", "publishedAt": "2026-09-08T18:04:56Z" },
        { "version": "0.48.1", "publishedAt": "2026-10-05T19:06:48Z" }
      ],
      "occurrences": [
        {
          "location": { "file": "build/bicepconfig.json", "line": 1, "column": 25 },
          "reference": "0.47.16",
          "version": "0.47.16",
          "status": "checked"
        }
      ]
    }
  ],
  "errors": []
}
```

- `bicep` describes the running CLI and the effective `bicep.version` constraint across all reached files, used by the [bootstrap](#bicep-cli-bootstrap) below. `downloadUrl` is for the running platform, so the caller doesn't need to know the CDN's layout.
- `name` is the canonical identity: for modules, the fully qualified reference with aliases resolved through each occurrence's own effective configuration. It's the form automation should pass to `update`.
- `versions` lists every version known at the source, with `publishedAt` where the source provides it. Dependabot uses this to apply its own filters, as it does with other registries.
- Paths are relative to the current directory, and positions are 1-based. A CLI pin's occurrence is in the file that declares it, which can differ for each source file.
- An occurrence that can't be checked has `status: "unsupported"` with a `reason` (for example, `"non-comparable-tag"` or `"template-spec"`) instead of disappearing. A lookup failure has `status: "error"` and a matching `errors` entry with a location, a stable `code` such as `"registry-auth-failed"`, and a message; it makes the exit code nonzero. An empty `dependencies` array is a success.
- Diagnostics go to stderr. Neither stream contains credentials.

The JSON is a CLI contract with its own `schemaVersion`, not an export of compiler objects. The `bicep` block is part of every schema version, because a newer CLI uses it to hand off to an older one.

#### `bicep update`

```text
bicep update <file> <dependency> [--from <version>] [--exclude <glob>]... [--config <path>] [--output-format human|json]
bicep update --pattern <glob> <dependency> [--from <version>] [--exclude <glob>]... [--config <path>] [--output-format human|json]
```

`<dependency>` names a dependency and its target version in Bicep's own syntax:

```sh
bicep update infra/main.bicep 'br/public:avm/res/storage/storage-account:0.8.0'
bicep update infra/main.bicep 'Microsoft.Storage/storageAccounts@2025-06-01' --from 2025-01-01
bicep update infra/main.bicep 'bicep@0.48.1'
```

Modules use the `:tag` form of a module reference, resources use the `Type@apiVersion` form of a resource declaration, and the CLI uses the reserved name `bicep`. That name can't be confused with the other two forms: resource types always contain `/`, and module references always have a scheme or alias prefix.

`update` changes every non-excluded occurrence reachable from the entrypoints whose identity matches and whose current version is older than the target. Occurrences already at or past the target are left alone, so with `0.6.0` and `0.7.0` occurrences, updating to `0.7.0` moves only the first. A module given as an alias is resolved through the entrypoint's effective configuration, and each occurrence is compared through its own, so `br/public:...` and the equivalent `br:mcr.microsoft.com/...` both match and each keeps its own spelling. If an alias resolves to different registries in different parts of the graph, `update` rejects it as ambiguous and the caller passes the canonical `name` from `outdated` instead. `--from` narrows the change to occurrences currently at that version; Dependabot always passes it, so a pull request changes exactly the requirements it describes. For a CLI pin, the edit goes to the configuration file that declares `bicep.version`, which may be a base configuration reached through `extends`.

Only the version text changes; aliases, formatting, comments, and other references are preserved. Before writing anything, `update` checks that:

- at least one occurrence would change;
- the target exists at the dependency's source: a registry tag, an API version in the running Bicep's type catalog, or a CDN release;
- the target compares with the current versions under that kind's rules (SemVer for modules and the CLI, dates with stable versus preview for resources), so a non-comparable tag is never "upgraded"; and
- a CLI pin is an exact version, not a range.

If any check fails, it exits nonzero and changes nothing; otherwise all edits are written together. It prints `Updated br/public:avm/res/storage/storage-account 0.6.0 -> 0.8.0 in shared/storage.bicep`, or with `--output-format json`, `{"schemaVersion":1,"changedFiles":["shared/storage.bicep"]}`.

`update` applies no selection policy of its own. Its caller has already chosen an exact version, whether a person reading `outdated` or Dependabot applying its own filters. This mirrors [`cargo update <spec> --precise <version>`](https://doc.rust-lang.org/cargo/commands/cargo-update.html), [`go get <module>@<version>`](https://go.dev/ref/mod#go-get), [`npm install <package>@<version>`](https://docs.npmjs.com/cli/v11/commands/npm-install/), and [`dotnet package add <package> --version <version>`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-add). It's also Dependabot's own model: one dependency moves to one version across every file that requires it.

#### Following the source and configuration graph

`bicep outdated main.bicep` follows the local references it reaches, including local targets selected through aliases or redirects. For each reached file, Bicep resolves its own effective configuration, including an `extends` chain that can leave the entrypoint's directory. Bicep also tracks which file declares each setting, which is how `update` knows to edit a base configuration rather than add an override to the leaf.

Traversal covers local source and configuration only, not the contents of published registry modules. A reference to a file outside the repository can work locally, but a hosted Dependabot job can't read it; the job must report that it couldn't analyze the graph rather than return a partial inventory.

#### Version sources

**Modules.** Bicep's registry code lists tags in the resolved repository, using the configured registry and credentials. Never assume a public index: an alias named `public` can point to a mirror. Tags that compare as SemVer are candidates; other tags make the occurrence unsupported. This is the comparison [`use-recent-module-versions`](https://github.com/Azure/bicep/pull/14309) already uses, but that rule reads a cached index of public modules, so we'll reuse its comparison with live tags rather than its lookup. A reference that a mock or [redirect](https://github.com/Azure/bicep-reps/blob/main/active/0022-artifact-aliases-and-redirects.md) resolves to a local file gets no registry versions; Bicep follows the local file instead. Template Specs and digest references are reported as unsupported.

Private registries rely on Dependabot's existing registry credentials and proxy. That needs an end-to-end test in the packaged updater before we claim support. An authentication failure is an error, never a fallback to a public registry.

**Resources.** Versions come from the type catalog built into the running Bicep release, the same lookup `use-recent-api-versions` uses. `outdated` reports every newer version in the catalog and doesn't apply that rule's age and grace-period logic; whether the resource ecosystem should is the [open question](#resource-update-policy-open) below. API versions are compared as dates, with `-preview` treated as a prerelease, never as SemVer. The catalog doesn't prove a version is available in a particular subscription or region, and an API version's date isn't its publication date.

Because the catalog ships with each Bicep release, which CLI runs the analysis matters. A newer CLI knows API versions that an older pinned compiler doesn't, which is one reason for the bootstrap below.

**Bicep CLI.** Versions come from the CDN's release list at `https://downloads.bicep.azure.com/releases`, which mirrors GitHub's release objects including `published_at`. This is the same host that `BicepInstaller` in `Azure.Bicep.RpcClient` already uses.

#### Resource update policy (open)

`bicep outdated` reports every newer API version either way. The open question is which of them the `bicep-resources` ecosystem proposes. There's no Dependabot setting that could let users choose, so this is a decision for the ecosystem.

**Option A: follow `use-recent-api-versions`.** Dependabot proposes only the version that rule would suggest for each occurrence, under that file's effective configuration and whether or not the diagnostic is enabled. Enabling the ecosystem gives exactly the updates the linter would warn about. The rule was [ported from the ARM TTK](https://github.com/Azure/bicep/pull/7612), with a [730-day default maximum age](https://github.com/Azure/bicep/pull/10344), a [90-day grace period](https://github.com/Azure/bicep/pull/19205), and its own stable versus preview handling; it can accept a current version even when newer ones exist. `outdated` would need to report that suggestion as a resource-only JSON field, added only if this option is chosen. This is conservative and consistent with the linter, and users who want more or fewer updates can tune `maxAgeInDays` in `bicepconfig.json`, which changes the linter in the same way. The cost: someone who has deliberately opted in will likely expect to see the newest API versions and won't, and the rule's thresholds were chosen for a warning rather than reviewed as an update policy.

**Option B: propose every newer version.** Dependabot proposes the newest version newer than each occurrence, filtered by `allow`/`ignore` (by resource type and version) and treating `-preview` as a prerelease. This matches what users generally expect from Dependabot once they've opted in, and it's simple to explain. The cost: many more pull requests, each of which can change deployment behavior, so safety depends on the opt-in documentation and on users scoping the ecosystem with `allow`.

Under either option the ecosystem is documented with an explicit warning about what resource API updates mean, and enabling `bicep-modules` never enables it.

#### Bicep CLI bootstrap

The updater always starts with the latest Bicep CLI from the CDN, then switches to the project's pinned version when there is one. This happens in the updater's file parser step, described under [Dependabot integration](#dependabot-integration):

1. Run the latest CLI's `bicep outdated --kind <kind> --output-format json` on the entrypoints. Its `bicep` block reports the effective [`bicep.version`](https://github.com/Azure/bicep-reps/blob/main/active/0000-bicep-version-pinning.md) constraint across every reached file, whether the running CLI satisfies it, and the newest CDN release that does. Resolving the pin reads configuration only, and `outdated` reports an unsatisfied pin in this block rather than failing on it, so the latest CLI can always get this far.
2. If the running CLI satisfies the constraint, or there's no constraint, use its results.
3. Otherwise, download `downloadUrl` and rerun with that version. The download logic is the same as `BicepInstaller`'s, and the `outdated` output is how the CLI shares it with an updater that isn't written in .NET.
4. If reached files declare constraints no single release satisfies, the job fails with that conflict rather than guessing.

Running the pinned version means resource versions come from the type catalog the project compiles with, and that `CompilerVersionValidator` doesn't reject the project for running the wrong compiler. Unlike the entrypoint-only lookup that the version pinning REP describes for AzCLI, step 1 considers every reached file. The `bicep-cli` ecosystem itself always uses the latest CLI, since its job is to find newer releases.

A pin older than the first release with `outdated` and `update` can't run step 3; see [unresolved questions](#unresolved-questions).

### Dependabot integration

No new `dependabot.yml` keys or job fields are needed. Each kind is enabled on its own:

```yaml
version: 2
updates:
  - package-ecosystem: "bicep-modules"
    directory: "/infra"
    schedule:
      interval: "weekly"
  - package-ecosystem: "bicep-cli"
    directory: "/infra"
    schedule:
      interval: "monthly"
```

A single `bicep` updater image handles all three `package-ecosystem` values and routes them internally, sharing the CLI bootstrap, process runner, and JSON adapter. Following [Dependabot's new ecosystem guide](https://github.com/dependabot/dependabot-core/blob/main/NEW_ECOSYSTEMS.md):

- **File fetcher** clones the repository, as several existing ecosystems do, so Bicep can follow references and `extends` chains without the updater parsing them first. The entrypoints are the `.bicep` and `.bicepparam` files directly in `directory`; `directories` adds more. They select where analysis starts, not the edge of the graph.
- **File parser** downloads the latest CLI from the CDN and runs the [bootstrap](#bicep-cli-bootstrap): `bicep outdated --kind <kind>` with `--exclude` for each `exclude-paths` entry, then a rerun with the pinned CLI if the `bicep` block calls for one. It records which CLI produced the final results and maps each dependency to a Dependabot dependency with one requirement per occurrence. `bicep-cli` jobs skip the switch and always use the latest CLI.
- **Update checker** selects a target version from `versions`, as below.
- **File updater** runs `bicep update` with the same CLI the parser settled on, passing the canonical `name`, the target, `--from`, and the same exclusions, and returns the changed files. Using the same CLI matters for resources, because a target API version must exist in that release's type catalog.

| Ecosystem | Version selection |
| --- | --- |
| `bicep-modules` | All `versions`, filtered by `allow`/`ignore` (including SemVer update types), prerelease handling, and cooldown where `publishedAt` is known. With no filters, the newest version, matching the module linter. If the newest is ignored, an older permitted version can still be proposed. |
| `bicep-resources` | Per the [open policy question](#resource-update-policy-open). Dates have no SemVer update types. |
| `bicep-cli` | CDN releases for exact pins, with cooldown from `published_at`. Range constraints are reported but not edited. |

Files reached from an entrypoint are eligible for updates even if they're outside `directory`, unless excluded with `exclude-paths`; excluded files are still read for analysis. This matches how the Terraform and Cargo ecosystems treat reachable local files. A dependency reached from several entrypoints is proposed once. A reference that leaves the repository fails the job, as described under [graph traversal](#following-the-source-and-configuration-graph).

### Server side changes

None. No new Bicep registry protocol or Azure service is needed. Hosted Dependabot enablement is coordinated with the Dependabot team separately from merging the updater.

### Microsoft.Resources/deployments API changes

None.

### Examples

Consider this repository:

```text
bicepconfig.json          { "extends": "./build/bicepconfig.json" }
build/bicepconfig.json    { "bicep": { "version": "0.47.16" } }
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

1. The `bicep-modules` job starts with the latest CLI, finds the `0.47.16` pin through the root configuration's `extends`, downloads `0.47.16`, and reruns `outdated` with it.
2. `outdated` follows `infra/main.bicep` to `shared/storage.bicep`, outside `/infra`, and reports `0.7.0` and `0.8.0` as newer versions.
3. Dependabot selects `0.8.0` and runs `bicep update infra/main.bicep 'br:mcr.microsoft.com/bicep/avm/res/storage/storage-account:0.8.0' --from 0.6.0`. Only the tag in `shared/storage.bicep` changes, and it keeps its `br/public:` spelling.
4. The `bicep-cli` job proposes `bicep@0.48.1`, editing `build/bicepconfig.json`, where the pin is declared.
5. The resource API version is untouched, because `bicep-resources` isn't enabled.

Without Dependabot, a person can run `bicep outdated infra/main.bicep` to see the same table and `bicep update` to apply a version they choose.

## Tradeoffs

- **Three ecosystems mean three opt-ins.** Users enable and document each separately. This is what keeps resource API changes from appearing in an existing module job, and the single updater image keeps the implementation cost close to one ecosystem.
- **One edit can reach outside `directory`.** Following the graph can update a shared file used by several deployments. Pull requests show every changed file, and `exclude-paths` keeps files read-only.
- **The JSON output becomes a compatibility commitment.** Bicep must version and maintain it. In exchange, Dependabot doesn't need a Ruby Bicep parser or an unsupported compiler library API.
- **The bootstrap costs a download per job and depends on the CDN.** It's needed for the analysis to match the pinned compiler. Projects pinned to a release older than these commands need a defined behavior.
- **Neither resource policy suits everyone.** Option A will feel too slow to some users who opted in; Option B will produce more updates than some want. Dependabot can't offer a per-repository switch between them.

## Alternatives

**One ecosystem with Bicep-side settings.** Dependabot has no ecosystem-specific configuration, so selecting kinds would need a Dependabot section in `bicepconfig.json`. That puts one tool's configuration in another's file, and the three kinds would still differ in version scheme, source, and cadence underneath one name.

**One updater image per ecosystem.** Earlier drafts assumed each ecosystem would be its own updater. The Dependabot team prefers one image because each updater carries significant packaging and onboarding overhead, and the three would share nearly all of it.

**Parse and edit Bicep in the updater.** This is how the proof of concept worked. It duplicates alias, redirect, mock, and configuration resolution, and depends on compiler internals with no compatibility promise. It also leaves people without a native `outdated` command.

**Occurrence-scoped `update`.** An earlier draft had `update` take a file, line, column, and expected reference. That's exact, but awkward to type, requires rediscovery after every edit, and doesn't match how Dependabot or other package managers model an update. `--from` provides the narrowing that matters.

**Return edits from `outdated`.** `outdated` could return text edits for each candidate and let the updater apply them. This removes a command but leaves people without an update command, and puts file writes back in Ruby.

**Have Bicep switch versions itself.** Go's [`toolchain` directive](https://go.dev/doc/toolchain) and rustup's [`rust-toolchain.toml`](https://rust-lang.github.io/rustup/overrides.html#the-toolchain-file) download and run the pinned compiler automatically. Bicep could do the same for every command, which would help CI and AzCLI as well as Dependabot. That's a broader change to the version pinning design and isn't needed here. If Bicep gains it later, steps 2 and 3 of the bootstrap collapse into Bicep.

## Rollout plan

1. Add `outdated` and `update` for modules and CLI pins, marked experimental in CLI help until the JSON schema stabilizes. Extract the linters' version lookup and comparison logic without changing linter results. Test that `outdated` compares versions the same way the linters do, including stable versus preview API versions. Also test unsupported references, lookup errors, inherited pins, and stale `--from` values.
2. Add the `bicep` updater image to dependabot-core with the `bicep-modules` ecosystem. Validate end to end against public AVM consumers, such as [Azure/alz-bicep-accelerator](https://github.com/Azure/alz-bicep-accelerator) and [Azure-Samples/azure-search-openai-demo](https://github.com/Azure-Samples/azure-search-openai-demo), then against a private registry before claiming support.
3. Add `bicep-cli` to the same image.
4. Add resource support to `outdated` and `update`, then `bicep-resources`, once the policy question is decided.

No ARM feature flag or deployment API change is needed. `outdated` doesn't deploy anything, and a proposed version isn't proof that a deployment will succeed with it.

## Unresolved questions

To resolve through this REP before merging:

- **Resource policy**: Option A (linter rules) or Option B (every newer version)?
- **Old pins**: If a project pins a CLI older than the first release with these commands, should the job fail with a clear error, or fall back to the latest CLI with a warning, knowing its resource catalog may not match?
- **Reachable files**: Confirm with Dependabot maintainers that updates to reachable files outside `directory` are acceptable for hosted jobs.

To resolve through implementation before stabilization:

- **Publication dates for cooldowns**: Dependabot's [`cooldown`](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#cooldown--) waits until a version has been published for a set number of days, so it needs each version's release date. When a date is missing, Dependabot logs that cooldown wasn't applied and proposes the update anyway. The three kinds differ:
  - CLI releases are covered: the CDN's release list includes `published_at`.
  - Modules are mostly covered: `bicep publish` writes an `org.opencontainers.image.created` annotation to each module's manifest. Reading it costs one manifest request per newer tag, so implementation should measure that against large repositories, and modules published by other tools may not have it.
  - Resources are open. An API version's date is its name, not when it became usable; a version can appear in Bicep's type catalog months later. The closest stand-in is the release date of the first Bicep release whose catalog included it, which needs that history recorded somewhere. Under Option A the grace period plays a similar role; under Option B, resource updates would have no cooldown until this is solved.
- **Initial reference coverage**: Versions appear in more places than `module` and `resource` declarations, and the first preview needn't handle all of them. Anything not yet handled is reported as `unsupported`, not skipped. The forms are:
  - Registry references: `module` declarations, compile-time `import ... from 'br:...'`, `.bicepparam` `using 'br:...'`, and `extension` declarations and `extensions` entries in `bicepconfig.json`. Extensions are OCI artifacts with SemVer tags like modules, but whether they belong in `bicep-modules` or need their own treatment is part of this question.
  - API versions: `resource` declarations, including `existing` and nested child resources with shortened type names; explicit API versions passed to functions like `reference()` and `list*()`, which `use-recent-api-versions` already checks; and resource-derived types such as `resourceInput<'Type@version'>`.

## Out of scope

- Dependencies inside published modules, Template Spec updates, and digest refreshes.
- Editing `bicep.version` ranges, and range constraints for modules.
- Automatic compilation, deployment validation, or merging.
- Bicep automatically switching to a pinned version for all commands.

_Drafted by Copilot with GPT-6 Sol and Claude Opus 5.5._
