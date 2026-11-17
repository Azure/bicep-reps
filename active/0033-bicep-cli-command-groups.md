---
REP Number: "0033"
Author: shenglol (Shenglong Li)
Start Date: 2026-11-16
Feature Status: Public
---

# Bicep CLI command groups

## Summary

The Bicep CLI has grown one top-level command at a time. Today it has `build`, `build-params`, `generate-params`, `decompile-params`, `publish-extension`, and a single command group, `docs`. Before the CLI reaches 1.0, we want to settle on a structure that we can commit to long term.

The approach is straightforward. Commands that operate on a `.bicep` file remain top-level verbs. Other commands move into command groups named after what they operate on, and group names are singular nouns:

```text
bicep compile main.bicep              # `bicep build` continues to work
bicep param compile main.bicepparam   # replaces `bicep build-params`
bicep param generate main.bicep       # replaces `bicep generate-params`
bicep extension publish ...           # replaces `bicep publish-extension`
bicep doc generate ...                # replaces `bicep docs generate`
bicep local deploy main.bicepparam    # replaces `bicep local-deploy`
```

Nothing breaks before 1.0. `build` remains a permanent alias of `compile`, and every existing command keeps working with the same options. We will announce the change first, introduce deprecation warnings in a later release, and remove the old names in 1.0.

## Terms and definitions

- **Command group**: A top-level noun that contains subcommands but doesn't perform an action by itself, such as `bicep doc`.
- **Alias**: An alternative name for a command, shown next to it in help. Aliases are never deprecated.
- **Legacy name**: A shipped command name that this REP replaces, such as `build-params`.

## Motivation

The CLI currently has 19 top-level entries:

| Command | Input | Description |
| :-- | :-- | :-- |
| `build` | `.bicep` | Compiles a Bicep file to an ARM template. |
| `build-params` | `.bicepparam` | Compiles a parameters file to a parameters JSON file. |
| `generate-params` | `.bicep` | Generates a parameters file from a Bicep file. |
| `decompile` | ARM template `.json` | Decompiles an ARM template to a Bicep file. |
| `decompile-params` | Parameters `.json` | Decompiles a parameters JSON file to a `.bicepparam` file. |
| `format` | `.bicep` | Formats a Bicep file. |
| `lint` | `.bicep` | Lints a Bicep file. |
| `test` | `.bicep` | Runs tests in a Bicep file. |
| `restore` | `.bicep` | Restores external modules to the local cache. |
| `publish` | `.bicep` | Publishes a module to a registry. |
| `publish-extension` | Extension index `.json` | Publishes an extension to a registry (experimental). |
| `docs generate` | `.bicep` | Generates module documentation (experimental). |
| `snapshot` | `.bicepparam` | Generates or validates a deployment snapshot. |
| `deploy`, `what-if`, `teardown` | `.bicepparam` | Deploys, previews, or tears down a deployment (experimental). |
| `local-deploy` | `.bicepparam` | Performs a local deployment (experimental). |
| `console` | n/a | Opens an interactive Bicep console. |
| `jsonrpc` | n/a | Starts listening for JSON-RPC requests. |

They all sit in one flat namespace, and the list continues to grow. This causes several problems.

When a verb needs to apply to something other than a `.bicep` file, we add a suffix: `build-params`, `decompile-params`, `publish-extension`. The suffix doesn't mean the same thing in each case. `build-params` reads a parameters file, `generate-params` writes one, and `decompile-params` writes one from JSON. None of this is apparent from the names.

`bicep --help` doesn't show these relationships either. The three `*-params` commands are closely related, but help lists them among all the other commands.

