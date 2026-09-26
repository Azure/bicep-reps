---
REP Number: "Unassigned"
Author: andyleejordan (Andy Jordan)
Start Date: 2026-09-18
Feature Status: Private Preview
---

# Bicep update discovery and Dependabot integration

## Summary

Add `bicep outdated` to find newer versions of Bicep dependencies, with human-readable and versioned JSON output, and `bicep update` to apply an explicitly selected version to one source occurrence. Bicep owns reference discovery, configuration resolution, candidate lookup, and source edits. Dependabot chooses versions under its existing policies and opens pull requests.

We propose three independently enabled Dependabot ecosystems for registry modules, resource API versions, and Bicep CLI version pins. Their names and separation need review with the Dependabot team. AVM motivates the first integration test, but the module ecosystem covers Bicep registry modules generally, not just AVM; private registry support needs end-to-end authentication validation before it can be claimed.

## Terms and definitions

- **Entrypoint**: A Bicep or Bicep parameters file selected for analysis. Its reachable local files can be outside the entrypoint's directory.
- **Occurrence**: One versioned reference in a source or configuration file. The same dependency can have several occurrences at different versions.
- **Effective configuration**: The settings Bicep resolves for a source file from its nearest `bicepconfig.json`, any inherited configuration files, and the built-in defaults.
- **Candidate**: An available version at the resolved source. Bicep's existing recommendation logic distinguishes candidates it would suggest by default; availability alone is not a claim that an update is safe.

## Motivation

