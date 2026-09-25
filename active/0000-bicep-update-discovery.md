---
REP Number: "Unassigned"
Author: andyleejordan (Andy Jordan)
Start Date: 2026-09-18
Feature Status: Private Preview
---

# Bicep update discovery and Dependabot integration

## Summary

Add `bicep outdated` to find newer versions of Bicep dependencies, with human-readable and versioned JSON output, and `bicep update` to apply an explicitly selected version to one source occurrence. Bicep owns reference discovery, configuration resolution, candidate lookup, and source edits. Dependabot chooses versions under its existing policies and opens pull requests.

We propose three independently enabled Dependabot ecosystems for registry modules, resource API versions, and Bicep CLI version pins. Their names and separation need review with the Dependabot team. The implementation can start with public AVM modules without implicitly enabling the other two update types.

## Terms and definitions

- **Entrypoint**: A Bicep or Bicep parameters file selected for analysis. Its reachable local files can be outside the entrypoint's directory.
- **Occurrence**: One versioned reference in a source or configuration file. The same dependency can have several occurrences at different versions.
- **Effective configuration**: The settings Bicep resolves for a source file from its nearest `bicepconfig.json`, any inherited configuration files, and the built-in defaults.
- **Candidate**: An available version at the resolved source. Bicep's existing recommendation logic distinguishes candidates it would suggest by default; availability alone is not a claim that an update is safe.

## Motivation