We have also started introducing command groups without an agreed convention. `bicep docs` ([REP 0025](./0025-module-documentation-generation.md)) was the first, and the dependency update proposal in [#28](https://github.com/Azure/bicep-reps/pull/28) is considering a second. Each new feature raises the same questions: whether to use a group, whether its name should be singular or plural, and whether to use a full word or an abbreviation. Without a rule, these will be answered differently each time.

Finally, all top-level verbs share one namespace. #28 has already run into this: a top-level `bicep update` could be read as updating the Bicep CLI itself.

Each of these issues is minor on its own. Together, they make the CLI harder to learn, and they become harder to fix after 1.0. Now is the right time to address them.

## Detailed design

### Client side changes

#### Naming rules

The following rules apply to all new commands and to the renames proposed below.

1. **Commands that operate on a `.bicep` file stay top-level.** This includes `compile`, `decompile`, `format`, `lint`, `test`, `restore`, and `publish`, as well as session-level commands such as `console` and `jsonrpc`. Working with Bicep files is the core purpose of the tool, so these commands should stay as short as possible.
2. **Other commands belong to a group.** If a command operates on a different kind of artifact, or belongs to an area that has (or is expected to have) several operations, it uses the form `bicep <noun> <verb>`.
3. **Group names are singular nouns.** See [Why singular](#why-singular).
4. **Names don't use hyphenated verb-noun compounds.** A hyphen is allowed only when the whole name is one established term, such as `what-if`.
5. **The same operation uses the same verb.** Turning source into ARM JSON is `compile`, both at the top level and in the `param` group.
6. **A command or group has at most one alias.** An alias is either a shipped name that must be preserved (`build`) or a shortened form of the singular noun (`dep`). Plural aliases are not allowed.
7. **Moving a command doesn't change its options.** A regrouped command accepts exactly the same arguments, options, and defaults as the command it replaces. Migrating only requires changing the command name.
8. **Legacy names keep working until 1.0.** See [Legacy names](#legacy-names).

#### Why singular

A group is named after the kind of thing its commands operate on. The name stays singular even when a command processes many items at once. For example, `param compile --pattern` may compile many parameters files, but the group is still `param`. The group name identifies a category; it isn't a count.

This is the prevailing convention across widely used CLIs:

| CLI | Groups |
| :-- | :-- |
| Azure CLI | `az group`, `az vm`, `az storage account`, `az extension`, `az bicep` |
| GitHub CLI | `gh repo`, `gh issue`, `gh pr`, `gh release`, `gh extension`, `gh secret` |
| Docker | `docker container`, `docker image`, `docker volume`, `docker network`, `docker plugin` |
| .NET CLI | `dotnet tool`, `dotnet workload`, `dotnet package`, `dotnet reference` |
| Helm | `helm dependency`, `helm plugin`, `helm repo` |
| npm | `npm cache`, `npm config`, `npm pkg` |
| Git | `git remote`, `git submodule`, `git worktree` |
| Rust and Go | `rustup toolchain`, `rustup component`, `cargo doc`, `go doc`, `go mod` |

Azure CLI is the most relevant precedent and the strongest reason to choose singular names. Many users install and run Bicep through `az bicep`, which is itself a group in a CLI that uses singular group names (`az group`, `az vm`, `az storage account`). `az bicep` currently mirrors our commands one for one (`az bicep build`, `az bicep build-params`, `az bicep generate-params`). If it mirrors our groups in the future, singular names fit naturally: `az bicep param compile` follows Azure CLI's convention, while `az bicep params compile` would not. Sharing the same convention lets both CLIs expose the same names without either one making an exception.

Several other examples closely match the groups proposed here: `helm dependency` (which also accepts `dep`), `cargo doc`, and `go doc`.

We considered plural names, because "docs" and "params" are common in everyday usage. However, these are informal shorthands that don't generalize to other nouns: "extensions" and "modules" aren't used that way. If `docs` and `params` were plural while `extension` was singular, users would need to remember which is which, and the question would come up again with every new group. Singular names leave no exceptions to remember.

#### Command changes

| Current | Proposed | Legacy name status |
| :-- | :-- | :-- |
| `build` | `compile` (alias `build`) | Permanent alias, never deprecated |
| `build-params` | `param compile` (alias `param build`) | Soft-deprecated |
| `generate-params` | `param generate` | Soft-deprecated |
| `decompile-params` | `param decompile` | Soft-deprecated |
| `publish-extension` | `extension publish` | Soft-deprecated |
| `docs generate` | `doc generate` | Soft-deprecated |
| `local-deploy` | `local deploy` | Soft-deprecated |
| `decompile`, `format`, `lint`, `test`, `restore`, `publish`, `snapshot`, `console`, `jsonrpc` | Unchanged | n/a |
| `deploy`, `what-if`, `teardown` | Unchanged for now (experimental, see [Unresolved questions](#unresolved-questions)) | n/a |
| n/a ([#28](https://github.com/Azure/bicep-reps/pull/28)) | `dependency outdated`, `dependency update` (alias `dep`) | n/a |

The resulting `bicep --help` output would look roughly like this:

```text
Commands:
  compile, build        Compiles a .bicep file to an ARM template.
  decompile             Decompiles an ARM template .json file to .bicep.
  format                Formats a .bicep file.
  lint                  Lints a .bicep file.
  test                  Runs tests in a .bicep file.
  restore               Restores external modules referenced by a .bicep file.
  publish               Publishes a .bicep file to a module registry.
  snapshot              Generates or validates a deployment snapshot.
  console               Opens an interactive Bicep console.
  jsonrpc               Starts listening for JSON-RPC requests.
  param                 Compiles, generates, and decompiles Bicep parameters files.
  extension             Publishes Bicep extensions.
  doc                   Generates documentation for Bicep modules.
  dependency, dep       Discovers and updates versions of Bicep dependencies.
  local                 Runs Bicep deployments locally.
  ...experimental deployment commands...
```

```text
$ bicep param --help
Commands:
  compile, build <Input file>  Compiles a .bicepparam file to a parameters .json file.
  generate <Input file>        Generates a parameters file from a .bicep file.
  decompile <Input file>       Decompiles a parameters .json file to .bicepparam.
```

Running a group without a subcommand, such as `bicep param`, prints the group's help and exits with a non-zero code. This matches the current behavior of `bicep docs`.

#### `compile` and `build`

We use `compile` as the primary name because it pairs with `decompile`.

`build` is by far the most widely used Bicep command. It appears in pipelines, documentation, blog posts, and first-party tools, and users shouldn't have to change it. `build` therefore becomes a full alias of `compile`: it appears on the same help line, and it is never hidden, deprecated, or warned about. New documentation and samples will use `compile`.

#### `param` group

All three parameters-file commands move into this group:

- `param compile` takes a `.bicepparam` file, the same as `build-params`. It uses `compile` to match the top level, and `param build` is an alias for the same reason `build` is.
- `param generate` takes a `.bicep` file and produces a parameters file, the same as `generate-params`. The group is named after the artifact being produced, not the input.
- `param decompile` takes a parameters `.json` file, the same as `decompile-params`.

Options are unchanged. `param compile` accepts `--bicep-file`, `--outfile`, `--outdir`, `--stdout`, `--pattern`, `--no-restore`, `--diagnostics-format`, and `--config`. `param generate` keeps `--output-format json|bicepparam` and `--include-params`.

We chose `param` over `bicepparam` and `parameter`. `bicep bicepparam compile` repeats the tool name, and `parameter` is longer and doesn't match the `.bicepparam` file extension. `param` is short and matches the file extension. It shares its name with the `param` keyword. We consider that acceptable, because a parameters file consists mostly of `param` assignments.

#### `extension` group

`publish-extension` becomes `extension publish`. Although the command is still experimental, extension authors use it in CI pipelines, so it receives the same soft deprecation as the GA commands rather than a hard rename. The group also provides a natural place for future extension-authoring commands.

Module publishing remains the top-level `publish` command, because its input is a `.bicep` file. Whether to also add `module publish` is listed under [Unresolved questions](#unresolved-questions).

#### `doc` group

`docs generate` becomes `doc generate`, consistent with `cargo doc`, `go doc`, and `deno doc`. Because `docs` is new and still experimental, renaming it now has little cost. After it reaches GA, the same change would be a breaking change. Early adopters still get the soft deprecation, and [REP 0025](./0025-module-documentation-generation.md) will be updated to match.

#### `dependency` group

This REP doesn't design the dependency commands; #28 covers that. It only specifies where they belong: `bicep dependency outdated` and `bicep dependency update`, with `dep` as the group alias. This resolves #28 in favor of its Option A (a command group) and is consistent with `helm dependency`.

#### `local` group

`local-deploy` becomes `local deploy`. The `local` group holds commands that run against the local deployment engine instead of Azure. It leaves room for local counterparts of the other deployment commands, such as `local what-if`, without adding more hyphenated names like `local-what-if`. AWS SAM CLI uses the same structure for running serverless applications locally (`sam local invoke`, `sam local start-api`).

`local` is an adjective rather than a noun, so it is a deliberate exception to rule 3. It names where the commands run rather than what they operate on, and no noun says this as clearly. The subcommand names match the Azure deployment commands (`deploy`, `what-if`), so `bicep local deploy` reads as "deploy, but locally."

`local-deploy` is experimental, but extension authors use it to test their extensions, so it receives the same soft deprecation as `publish-extension`.

#### Legacy names

System.CommandLine only supports aliases at the same level, so `build-params` can't be an alias of `param compile`. Instead, each legacy name is registered as a hidden top-level command (a hidden group, in the case of `docs`) that reuses its replacement's options, arguments, and handler, so the two can't drift apart. The output, the files written, and the exit codes are identical. The only difference is the deprecation warning on stderr once warnings are enabled. `bicep build-params --help` continues to work and includes a line that points to `bicep param compile`.

Messages that refer to a command by name use the new name. For example, this applies to the experimental warning that `publish-extension` prints today.

Legacy names are phased out gradually.

**Announcement.** The release that introduces the new commands doesn't change the behavior of the legacy names. They produce no warnings and remain in help, labeled `[Deprecated] Use 'bicep param compile'.` We will pin an issue in [Azure/bicep](https://github.com/Azure/bicep/issues) that lists each legacy name and its replacement, describes the timeline, and collects feedback. Documentation, samples, and the CLI's own messages switch to the new names. This phase lasts at least one full release.

**Warnings.** No earlier than the next release, and no later than the last 0.x release before 1.0, legacy names are hidden from help. Each invocation of a legacy name prints a single line to stderr:

```text
WARNING: 'bicep build-params' is deprecated. Use 'bicep param compile' instead. See https://aka.ms/bicep/cli-commands.
```

The warning is written to stderr. It doesn't change the exit code and isn't counted as a diagnostic.

The warning is suppressed when `--stdout` is specified. Callers that use `--stdout` capture the command's output, and they often merge stderr into it (for example, with `2>&1`). They may not be prepared for an extra line that isn't part of the output. These callers are told about the deprecation through the pinned issue, the release notes, and help, rather than at run time.

**Removal.** Legacy names are removed in 1.0. Bicep 1.0 is still far away, and it is the natural point for breaking changes. Shipping 1.0 with two names for the same commands would undo much of the cleanup this REP is meant to achieve. The announcement and warning phases together give users at least two releases, and in practice many more, to migrate before then.

`build` and `param build` are aliases, not legacy names. They never produce a warning and aren't removed in 1.0.

#### First-party callers

Azure CLI, Azure PowerShell, Azure Developer CLI, and the Bicep MSBuild task all invoke the Bicep CLI. Any of them may run an older Bicep version: one installed with `az bicep install --version`, one found on `PATH`, or one selected by a version constraint ([REP 0011](./0011-bicep-version-pinning.md)).

Most of these invocations use `build`, which continues to work indefinitely. For the other commands, callers should use the new names only when the running Bicep version supports them. They should switch before warnings are enabled, so that users never see warnings they can't act on, and they must switch before 1.0.

Whether `az bicep` reorganizes its own commands is up to the Azure CLI team. If it does, the names carry over directly: `az bicep param compile` and `az bicep extension publish` already follow Azure CLI's singular-group convention.

#### Language server and JSON-RPC

There are no changes. The language server and VS Code extension don't invoke the CLI, and JSON-RPC method names are a separate contract.

### Server side changes (if applicable)

None.

### Microsoft.Resources/deployments API changes (if applicable)

None.

### Examples

Compiling a Bicep file. The two commands are equivalent, and neither produces a warning:

```text
bicep compile ./main.bicep --outdir ./out
bicep build ./main.bicep --outdir ./out
```

Compiling a parameters file. The two commands are equivalent, and neither produces a warning:

```text
bicep param compile ./main.bicepparam --outfile ./out/main.parameters.json
bicep param build ./main.bicepparam --outfile ./out/main.parameters.json
```

Generating a parameters file from a Bicep file:

```text
bicep param generate ./main.bicep --output-format bicepparam --include-params all
```

Invoking a legacy name after warnings are enabled. The warning is printed to stderr, but the output file and exit code are unchanged:

```text
$ bicep build-params ./main.bicepparam --outfile ./out/main.parameters.json
WARNING: 'bicep build-params' is deprecated. Use 'bicep param compile' instead. See https://aka.ms/bicep/cli-commands.
$ echo $?
0
```

With `--stdout`, no warning is printed, and the output is exactly what it was before:

```text
$ bicep build-params ./main.bicepparam --stdout
{ ... }
```

Publishing an extension:

```text
bicep extension publish ./index.json --target br:contoso.azurecr.io/extensions/foo:1.0.0 --bin-linux-x64 ./bin/linux-x64/foo
```

Generating documentation for every module in a repository:

```text
bicep doc generate --pattern './modules/**/main.bicep'
```

## Tradeoffs

- **Two names during the transition.** Until 1.0, we need to keep both names working, and documentation, blog posts, and community answers will continue to reference the legacy names after that. Because legacy names reuse the new commands' implementation, the maintenance cost is low, and both help and warnings point users to the new names.
- **`build` and `compile` will coexist permanently.** This is the cost of renaming the most widely used command without breaking existing users. Our documentation will use `compile` consistently.
- **Singular names differ from common usage.** Users may type `bicep docs` or `bicep params` out of habit. System.CommandLine suggests the closest match for unknown commands, and `docs` remains available as a hidden legacy group. We believe a single rule without exceptions is worth the occasional mistyped command.
- **Warnings on stderr can affect some scripts**, such as Windows PowerShell 5.1 scripts that use `2>&1` with `$ErrorActionPreference = 'Stop'`. Bicep already writes diagnostics and experimental warnings to stderr, so this behavior isn't new. The warning is suppressed with `--stdout`, the announcement phase gives first-party tools time to migrate, and `build` never warns.
- **Removing the legacy names in 1.0 is a breaking change.** Scripts that still use them will fail after upgrading to 1.0, and scripts that target both pre-announcement versions and 1.0 can't use a single name for these commands. The `--stdout` exemption means some callers never see a run-time warning. Mitigation: the long warning period before 1.0, the pinned issue, and release notes. `build`, the most widely used command, isn't affected.

## Alternatives

- **Keep the flat structure.** Each new feature would add another top-level verb, and the problems described in [Motivation](#motivation) would continue to grow.
- **Use plural group names** (`params`, `docs`, `extensions`, `dependencies`). Some CLIs use plural nouns, notably `gcloud` (`gcloud compute instances`) and kubectl resource types (`kubectl get pods`). Those tools name collections of cloud resources, while our groups name kinds of artifacts and feature areas, which most developer tools name in the singular. More importantly, plural names would conflict with Azure CLI's convention if `az bicep` mirrors our groups (`az bicep params`).
- **Mix plural and singular names** (`docs`, `params`, `deps`, but `extension`). This matches everyday usage, but it provides no rule for naming the next group. See [Why singular](#why-singular).
- **Name the parameters group `bicepparam`.** This names the file type exactly, but `bicep bicepparam compile` repeats the tool name and is long to type.
- **Group all commands, including the core commands** (`bicep file compile`, `bicep module publish`). This makes the most common commands longer without making them clearer.
- **Select the behavior based on the file extension** (`bicep compile main.bicepparam`, `bicep decompile main.parameters.json`). This handles `build-params`, and `decompile-params` only if Bicep can reliably tell a parameters JSON file from an ARM template. It doesn't solve `generate-params` at all. That command also takes a `.bicep` file, the same as `compile`, so the extension can't select it. It would have to stay a hyphenated top-level command or become a flag on `compile` that changes what the command produces. A single command would also need options that apply only to some inputs, such as `--bicep-file`.
- **Keep the legacy names indefinitely.** This avoids any breaking change, but 1.0 would then carry two names for the same commands, which works against the goal of a clean 1.0 command surface. Breaking changes are expected at a major version, and 1.0 is the natural point for this one.

## Rollout plan

1. Ship the new commands and the hidden legacy commands without a feature flag. The new commands reuse existing implementations, so each keeps the status of the command it replaces: `param` is GA, while `extension publish`, `doc generate`, and `local deploy` remain experimental. No ARM feature flag is required.
2. **Announcement release.** Pin the GitHub issue. Update CLI help, Microsoft Learn documentation, samples, release notes, and REP 0025. Notify the Azure CLI, Azure PowerShell, Azure Developer CLI, and MSBuild task owners so that they can adopt the new names, gated on the Bicep version. Legacy names don't produce warnings in this release.
3. **Warning release.** At least one release later, and no later than the last 0.x release before 1.0, hide legacy names from help and enable warnings. Announce this one release in advance in the pinned issue, the release notes, and the community call. If feedback on the pinned issue reveals significant problems, this phase will be delayed.
4. **1.0.** Remove the legacy names. Call out the removal in the 1.0 release notes and the pinned issue.

Commands added after this REP, including those proposed in #28, follow these rules from the start and don't need legacy names.

## Unresolved questions

- **Should users be able to suppress the warning?** An environment variable, or printing warnings only when stderr is a terminal, would reduce noise in CI. It would also make it more likely that pipelines never migrate.
- **How should the Azure deployment commands be named?** `deploy`, `what-if`, and `teardown` are experimental and still being designed ([Azure/bicep#17949](https://github.com/Azure/bicep/issues/17949)). They must follow these rules before reaching GA. They could stay top-level or move into a group, and their names should line up with the `local` group's subcommands. A future `stack` group is one possibility (see [Out of scope](#out-of-scope)). Because these commands are experimental, they can be renamed without a deprecation period.
- **Should `module publish` be added** alongside top-level `publish`, for symmetry with `extension publish`?
- **Should `restore` also be available as `dependency restore`?** `helm dependency build` performs a similar operation.

## Out of scope

- Renaming options. Options have their own inconsistencies, such as `--outfile` and `--outdir` versus `--output-file`, the different meanings of `--output-format` in `param generate` and #28, and `--format` on the deployment commands. Rule 7 intentionally leaves options unchanged so that this REP remains a pure rename. Option naming can be addressed in a separate proposal.
- The command structure of `az bicep`, Azure PowerShell, and Azure Developer CLI.
- JSON-RPC method names and language server protocol extensions.
- The design of the `dependency` commands, which is covered by #28.
- A future `stack` command group for deployment stacks. `teardown` only works with deployment stacks today, so stack operations may eventually fit better in their own group than as top-level deployment commands. That decision belongs with the deployment commands design and isn't made here.
