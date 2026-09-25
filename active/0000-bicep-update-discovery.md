---
REP Number: "Unassigned"
Author: andyleejordan (Andy Jordan)
Start Date: 2026-09-18
Feature Status: "Proposed; not shipped"
---

# Bicep update discovery and Dependabot integration

## Summary

Add `bicep outdated` for read-only, JSON-capable update discovery and `bicep update` for an explicitly requested, one-occurrence-at-a-time edit. Bicep owns parsing, version lookup, and source edits; Dependabot owns scheduling, version selection, and pull requests. We propose three independently enabled Dependabot ecosystems: modules, resource API versions, and Bicep CLI version pins. The ecosystem split is pending review with the Dependabot team.

## Motivation

[dependabot/dependabot-core#15952](https://github.com/dependabot/dependabot-core/issues/15952) asks for public AVM module updates. Bicep already understands registry references, aliases, configuration, and resource API versions; duplicating that knowledge in Ruby would make the integration fragile. Native commands also let users inspect and apply individual updates without Dependabot.

## Detailed design

### One Bicep interface, three proposed ecosystems

| Proposed `package-ecosystem` | Inventory | Update target |
| --- | --- | --- |
| `bicep-modules` | Exact-tag OCI module references | The selected reference in a `.bicep` or `.bicepparam` file |
| `bicep-resources` | Resource API versions known to Bicep | One resource API-version occurrence |
| `bicep-cli` | The effective `cli.version` constraint in `bicepconfig.json` | That configuration property (exact pins first) |

These names and the three-way split are proposals, not accepted Dependabot identifiers. A module job must never begin proposing resource API or compiler updates merely because the Bicep command learns to report them. Resource API changes can affect deployment behavior, so they require their own opt-in and schedule. Compiler pin updates also have a different target and release source. Each ecosystem invokes the same Bicep commands with an explicit kind; shared logic belongs in Bicep where practical.

For example, enrollment in module updates alone would look like:

```yaml
version: 2
updates:
  - package-ecosystem: "bicep-modules"
    directory: "/infra"
    schedule:
      interval: "weekly"
```

Dependabot selects files within its configured directory and exclusions and invokes Bicep for each selected file. It does not follow an excluded local file into the update inventory. Bicep may still read applicable configuration to resolve a selected reference. A person can instead name one source file; the initial contract need not scan an entire repository implicitly.

### Discovery and editing

Proposed commands (option spelling is subject to CLI review):

```sh
bicep outdated main.bicep --kind module
bicep outdated main.bicep --kind module --output-format json --schema-version 1
bicep outdated main.bicep --kind resource-api --output-format json --schema-version 1
bicep outdated bicepconfig.json --kind cli --output-format json --schema-version 1
bicep update --request update.json
```

`outdated` returns each checked occurrence with its kind, source file, original spelling, resolved identity, current version or constraint, candidate versions, and source provenance. It reports unsupported references and lookup errors distinctly from up-to-date references. JSON has a versioned envelope, Bicep CLI version, requested kind, results, and diagnostics; it is a Bicep contract, not a dump of compiler objects or a Dependabot-specific protocol. Machine output is one JSON document on stdout, with diagnostics to stderr as appropriate. Finding an update is not an error; failed lookups must not become empty successful results.

`update.json` specifies exactly one occurrence from an `outdated` result, its expected original value and file state, and the exact target version selected by the caller. `bicep update` checks those preconditions and that the target belongs to the resolved source, changes only that occurrence, and reports the changed file. It rejects stale or ambiguous requests rather than editing a different reference. Callers rerun discovery before the next edit to the same file. Bicep preserves comments, formatting, aliases, and unrelated versions; the operation does not choose a version, restore modules, compile, or deploy. The JSON request format and stale-file guard need protocol fixtures before release.

This is deliberately two commands. [.NET's `dotnet package list --outdated --format json --output-version 1`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-list) separates machine-readable discovery from [`dotnet package add ... --version`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-add), which can update a particular package. [`npm outdated [package-spec] --json`](https://docs.npmjs.com/cli/v11/commands/npm-outdated/) also supports targeted discovery. We follow the discover/explicit-update pattern, but require an exact occurrence because Bicep files can contain the same module at different versions. Dependabot, not Bicep, applies `allow`, `ignore`, groups, cooldowns, and prerelease rules.

### Version sources and CLI bootstrap

Module lookup uses Bicep's registry and configuration code, including alias resolution and the registry selected by the project. It queries that repository's available tags; it does not substitute a public index for a private mirror or reimplement OCI access in Ruby. Dependabot's configured registry secrets and proxy appear workable for private registries, subject to an end-to-end test of authentication, proxy routing, and noninteractive execution. Credentials must not appear in JSON, command arguments, logs, or edited source. Authentication errors are errors, not permission to fall back to a public registry.

Resource API candidates come from Bicep's type catalog, reusing the lookup behind [`use-recent-api-versions`](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/linter-rule-use-recent-api-versions), not its warning threshold. A newer known API version is not necessarily available in every region or safe to adopt. CLI release candidates come from the [Bicep CDN](https://msazure.visualstudio.com/One/_wiki/wikis/Azure%20Deployments%20Team%20Wiki/750669/Bicep-CDN). Missing publication dates must not be invented for Dependabot cooldown decisions.

The Dependabot runner always bootstraps the latest Bicep CLI from the CDN first. It uses that executable to find the effective `cli.version` in `bicepconfig.json` and, when a pin exists, resolves and downloads the pinned version from the CDN and switches to it for module and resource work. For a range, use the newest available release satisfying the constraint, consistent with the [version-pinning REP](https://github.com/Azure/bicep-reps/pull/11). Without a pin, continue with latest. A CLI-pin update is discovered and applied with the bootstrap executable, since changing the constraint may target a different executable. Initially, update exact pins; report range changes as unsupported rather than guessing how to widen a constraint. Discovering a pin must not require successful compilation with a CLI that violates it.

The pinning REP also describes AzCLI resolving `bicepconfig.json` and downloading a matching version; it is design precedent, not a claim that current AzCLI already implements it. [AzCLI's Bicep downloader](https://github.com/Azure/azure-cli/blob/dev/src/azure-cli/azure/cli/command_modules/resource/_bicep.py) shows the CDN release-list and platform-binary download paths. A pinned CLI older than the commands or JSON schema this integration requires cannot be silently replaced by latest: report an unsupported pin and agree on a minimum supported CLI version during rollout.

## Tradeoffs and alternatives

- Three ecosystems add Dependabot registration and packaging work, but make each kind an explicit enrollment. Dependabot maintainers may recommend a different separation; do not settle the identifiers before their review.
- A stable Bicep JSON boundary costs schema maintenance, but avoids both a Ruby Bicep parser and a public Bicep .NET API commitment.
- Native registry lookup must work through Dependabot's credential proxy. If that fails for a particular registry, report it; do not quietly use Ruby or a different source.
- Exact, one-at-a-time edits require rediscovery between edits, but avoid accidental convergence of references that intentionally use different versions.

## Implementation and rollout

1. Agree with Dependabot maintainers on ecosystem boundaries and names, CLI packaging, registry-secret delivery, and the minimum supported pinned Bicep version.
2. In Bicep, implement the shared occurrence inventory, candidate lookup, versioned `outdated` JSON, and exact-target `update` for module references, resource APIs, and `cli.version`. Reuse existing parser, registry, configuration, and API-version code; do not tie discovery to linter warnings.
3. In dependabot-core, follow its [new-ecosystem guide](https://github.com/dependabot/dependabot-core/blob/main/NEW_ECOSYSTEMS.md): scaffold each approved ecosystem with `rake ecosystem:create[NAME]`, then implement its `FileFetcher`, `FileParser`, `UpdateChecker`, and `FileUpdater`. Share a CLI runner and Bicep JSON adapter across them. Fetch the selected source files and applicable configuration; bootstrap from the CDN and resolve the pin; parse `outdated` for the requested kind; apply Dependabot's version policy and cooldown; pass the chosen occurrence and version to `update`; return only the changed files. Wire credentials through the existing Dependabot mechanism, package the CLI for the updater image, and complete infrastructure registration and hosted-service onboarding. Recheck generated registrations rather than assuming an ecosystem directory alone enables jobs.
4. Test public modules, private registries through the credential proxy, mirrors/aliases, inherited configuration, multiple versions of one dependency, excluded files, API-version candidates, exact and range CLI pins, stale edits, unsupported pins, JSON errors, and cooldown metadata. Validate the packaged runner and hosted job before enabling the ecosystems.

## Unresolved questions

- Will Dependabot accept three ecosystems, and what identifiers and release gates should they use?
- What is the exact file-selection, JSON request/response, occurrence identity, exit-code, and compatibility contract? How should a human-facing `update` select one occurrence without preparing JSON?
- Which reference forms (`module`, `import`, `.bicepparam` `using`, resource declarations, API-version functions) are supported in the first increment, and which are reported as unsupported?
- How will the runner handle a pinned Bicep release below the supported command version, and which publication metadata is reliable enough for cooldowns?

## Out of scope

Automatic upgrades inside published artifacts, Template Specs, digest-only references, arbitrary version-looking parameter values, compilation/deployment validation, and automatic PR merging. Reporting a candidate does not establish deployment compatibility.

_Drafted by Copilot with GPT-6 Sol._