[dependabot/dependabot-core#15952](https://github.com/dependabot/dependabot-core/issues/15952) requests Dependabot updates for public AVM modules. Bicep already has [`use-recent-module-versions`](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/linter-rule-use-recent-module-versions) and [`use-recent-api-versions`](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/linter-rule-use-recent-api-versions) linter rules, but warnings and "use latest" quick fixes are not an update-discovery or exact-target editing interface. A module reference may use a registry alias, a local redirect, or configuration inherited from another directory. Bicep source files can also refer to local modules whose own configuration differs from the entrypoint's. A Ruby parser or registry client would have to duplicate these Bicep-specific rules and keep up as the language changes.

The native commands should be useful without Dependabot: a person can see which versions are available, then request one exact edit. Neither `outdated` nor `update` silently selects "latest" in a Bicep source file. Dependabot applies its own `allow`, `ignore`, grouping, cooldown, and prerelease rules before it requests an edit.

## Detailed design

### Client side changes: one native interface

These commands illustrate the intended interface; exact flag spelling can be settled during CLI implementation:

```sh
bicep outdated infra/main.bicep
bicep outdated infra/main.bicep --kind module --output-format json --schema-version 1
bicep outdated infra/main.bicep --kind resource-api --output-format json --schema-version 1
bicep outdated infra/main.bicep --kind cli --output-format json --schema-version 1
bicep update shared/storage.bicep --kind module --reference 'br/public:avm/res/storage/storage-account:0.6.0' --version 0.7.0
bicep update --request update.json
```

By default, the human-facing command reports modules; other kinds are explicit so adding them cannot change a user's module-only results. Dependabot always requests exactly one kind. `outdated` makes no source changes. Its JSON identifies each occurrence, its original spelling and resolved dependency identity, its current version or constraint, available candidates and their source, Bicep's recommended candidates where a recommendation policy exists, and whether the reference was checked, unsupported, or failed. A registry module identifies its registry and repository; a resource API identifies its resource type, not a fictional registry. For modules and resource APIs, human output and Dependabot's default update proposals use Bicep's existing recommendation logic, even if the corresponding linter rule is disabled; the broader availability list is context, not a separate default policy. An authentication failure or unresolved reference must not look like "up to date." The CLI writes machine results to stdout and progress or errors to stderr.

`bicep update` takes one occurrence, an expected current value, and an exact target chosen by its caller. It verifies that the request still identifies the same source and reference, changes only that occurrence, and preserves aliases, formatting, comments, and unrelated versions. The human-facing command identifies an unambiguous reference within the named file; if two occurrences match, it asks for a more precise selection instead of editing both. Dependabot supplies the precise occurrence in its request and can rerun discovery between successive edits to the same file. Stale or ambiguous requests fail.

This discover-then-explicit-update design has familiar precedent: [.NET's `dotnet package list --outdated --format json --output-version 1`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-list) separates discovery from [`dotnet package add ... --version`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-add), while [`npm outdated`](https://docs.npmjs.com/cli/v11/commands/npm-outdated/) supports targeted and JSON output. Bicep needs occurrence-level selection because one file can use the same module at two versions. The JSON boundary is a Bicep CLI contract, not a supported public .NET API or an export of compiler syntax-tree objects; field names, spans, and exit codes can be finalized with implementation tests.

### Following the source and configuration graph

`bicep outdated main.bicep` follows its local Bicep references, including local implementations selected through aliases or redirects. For each reached source file, Bicep independently discovers the nearest `bicepconfig.json` and follows its `extends` chain as specified in [REP 0023](https://github.com/Azure/bicep-reps/blob/main/final/0023-bicep-configuration-inheritance.md). An inherited base need not be named `bicepconfig.json` or live under the entrypoint's directory. Bicep also knows which configuration file actually declares a version pin, so `update` edits that file rather than adding an override to the leaf. This traversal covers local source files, not dependencies inside published registry artifacts.

Dependabot's `directory` selects entrypoints, not the boundary of a Bicep dependency graph. It must make the repository files available to Bicep for traversal, rather than parse references and `extends` in Ruby to decide what to fetch. Reachable files outside `directory` are eligible for updates unless explicitly excluded; excluded files may still be read to interpret an entrypoint but are not edit targets. The same occurrence reached from several entrypoints should be proposed once. We need to confirm this update scope with Dependabot maintainers. A referenced file outside the repository may work for a person's local invocation but is not available to a hosted repository job; that job must say it cannot analyze the graph, not quietly return a partial inventory.

### Version sources and supported references

Module discovery uses Bicep's artifact resolution and registry code, extended as necessary to list tags in the resolved repository. It starts with exact, comparable OCI tags and public AVM modules; expanding to other public and private OCI repositories must use the same source-aware path. An alias named `public` or `avm` can point to a mirror. Candidates must come from the actual configured registry, never an assumed public index. The original tag spelling is preserved for edits, even if versions are normalized for comparison.

The existing `use-recent-module-versions` rule, reviewed in [Azure/bicep#14309](https://github.com/Azure/bicep/pull/14309), provides a starting point for reference discovery, strict SemVer comparison, and a narrow source replacement. Today it checks public MCR modules against cached public-module metadata and suggests the newest version; it does not provide live tag discovery for an arbitrary configured registry. Reuse or extract its comparison and default recommendation behavior with the actual registry's candidates, without turning a missing cache entry into a successful "no updates" result. `update` takes the caller's exact target rather than the linter's preferred version.

When a registry-looking reference resolves to a local file through the existing `moduleAliasesMock` or the `artifacts.redirects` proposed in [REP 0022](https://github.com/Azure/bicep-reps/blob/main/active/0022-artifact-aliases-and-redirects.md), Bicep should follow that local file for analysis but not offer registry updates for the redirected occurrence as though it had loaded a published module. This does not make REP 0022 a prerequisite for the update commands. Template Specs, digest-only references, and tags that cannot be compared are reported as unsupported, not silently omitted. Initial support for `module`, compile-time `import`, and `.bicepparam` `using` references needs an explicit coverage decision; unimplemented forms should be identified as such.

Resource API candidates use Bicep's existing type catalog and the version lookup behind `use-recent-api-versions`, not a new Azure service. The rule was [ported from the ARM TTK](https://github.com/Azure/bicep/pull/7612); its [730-day default maximum age](https://github.com/Azure/bicep/pull/10344) and [90-day recommendation grace period](https://github.com/Azure/bicep/pull/19205) have also been reviewed. Its stable/preview handling and fallback when every version is within the grace period are part of that policy. Extract and reuse the same logic and applicable `bicepconfig.json` settings for `outdated` and the resource ecosystem; do not invent a second API-version timer or make linter enablement a prerequisite. A newer known version may appear in the JSON inventory without making the current version outdated under that policy. Dependabot's explicit `ignore` and cooldown settings still apply after Bicep's recommendation. The catalog is not proof of availability in a particular subscription or region, and an API-version date is not necessarily the publication date required for Dependabot cooldowns. Date-based API versions must not be compared as SemVer.

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

The Bicep integration shares a CLI runner and JSON adapter across the approved ecosystems. Following [Dependabot's new-ecosystem guide](https://github.com/dependabot/dependabot-core/blob/main/NEW_ECOSYSTEMS.md), each ecosystem still needs registration, `FileFetcher`, `FileParser`, `UpdateChecker`, and `FileUpdater` behavior, updater-image packaging, tests, and hosted-service onboarding. Fetching must provide a repository snapshot or equivalent access sufficient for Bicep to follow reachable files and configurations. The parser maps Bicep occurrences to Dependabot dependencies; for modules and resource APIs, the checker starts with Bicep's recommended candidates and applies Dependabot's configured rules; the updater requests exact Bicep edits and returns changed files. The CLI-pin checker uses Bicep CDN release candidates rather than a linter policy. Publication dates for cooldowns need a reliable source; tag order must not be treated as a release date. We do not propose new `dependabot.yml` keys or Bicep-specific meanings for `allow.dependency-type`.

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
- A stable JSON boundary requires compatibility work, but avoids a Ruby Bicep parser and an unsupported public compiler-library API.
- Graph traversal can reach shared files outside a configured directory and make one edit affect several deployments. Respect explicit exclusions, identify the actual edit target, and let users validate proposed PRs through CI.
- Native registry lookup must work with Dependabot's credential proxy. Fail visibly when it does not; do not switch sources or leak secrets to make a job appear successful.

## Alternatives

**One ecosystem with name filters.** A user could restrict dependency names with `allow`, but ordinary or broad rules would also enable resource API updates whenever that inventory was added. Separate enrollment avoids changing existing jobs' meaning.

**Parse references or list registries in Ruby.** This could resemble other Dependabot integrations, but would duplicate Bicep's aliases, redirects, inherited configuration, and version-source rules. It would not provide the same native `outdated` experience.

**Let Dependabot edit Bicep text.** The Bicep linter already has targeted quick-fix machinery, but a diagnostic-driven fix cannot express any exact target Dependabot selects. Bicep-owned editing preserves one source of truth for reference syntax and occurrence selection.

## Rollout plan

1. Confirm ecosystem boundaries and names, reachable-file update behavior, registry-secret delivery, and supported pinned CLI versions with Dependabot maintainers.
2. Add Bicep graph-aware inventory, registry and API-version candidate discovery, versioned `outdated` output, and exact `update` edits. Extract applicable linter lookup, comparison, recommendation, and edit logic while preserving existing linter results. Test that default CLI and Dependabot suggestions match Bicep's linter recommendations, including configured API-version ages, grace periods, and stable/preview cases. Start end-to-end validation with public AVM modules; keep resource and CLI-pin kinds explicit rather than exposing them through the module job.
3. Contribute the module ecosystem to dependabot-core using the native commands. Add registry-proxy, inherited-config, redirect, multiple-occurrence, excluded-file, and stale-edit cases to its packaged-runner tests. Enable it in the hosted service under the agreed preview process.
4. Add the separately enrolled resource API and CLI-pin ecosystems against the same Bicep interface, validating API-version policy, inherited pin locations, cooldown metadata, and older pinned CLIs before hosted enablement. Test private registry jobs end to end before claiming support for them.

No ARM feature flag or deployment API change is needed. `outdated` does not compile or deploy a project, and a candidate is not proof that the update works with its pinned compiler or deployment environment.

## Unresolved questions

- Will Dependabot accept three ecosystems and updates to reachable, non-excluded files outside `directory`?
- Which module reference forms and resource API expressions are supported first?
- What minimum pinned Bicep version can run the native commands, and how should older pins be handled without silently ignoring them?
- Where can the ecosystems obtain reliable publication dates for cooldowns, particularly from OCI registries?

## Out of scope

Updating dependencies inside published modules, Template Spec updates, digest refreshes, arbitrary version-looking deployment values, automatic compilation or deployment validation, and automatic PR merging. Extension dependencies and module version ranges can be addressed separately without making them look like OCI module tags or resource API dates.

_Drafted by Copilot with GPT-6 Sol._