[dependabot/dependabot-core#15952](https://github.com/dependabot/dependabot-core/issues/15952) requests Dependabot updates for public AVM modules. Bicep already has [`use-recent-module-versions`](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/linter-rule-use-recent-module-versions) and [`use-recent-api-versions`](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/linter-rule-use-recent-api-versions) linter rules, but warnings and "use latest" quick fixes are not an update-discovery or exact-target editing interface. A module reference may use a registry alias, a local redirect, or configuration inherited from another directory. Bicep source files can also refer to local modules whose own configuration differs from the entrypoint's. A Ruby parser or registry client would have to duplicate these Bicep-specific rules and keep up as the language changes.

The native commands should be useful without Dependabot: a person can see which versions are available, then request one exact edit. Neither `outdated` nor `update` silently selects "latest" in a Bicep source file. Dependabot selects its candidate pool, then applies its `allow`, `ignore`, grouping, cooldown, and prerelease rules before requesting an edit.

## Detailed design

### Client side changes: one native interface

The proposed command line is:

```sh
bicep outdated <entrypoint> [--kind module|resource-api|cli] [--selection recommended|available] [--output-format human|json] [--schema-version 1]
bicep update <file> --kind module|resource-api|cli --line <line> --column <column> --expected <current-reference> --version <exact-target> [--output-format human|json]
bicep update --request <json-file> [--output-format human|json]
```

`module`, `recommended`, and `human` are the defaults; the other kinds are explicit so adding them cannot change module-only results. `--schema-version` requires JSON and defaults to 1. Dependabot requests exactly one kind. `--selection recommended` proposes only versions Bicep would suggest for the current reference; `--selection available` shows the broader pool of comparable newer candidates (plus any linter-suggested stable replacement of a preview). People can explicitly request `available`; Dependabot uses the broader pool for SemVer module filtering, but never for resource API updates. Existing `bicepconfig.json` settings for the resource linter's maximum age and grace period still shape recommendations, even if the diagnostic is disabled. The CLI selection does not change the linter's policy or claim that every available version is safe.

`outdated` makes no source changes. In human mode, show a short table of **Source**, **Dependency**, **Current**, **Recommended**, and **Latest available**; identify the resolved registry for modules. Under `available`, add **Other newer** candidates, clearly marked as outside Bicep's recommendation. For example:

```text
Source                  Dependency                                                 Current  Recommended  Latest available
shared/storage.bicep:1  mcr.microsoft.com/bicep/avm/res/storage/storage-account    0.6.0    0.8.0        0.8.0
```

For `bicep outdated infra/main.bicep --selection available`, the same row is:

```text
Source                  Dependency                                                 Current  Recommended  Latest available  Other newer (not recommended)
shared/storage.bicep:1  mcr.microsoft.com/bicep/avm/res/storage/storage-account    0.6.0    0.8.0        0.8.0             0.7.0
```

The module linter suggests only the newest version. With no recommendations, print `No recommended module updates.` rather than an empty table; under `available`, print `No newer module versions found.` only if the broader candidate set is also empty. For resources, show the effective maximum age and grace period alongside the table. For example, if `2025-04-01` is still an acceptable resource API version and `2025-05-01` is in Bicep's catalog, `recommended` is empty while `available` includes `2025-05-01`: a newer version exists, but Bicep's reviewed policy does not yet suggest replacing the current one. The resource rule may also suggest a same-date stable version in place of a preview; preserve that recommendation rather than treating "newer date" as the only possible fix. For exact CLI pins, the default recommendation is the newest eligible CDN release; this is a release-selection rule, not a linter rule.

The JSON is an inventory, not just the rows shown in the default table. A representative module result:

```json
{
  "schemaVersion": 1,
  "kind": "module",
  "selection": "recommended",
  "occurrences": [
    {
      "location": { "file": "shared/storage.bicep", "line": 1, "column": 16 },
      "reference": "br/public:avm/res/storage/storage-account:0.6.0",
      "dependency": { "registry": "mcr.microsoft.com", "repository": "bicep/avm/res/storage/storage-account" },
      "currentVersion": "0.6.0",
      "status": "checked",
      "policy": { "source": "use-recent-module-versions" },
      "recommended": ["0.8.0"],
      "available": ["0.7.0", "0.8.0"]
    }
  ],
  "errors": []
}
```

Paths and 1-based line/column positions identify the original text to edit; the updater must also match `reference` before writing. Each occurrence is emitted once even if several entrypoints reach it. `recommended` contains only the versions the applicable Bicep policy would suggest for this **current** reference, not every version considered acceptable by a linter; it can be empty while `available` is nonempty. `available` contains comparable newer candidates from the resolved source as well as linter-suggested stable replacements; availability alone is not a claim of safety. JSON includes both lists in either selection mode, and `selection` tells an adapter which pool to use for proposals. The per-occurrence `policy` identifies the recommendation source; for resource API versions it also includes the effective `maxAgeInDays` and `gracePeriodInDays` from that file's configuration. A resource dependency identifies its resource type and scope and obtains candidates from Bicep's API catalog; a CLI pin identifies its declaring configuration file and CDN release source, and its `recommended` value comes from the CDN release-selection rule. A resolved local redirect has no remote module candidates but its local target is still traversed. Unsupported references have `status: "unsupported"` and a `reason`, for example `"non-comparable-tag"`, rather than disappearing. Lookup or graph failures have `status: "error"` and a corresponding `errors` entry with location, code, and message (for example, `"registry-auth-failed"`); any such failure gives a nonzero exit, even when other occurrences were checked. An empty successful `occurrences` array is not a failure. Machine results go to stdout, diagnostics to stderr; neither stream includes credentials.

`bicep update` takes one occurrence, the exact `reference` returned by discovery, and an exact target chosen by its caller. For example, `bicep update shared/storage.bicep --kind module --line 1 --column 16 --expected 'br/public:avm/res/storage/storage-account:0.6.0' --version 0.8.0` prints `Updated shared/storage.bicep:1:16: 0.6.0 -> 0.8.0`. Dependabot can send the same values in a request file:

```json
{
  "schemaVersion": 1,
  "kind": "module",
  "location": { "file": "shared/storage.bicep", "line": 1, "column": 16 },
  "expected": "br/public:avm/res/storage/storage-account:0.6.0",
  "version": "0.8.0"
}
```

With `--output-format json`, a successful update writes `{"schemaVersion":1,"changedFiles":["shared/storage.bicep"]}` to stdout. Both forms verify the source, location, current reference, target's validity for that dependency, and the single exact edit before writing; a stale location or ambiguous match is a nonzero error on stderr and changes no file. Preserve aliases, formatting, comments, and unrelated versions. For a pin inherited from another configuration file, the `location` is the declaration's actual file, not the entrypoint. Dependabot reruns discovery after edits that could move later occurrences in the same file. `update` does not enforce the `outdated` selection policy: its caller explicitly selects a valid exact target.

This discover-then-explicit-update design follows [.NET's `dotnet package list --outdated --format json --output-version 1`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-list) and its explicit [`dotnet package add ... --version`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-add); [.NET's `--include-prerelease`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-list) also makes broadened discovery opt-in. [`npm outdated`](https://docs.npmjs.com/cli/v11/commands/npm-outdated/) shows current and latest versions in a concise table and supports JSON, but `npm update` is not a model for editing one Bicep occurrence. Bicep's existing commands accept a positional source path, use `--stdout` for generated artifacts, and keep errors off stdout; `--output-format` here selects a report format, not a generated file destination. This versioned JSON boundary is a Bicep CLI contract, not an export of compiler syntax-tree objects.

### Following the source and configuration graph

`bicep outdated main.bicep` follows its local Bicep references, including local implementations selected through aliases or redirects. For each reached source file, Bicep independently discovers the nearest `bicepconfig.json` and follows its `extends` chain as specified in [REP 0023](https://github.com/Azure/bicep-reps/blob/main/final/0023-bicep-configuration-inheritance.md). An inherited base need not be named `bicepconfig.json` or live under the entrypoint's directory. Bicep also knows which configuration file actually declares a version pin, so `update` edits that file rather than adding an override to the leaf. This traversal covers local source files, not dependencies inside published registry artifacts.

Dependabot's `directory` selects entrypoints, not the boundary of a Bicep dependency graph. It must make the repository files available to Bicep for traversal, rather than parse references and `extends` in Ruby to decide what to fetch. Reachable files outside `directory` are eligible for updates unless explicitly excluded; excluded files may still be read to interpret an entrypoint but are not edit targets. The same occurrence reached from several entrypoints should be proposed once. We need to confirm this update scope with Dependabot maintainers. A referenced file outside the repository may work for a person's local invocation but is not available to a hosted repository job; that job must say it cannot analyze the graph, not quietly return a partial inventory.

### Version sources and supported references

Module discovery uses Bicep's artifact resolution and registry code, extended as necessary to list tags in the resolved repository. The module ecosystem covers Bicep OCI registry modules with exact, comparable tags, regardless of whether they are AVM modules, other public modules, or private modules with configured credentials. Public AVM is the first end-to-end test case, not a permanent filter; private registry jobs require a packaged-runner authentication test before being enabled. An alias named `public` or `avm` can point to a mirror. Candidates must come from the actual configured registry, never an assumed public index. The original tag spelling is preserved for edits, even if versions are normalized for comparison.

The existing `use-recent-module-versions` rule, reviewed in [Azure/bicep#14309](https://github.com/Azure/bicep/pull/14309), provides a starting point for reference discovery, strict SemVer comparison, and a narrow source replacement. Today it checks public MCR modules against cached public-module metadata and suggests the newest version; it does not provide live tag discovery for an arbitrary configured registry. Reuse or extract its comparison and default recommendation behavior with the actual registry's candidates, without turning a missing cache entry into a successful "no updates" result. `update` takes the caller's exact target rather than the linter's preferred version.

When a registry-looking reference resolves to a local file through the existing `moduleAliasesMock` or the `artifacts.redirects` proposed in [REP 0022](https://github.com/Azure/bicep-reps/blob/main/active/0022-artifact-aliases-and-redirects.md), Bicep should follow that local file for analysis but not offer registry updates for the redirected occurrence as though it had loaded a published module. This does not make REP 0022 a prerequisite for the update commands. Template Specs, digest-only references, and tags that cannot be compared are reported as unsupported, not silently omitted. Initial support for `module`, compile-time `import`, and `.bicepparam` `using` references needs an explicit coverage decision; unimplemented forms should be identified as such.

Resource API candidates use Bicep's existing type catalog and the version lookup behind `use-recent-api-versions`, not a new Azure service. The rule was [ported from the ARM TTK](https://github.com/Azure/bicep/pull/7612); its [730-day default maximum age](https://github.com/Azure/bicep/pull/10344) and [90-day recommendation grace period](https://github.com/Azure/bicep/pull/19205) have also been reviewed. Its stable/preview handling and fallback when every version is within the grace period are part of that policy. Extract and reuse the same logic and applicable `bicepconfig.json` settings for `outdated` and the resource ecosystem; do not invent a second API-version timer or make linter enablement a prerequisite. In particular, the rule may accept an existing version despite newer candidates, and its acceptable-version set alone is not a list of recommended edits. A newer known version may appear in the JSON inventory without making the current version outdated under that policy. The resource ecosystem uses only the versions the linter would suggest if enabled, with the same effective configuration. Dependabot's explicit `ignore` and applicable cooldown settings can further restrict those suggestions, not broaden them. The catalog is not proof of availability in a particular subscription or region, and an API-version date is not necessarily the publication date required for Dependabot cooldowns. Date-based API versions must not be compared as SemVer.

Dependabot's configured registry secrets and proxy appear sufficient for private registries, but that requires a packaged-runner test of authentication, proxy routing, and noninteractive tag listing. Credentials must not appear in CLI arguments, JSON results, logs, or edited files. Authentication errors cannot cause a fallback to a public registry. The published-module content itself is not scanned or rewritten.

### Bicep CLI bootstrap and version pins

The Dependabot runner always obtains the latest Bicep CLI from the [Bicep CDN](https://msazure.visualstudio.com/One/_wiki/wikis/Azure%20Deployments%20Team%20Wiki/750669/Bicep-CDN) first. That executable follows the source and configuration graph to find the effective `cli.version`, including values declared in an extended base configuration. If pinned, the runner gets the matching CLI from the CDN and uses it for module and resource analysis; without a pin, it keeps the latest. For a range, it chooses the newest available release satisfying the constraint, as proposed in the [version-pinning REP](https://github.com/Azure/bicep-reps/pull/11). If reachable files impose incompatible constraints, the runner must report the conflict rather than guess which pin wins.

The CLI-pin ecosystem uses the bootstrap executable to discover releases and edit an exact `cli.version` pin; a range can be resolved for running Bicep without pretending that its update policy is the same as an exact pin. Range edits are deferred until their intended meaning is agreed. Discovery must not require successful compilation with a bootstrap CLI that violates the project pin. [AzCLI's Bicep downloader](https://github.com/Azure/azure-cli/blob/dev/src/azure-cli/azure/cli/command_modules/resource/_bicep.py) demonstrates CDN release and platform downloads; the version-pinning REP proposes AzCLI reading `bicepconfig.json`, but does not establish that current AzCLI already does so. A pinned CLI too old to support the commands or JSON contract needs an explicit unsupported-version result and a minimum-version decision before rollout.

### Dependabot enrollment and implementation

These are proposed `package-ecosystem` identifiers, subject to Dependabot review:

| Ecosystem | Updates | Reason for separate enrollment |
| --- | --- | --- |
| `bicep-modules` | OCI module references | Published packages with registry and tag semantics |
| `bicep-resources` | Resource API-version occurrences | Date-based service contracts; updating can change deployment behavior |
| `bicep-cli` | Exact compiler pins in `bicepconfig.json` | Different source, file target, and schedule |

For example, enabling modules must not implicitly enable resource API changes:

```yaml
version: 2
updates:
  - package-ecosystem: "bicep-modules"
    directory: "/infra"
    schedule:
      interval: "weekly"
```

The existing [`allow` and `ignore` options](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#allow--) filter dependencies, version ranges, or SemVer update types; [`cooldown`](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#cooldown--) delays releases; [`versioning-strategy`](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#versioning-strategy--) governs how supported ecosystems edit version requirements, not which versions Bicep recommends. None is a sound override of Bicep's API-version age and grace policy. We do not propose new `dependabot.yml` keys or Dependabot job fields:

| Ecosystem | Dependabot selection |
| --- | --- |
| Modules | Use `available` comparable OCI tags for Dependabot's SemVer `allow`/`ignore` rules. With no restrictive rules, choose the newest, matching the module linter's recommendation before other Dependabot policies such as prerelease filtering. If the newest is ignored, an older permitted update remains possible. |
| Resource API versions | Use **only** Bicep's `recommended` replacements for the current occurrence; existing Dependabot rules can only narrow that set. Do not interpret dates as SemVer update types. |
| CLI pins | Use CDN releases for exact pins, with no linter policy; range edits remain deferred. |

Enrolling in the resource ecosystem therefore proposes the same resource API replacements that `use-recent-api-versions` would warn about under the effective `bicepconfig.json`, not every later date in the catalog. `outdated --selection available` remains a useful manual inventory, but Dependabot does not use that pool for resources. Aggressive resource updates through a new Dependabot setting are deferred rather than added to the initial integration.

The Bicep integration shares a CLI runner and JSON adapter across the approved ecosystems. Following [Dependabot's new-ecosystem guide](https://github.com/dependabot/dependabot-core/blob/main/NEW_ECOSYSTEMS.md), each ecosystem still needs registration, `FileFetcher`, `FileParser`, `UpdateChecker`, and `FileUpdater` behavior, updater-image packaging, tests, and hosted-service onboarding. Fetching must provide a repository snapshot or equivalent access sufficient for Bicep to follow reachable files and configurations. The parser maps Bicep occurrences to Dependabot dependencies; the checker selects the Bicep candidate pool as above and applies Dependabot's configured rules; the updater requests exact Bicep edits and returns changed files. The CLI-pin recommendation comes from Bicep CDN release selection rather than a linter policy. Publication dates for cooldowns need a reliable source; tag order must not be treated as a release date. We do not propose Bicep-specific meanings for `allow.dependency-type`.

### Server side changes

No new Bicep registry protocol or Azure service is required. Dependabot hosted enablement is coordinated separately from merging code into dependabot-core.

### Microsoft.Resources/deployments API changes

None.

### Example

Suppose `infra/main.bicep` references `../shared/storage.bicep`, whose own configuration inherits `shared/bicepconfig.base.json`. The shared source contains:

```bicep
module storage 'br/public:avm/res/storage/storage-account:0.6.0' = {
  name: 'storage'
}
```

`bicep outdated infra/main.bicep --kind module` follows the local file, resolves its own effective configuration, and reports available versions for the resolved registry and repository. If that registry offers `0.7.0`, Dependabot can select it under the module ecosystem and ask Bicep to change only `0.6.0` in this occurrence. An excluded shared file can inform analysis but is not edited. A resource API version in either file is unchanged unless the resource ecosystem is separately enabled. If configuration redirects the module to a local implementation, no remote update is proposed for that occurrence; Bicep continues following the local implementation.

## Tradeoffs

- Three ecosystems require more registration and onboarding, but prevent existing module jobs from unexpectedly proposing resource API or compiler changes. The split needs Dependabot maintainer approval.
- The resource ecosystem cannot opt into more aggressive API versions through the initial Dependabot configuration. This avoids a new hosted configuration contract and keeps automatic resource updates aligned with Bicep's reviewed linter policy; broader discovery remains available to people through the CLI.
- A stable JSON boundary requires compatibility work, but avoids a Ruby Bicep parser and an unsupported public compiler-library API.
- Graph traversal can reach shared files outside a configured directory and make one edit affect several deployments. Respect explicit exclusions, identify the actual edit target, and let users validate proposed PRs through CI.
- Native registry lookup must work with Dependabot's credential proxy. Fail visibly when it does not; do not switch sources or leak secrets to make a job appear successful.

## Alternatives

**One ecosystem with name filters.** A user could restrict dependency names with `allow`, but ordinary or broad rules would also enable resource API updates whenever that inventory was added. Separate enrollment avoids changing existing jobs' meaning.

**Parse references or list registries in Ruby.** This could resemble other Dependabot integrations, but would duplicate Bicep's aliases, redirects, inherited configuration, and version-source rules. It would not provide the same native `outdated` experience.

**Let Dependabot edit Bicep text.** The Bicep linter already has targeted quick-fix machinery, but a diagnostic-driven fix cannot express any exact target Dependabot selects. Bicep-owned editing preserves one source of truth for reference syntax and occurrence selection.

## Rollout plan

1. Confirm ecosystem boundaries and names, reachable-file update behavior, registry-secret delivery, and supported pinned CLI versions with Dependabot maintainers.
2. Add Bicep graph-aware inventory, registry and API-version candidate discovery, versioned `outdated` output, and exact `update` edits. Extract applicable linter lookup, comparison, recommendation, and edit logic while preserving existing linter results. Test that default CLI and Dependabot suggestions match Bicep's linter recommendations, including acceptable current versions, configured API-version ages, grace periods, and stable/preview cases; test `available` selection separately, along with empty inventories, unsupported references, lookup errors, and stale edits. Start end-to-end validation with public AVM modules and other public OCI modules; keep resource and CLI-pin kinds explicit rather than exposing them through the module job.
3. Contribute the module ecosystem to dependabot-core using the native commands and existing SemVer filters. Add registry-proxy, inherited-config, redirect, multiple-occurrence, excluded-file, and stale-edit cases to its packaged-runner tests. Enable it in the hosted service under the agreed preview process; test private registry jobs end to end before claiming support for them.
4. Add the separately enrolled resource API and CLI-pin ecosystems against the same Bicep interface, validating API-version policy, inherited pin locations, cooldown metadata, and older pinned CLIs before hosted enablement.

No ARM feature flag or deployment API change is needed. `outdated` does not compile or deploy a project, and a candidate is not proof that the update works with its pinned compiler or deployment environment.

## Unresolved questions

- Will Dependabot accept three ecosystems and updates to reachable, non-excluded files outside `directory`?
- Which module reference forms and resource API expressions are supported first?
- What minimum pinned Bicep version can run the native commands, and how should older pins be handled without silently ignoring them?
- Where can the ecosystems obtain reliable publication dates for cooldowns, particularly from OCI registries?

## Out of scope

Updating dependencies inside published modules, Template Spec updates, digest refreshes, arbitrary version-looking deployment values, automatic compilation or deployment validation, and automatic PR merging. Extension dependencies and module version ranges can be addressed separately without making them look like OCI module tags or resource API dates.

_Drafted by Copilot with GPT-6 Sol._
