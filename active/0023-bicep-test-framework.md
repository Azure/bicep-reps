---
REP Number: "0023"
Author: polatengin (Engin Polat)
Start Date: 2026-07-30
Feature Status: Private Preview
Bicep Issue Number(s): "[#11966](https://github.com/Azure/bicep/issues/11966), [#11967](https://github.com/Azure/bicep/issues/11967)"
---

# Bicep test framework

## Summary

Bicep users need a native, repeatable way to test modules before deployment, verify expected failures, and publish actionable results in CI. This REP proposes a Bicep-native test framework where Bicep is the primary test authoring language and `bicep test` is the built-in runner.

The rollout is phased. P0 and P1 focus on offline module tests that evaluate predicted behavior using real `.bicepparam` inputs, assertions, expected failures, mocks, and report formats. P2 adds Azure-backed deployment and post-deployment testing for real control-plane and data-plane outcomes. P3 adds optional integrations and helper libraries for external frameworks that call `bicep test`.

The framework remains experimental. Scope and contracts are staged to deliver immediate value for module authoring while reducing long-term compatibility and safety risks.

## Terms and definitions

- **Bicep test:** A named test case authored in Bicep syntax and executed by `bicep test`.
- **Offline test:** A test that evaluates predicted behavior locally and does not call Azure or create resources.
- **Azure-connected no-create workflow:** A workflow that calls Azure control-plane operations such as validation or what-if without creating target resources.
- **Online/deployment-backed test:** A test mode that can deploy Azure resources and evaluate post-deployment outcomes.
- **Snapshot mode:** Offline test mode for predicted resource and output evaluation, including regression comparison patterns.
- **Deploy mode:** Online test mode that can create real Azure resources and run deployment-backed checks.
- **Post-deployment check:** A test check that evaluates deployed resource state or runtime behavior after deployment completes.
- **Expected failure:** A test that passes only when the intended failure condition occurs, optionally with diagnostic matching.
- **Mock:** A fixed substitute for selected external boundaries such as deployment context and, later, richer dependency surfaces.
- **External test framework:** A non-Bicep test runner (for example, Jest, pytest, Pester, xUnit) that may invoke `bicep test`; it is not an alternative built-in Bicep runner.

## Motivation

### Testing methods and scope

Testing terms often overlap, so each test is grouped by what it checks.

| Method | What it checks | Coverage in this proposal |
| ------ | -------------- | --------------------- |
| Static validation | Whether Bicep code parses, passes type checks, and follows linter rules. Existing commands include `bicep build` and `bicep lint`. | **Already exists; not redesigned here.** Tests may check expected diagnostics, but they do not replace build or lint. |
| Offline module testing, also called unit or component testing | How one module, or a small set of modules, behaves for known parameters without creating Azure resources. This includes checks of predicted results and fixed deployment-context values. | **Main scope in P0-P1.** Includes parameter scenarios, assertions, expected failures, mocks, discovery, filtering, and CI results. |
| Negative testing | Whether bad inputs, `fail()` expressions, and module rules fail as expected. | **Main scope in P0.** Tests can expect a failure and may match specific diagnostics. |
| Snapshot and regression testing | Whether predicted resources or other visible results change for a known set of parameters. | **Covered in P1.** Snapshot mode is planned for P1. Clear rules for storing, updating, and reviewing baselines can be added later. |
| Contract testing | Whether a module keeps its promises about inputs, validation, outputs, and predicted resources. | **Covered in P0-P1.** Assertions, negative tests, and snapshots provide the basics. This proposal does not define compatibility or versioning rules. |
| Integration testing | Whether one or more modules deploy and work with real Azure control-plane services. | **Planned for P2.** Uses online `deploy` mode and deployment tests. |
| End-to-end testing | Whether a full infrastructure flow works across modules and steps, including order, waits, retries, and final checks. | **Planned for P2.** P2 adds multi-step tests, Azure deployments, and checks after deployment. |
| Post-deployment, smoke, and acceptance testing | Whether deployed resources are healthy and behave as expected. | **Planned for P2.** The exact checks still need a design. |
| Pre-deployment Azure validation | Whether Azure accepts a deployment and what live changes it would make, using validation or `what-if`. | **Outside this proposal.** These remain separate workflows. |
| Policy, security, and compliance testing | Whether infrastructure follows company rules and avoids known security problems, using tools such as the Bicep linter, Azure Policy, or external scanners. | **Outside this proposal.** Module assertions can check small project rules, but this framework is not a policy or security engine. |
| Operational and lifecycle testing | Performance, load, resilience, chaos, disaster recovery, drift, and repeat-deployment behavior. | **Outside this proposal.** External test runners may call these tools, but Bicep will not build them. |

### Customer scenarios

These scenarios define customer needs across offline evaluation and Azure-backed validation and deployment checks. Offline tests before deployment mainly drive P0 and P1. Richer mocks, Azure deployments, and multi-step tests drive P2. P3 supports scenarios that need another test framework or programming language.

Azure-connected no-create tests remain separate workflows in this REP. Azure validation and `what-if` remain outside the core test modes. Some specialized checks after deployment may use external tools instead of built-in Bicep test features.

Infrastructure testing is grouped into two main phases:

1. **Pre-deployment testing** checks infrastructure before the target resources are created or updated.
2. **Post-deployment testing** checks real infrastructure after it has been deployed.

| Infrastructure testing phase | Terraform test analogue | What it proves |
| --- | --- | --- |
| Pre-deployment testing | A `run` block with `command = plan` | The configuration can be evaluated and its planned values satisfy expectations without creating target resources |
| Post-deployment testing | A `run` block with `command = apply` | Real resources can be created, their resulting state or behavior satisfies expectations, and test resources can be cleaned up |

Bicep pre-deployment testing is comparable to Terraform test plan mode. Bicep post-deployment testing is comparable to Terraform test apply mode, where short-lived infrastructure is deployed, checked, and cleaned up.

This is a general comparison, not an exact match:

- Terraform plan mode does not create resources, but it may still call cloud APIs through a real provider. Mock providers support isolated plan tests.
- Terraform apply mode includes deployment and checks against the resulting state. It is deployment-backed testing with post-deployment checks.
- Some post-deployment checks, such as signed-in HTTP requests or protocol-specific probes, may need features outside Terraform or Bicep's declarative language.

#### Scenario summary

| ID | Customer scenario | Phase | Execution profile |
| --- | --- | --- | --- |
| PRE-01 | Verify parameter-driven module variants | Pre-deployment | Offline |
| PRE-02 | Reuse real deployment parameter files | Pre-deployment | Offline |
| PRE-03 | Verify loops, conditions, and resource topology | Pre-deployment | Offline |
| PRE-04 | Protect module inputs, outputs, and consumer contracts | Pre-deployment | Offline |
| PRE-05 | Verify invalid inputs fail as intended | Pre-deployment | Offline |
| PRE-06 | Enforce secure and organizational defaults | Pre-deployment | Offline |
| PRE-07 | Test deterministic deployment-context behavior | Pre-deployment | Offline |
| PRE-08 | Isolate existing resources and runtime dependencies | Pre-deployment | Offline with mocks or overrides |
| PRE-09 | Detect unintended deployment-shape regressions | Pre-deployment | Offline |
| PRE-10 | Verify composition across multiple modules | Pre-deployment | Offline |
| PRE-11 | Check whether Azure accepts a deployment | Pre-deployment | Azure-connected, no-create |
| PRE-12 | Check changes against an existing environment | Pre-deployment | Azure-connected, no-create |
| PRE-13 | Check policy and deployment permissions | Pre-deployment | Azure-connected, no-create |
| POST-01 | Prove that a module deploys successfully | Post-deployment | Azure-deployed |
| POST-02 | Verify provider-computed values and final resource state | Post-deployment | Azure-deployed |
| POST-03 | Verify managed identity and RBAC behavior | Post-deployment | Azure-deployed |
| POST-04 | Verify data-plane or application behavior | Post-deployment | Azure-deployed |
| POST-05 | Verify private networking and DNS integration | Post-deployment | Azure-deployed |
| POST-06 | Verify integration between deployed services | Post-deployment | Azure-deployed |
| POST-07 | Verify update, replacement, and idempotency behavior | Post-deployment | Azure-deployed |
| POST-08 | Verify multi-step deployments and eventual consistency | Post-deployment | Azure-deployed |
| POST-09 | Verify an Azure-side failure is expected | Post-deployment | Azure-deployed |
| POST-10 | Verify monitoring, backup, and operational readiness | Post-deployment | Azure-deployed |

#### Pre-deployment testing scenarios

Pre-deployment tests answer: **What should this infrastructure definition do, and is it safe to deploy?**

Most pre-deployment tests should run on every pull request. Offline tests should be fast, repeatable, and need no Azure credentials. Azure-connected tests are needed only when the answer depends on the target environment or Azure control plane.

Each scenario describes the customer situation, an example, expected behavior, and the reason for pre-deployment testing.

##### PRE-01: Verify parameter-driven module variants

**Customer situation:** A platform engineer maintains a reusable web application module for development and production.

**Example:** The module accepts `environment`, `sku`, `zoneRedundant`, and `enablePrivateEndpoint` parameters.

**The test should verify that:**

- The development setup uses the intended low-cost SKU.
- The production setup enables zone redundancy.
- Private endpoint and DNS resources appear only when requested.
- Every production setup disables public network access.
- Changing one option does not change unrelated resources.

**Why test before deployment:** These results come from source code and parameter values. They should be available in seconds without Azure credentials.

##### PRE-02: Reuse real deployment parameter files

**Customer situation:** The application team maintains `dev.bicepparam`, `test.bicepparam`, and `prod.bicepparam` files for the deployment pipeline.

**Example:** The production file selects a larger database SKU, enables backups, and uses two regions. The development file uses one region and a smaller SKU.

**The test should verify that:**

- Every checked-in parameter file works with the current module version.
- Each environment produces the expected resource count, locations, and SKUs.
- Production-only resources do not appear in development.
- Refactoring a module does not silently change an existing parameter file.
- Parameter files can use clear, non-secret test values for external inputs.

**Why test before deployment:** The parameter files describe real deployments, but their predicted results can be checked without creating resources.

##### PRE-03: Verify loops, conditions, and resource topology

**Customer situation:** The networking team creates subnets, network security groups, route-table associations, and private DNS links from arrays and objects.

**Example:** One parameter defines four subnets. Two need network security groups, and one needs a route table.

**The test should verify that:**

- Exactly four subnets are predicted.
- Only the selected subnets receive network security groups.
- Every generated association refers to the correct subnet.
- Empty optional collections create no resources.
- Resource names stay unique when loop inputs are reordered.
- Nested modules produce the expected flattened deployment topology.

**Why test before deployment:** Risk is in expressions, loops, and conditions. The predicted resource graph provides needed evidence.

##### PRE-04: Protect module inputs, outputs, and consumer contracts

**Customer situation:** A central platform team publishes versioned modules to an internal registry used by many application teams.

**Example:** The storage module returns `resourceId`, `blobEndpoint`, and `identityPrincipalId`. It also accepts a stable set of environment parameters.

**The test should verify that:**

- The required inputs and accepted shapes stay compatible.
- The documented defaults keep the same behavior.
- Important outputs remain and come from the intended resources.
- Resource IDs contain the correct subscription, resource group, type, and name.
- Refactoring nested modules does not break the public module contract.
- Ordinary outputs do not expose secure values.

**Why test before deployment:** Most contract behavior is visible in compilation results, predicted resources, and predicted outputs. Values assigned by Azure are tested after deployment.

##### PRE-05: Verify invalid inputs fail as intended

**Customer situation:** Modules use validation and `fail()` expressions to reject unsupported configurations.

**Example:** The module rejects an unsupported SKU, a production deployment without zone redundancy, or overlapping subnet prefixes.

**The test should verify that:**

- The valid input passes.
- Each representative invalid input fails.
- Each failure comes from the intended validation, not an unrelated error.
- The diagnostic identifies the input and gives a useful message.
- A later edit cannot silently remove or weaken the validation.

The framework must distinguish an expected module failure from malformed source, type errors, missing files, and internal evaluation failures. Terraform does something similar by limiting `expect_failures` to user-defined conditions.

**Why test before deployment:** Invalid configurations should fail before sign-in or resource creation.

##### PRE-06: Enforce secure and organizational defaults

**Customer situation:** A cloud center of excellence maintains approved modules with secure defaults.

**Example:** Storage accounts disable public blob access. Key Vaults use RBAC. Web applications require HTTPS, and resources have ownership tags.

**The test should verify that:**

- Modules stay secure when callers leave out optional parameters.
- Modules reject an opt-out or produce a clearly different result.
- Every relevant predicted resource has required tags and diagnostic settings.
- Minimum TLS and network-access settings stay correct.
- A module update does not weaken secure defaults.

**Why test before deployment:** These assertions check the goal state in the module. They support Azure Policy and security scanning but do not replace organization-wide enforcement.

##### PRE-07: Test deterministic deployment-context behavior

**Customer situation:** The module gets names, locations, resource IDs, and tags from deployment context functions.

**Example:** A resource uses the resource group's location, includes part of the subscription ID in its name, and stores the deployment name in a tag.

**The test should verify these contexts:**

- A resource group named `rg-app-test` in `westus3`.
- Known subscription and tenant IDs.
- A subscription-scope deployment in `eastus`.
- A fixed deployment name used for audit tags.

Teams need the same result on local machines and in CI.

**Why test before deployment:** Fixed deployment metadata can be provided as test inputs. Bicep snapshot already supports tenant, subscription, management group, resource group, location, and deployment-name inputs.

##### PRE-08: Isolate existing resources and runtime dependencies

**Customer situation:** The spoke-network module reads an existing hub network. The application module uses outputs from an identity module managed by another team.

**Example:** The module chooses a branch from an existing virtual network's address space. It can also build settings from another module's outputs.

**The test should verify that:**

- A known existing resource selects the correct branch.
- The module handles missing or unusual dependency values correctly.
- A dependent module receives expected resource IDs and endpoints.
- Tests do not need production subscriptions or secrets.
- Tests replace only the external boundary and still evaluate module logic.

**Why test before deployment:** This scenario needs fixed substitutes for selected existing resources, runtime function results, or module outputs. Terraform supports similar cases with mock providers and targeted overrides.

##### PRE-09: Detect unintended deployment-shape regressions

**Customer situation:** A module author is refactoring nested modules and needs proof that resulting infrastructure has not changed.

**Example:** The module owner moves resources between files and simplifies loops without intending to change the deployment.

**The test should verify that:**

- The normalized predicted resource set is unchanged.
- Resource types, names, scopes, dependencies, and important properties are unchanged.
- Outputs remain unchanged.
- Deliberate changes produce a focused baseline diff that is reviewable.
- Metadata changes do not produce noisy diffs.

**Why test before deployment:** Snapshots compare predicted state. The existing `snapshot --mode overwrite` and `snapshot --mode validate` workflow already covers much of this scenario.

##### PRE-10: Verify composition across multiple modules

**Customer situation:** A platform team maintains a solution that combines identity, networking, data, and application modules.

**Example:** The team passes the identity principal ID to role assignments, the subnet ID to the application, and the data endpoint to application settings.

**The test should verify that:**

- Each module output connects to the correct input.
- Resources use intended scopes.
- Optional modules are included only in correct scenarios.
- Final predicted resources use consistent IDs, locations, and names.
- A module contract change gives a useful failure at the composition boundary.

**Why test before deployment:** Module composition and parameter wiring can usually be checked from the complete predicted deployment.

##### PRE-11: Check whether Azure accepts a deployment

**Execution profile:** Azure-connected, no-create.

**Customer situation:** A module uses a new API version or an uncommon combination of resource properties.

**Example:** A SKU may not be available in the target region. A resource provider may enforce a rule that Bicep types do not include.

**The test should verify that:**

- Azure deployment validation accepts generated template and parameters.
- API versions and resource types are available at target scope.
- Provider rules do not reject configuration.
- Errors point to relevant Bicep source and test scenario.

**Why test before deployment:** Current Azure control-plane behavior is needed, but no resources need to be created.

##### PRE-12: Check changes against an existing environment

**Execution profile:** Azure-connected, no-create.

**Customer situation:** The application team needs to understand a module update before changing a long-lived environment.

**Example:** A database module update should add diagnostic settings. It must not replace the server, delete databases, or disable zone redundancy.

**The test should verify that:**

- What-if reports no unexpected deletes or replacements.
- Only approved properties change.
- Resources outside the deployment remain untouched.
- A no-op redeployment produces no material changes.
- The result can be reviewed and enforced in CI.

**Why test before deployment:** This comparison needs live environment state, but it can run before applying the change.

##### PRE-13: Check policy and deployment permissions

**Execution profile:** Azure-connected, no-create.

**Customer situation:** The platform team deploys with a restricted CI identity into subscriptions governed by Azure Policy.

**Example:** The identity can deploy only to approved regions and may not be able to create role assignments.

**The test should verify that:**

- Current policies do not block intended deployment.
- Deployment identity has required permissions at every scope.
- A denial identifies the relevant policy or missing action.
- Infrastructure is not created only to discover a permission problem.

**Why test before deployment:** Policy and RBAC assignments are live Azure state. They should be validated before creating resources whenever Azure can provide needed evidence.

#### Post-deployment testing scenarios

Post-deployment tests answer: **Did Azure create intended infrastructure, and does it work in the real environment?**

These tests can be slow, cost money, hit service limits, and change real environments. Each run must show target tenant, subscription, scope, resource owner, and cleanup behavior.

##### POST-01: Prove that a module deploys successfully

**Customer situation:** A module publishing team needs release confidence across supported scopes and configurations.

**Example:** The team deploys the Key Vault module to a dedicated test subscription and uses both a minimal setup and a production-like setup.

**The test should verify that:**

- Azure completes deployment successfully.
- Azure creates expected resources at intended scope.
- Deployment outputs are available and have correct shape.
- Supported regions and API combinations work.
- Test resources stay separate from production and are removed afterward.

**Why test after deployment:** Only a real deployment proves that Azure and its resource providers can create the infrastructure.

##### POST-02: Verify provider-computed values and final resource state

**Customer situation:** A module depends on values assigned or normalized by Azure.

**Example:** The scenario needs a real managed identity principal ID, service hostname, provisioning state, or provider default value.

**The test should verify that:**

- Resources reach a successful final state.
- Provider-computed IDs and endpoints are not empty and work.
- Azure final resource settings match module intent.
- Provider defaults remain compatible with module use.
- Outputs contain correct final values.

**Why test after deployment:** These values cannot be obtained, or trusted as final, until provider creation completes.

##### POST-03: Verify managed identity and RBAC behavior

**Customer situation:** The application module creates a managed identity and grants least-privilege access to Key Vault or Storage.

**Example:** The identity should read one secret but must not write or delete secrets.

**The test should verify that:**

- Expected role assignment uses the correct principal and scope.
- Identity can perform the allowed operation.
- Identity cannot perform a forbidden operation.
- Tests account for identity and role-assignment propagation delays.
- Failures distinguish bad configuration from propagation delay.

**Why test after deployment:** Real identities, role assignments, access tokens, and service APIs are required to test authorization.

##### POST-04: Verify data-plane or application behavior

**Customer situation:** The application team needs to know whether deployed infrastructure works, not only whether Azure created it.

**Example:** The web application should return a healthy HTTPS response, require sign-in for a protected endpoint, and reject plain HTTP. The storage account should reject anonymous blob access.

**The test should verify that:**

- Health endpoint returns expected status and response.
- TLS, hostname, and sign-in behavior are correct.
- Services block public access where required.
- Intended identity can consume a queue or topic message.
- System emits diagnostic signals after a representative request.

**Why test after deployment:** Real data-plane protocols and application behavior must be tested. This may require HTTP clients, sign-in, response parsing, retries, and protocol-specific libraries. An established test framework can still run these checks.

##### POST-05: Verify private networking and DNS integration

**Customer situation:** Enterprise services are deployed behind private endpoints in a hub-and-spoke network.

**Example:** A test agent runs in a spoke network and must resolve a service hostname to a private address and connect without a public endpoint.

**The test should verify that:**

- Private DNS records link to correct virtual networks.
- Name resolution returns expected private address from an approved network.
- Service is reachable through private endpoint.
- Public endpoint is unreachable, or disabled.
- Network security groups and routes allow only intended path.

**Why test after deployment:** This scenario needs real resources and an agent in the right network to validate DNS, routing, firewalls, and private endpoints.

##### POST-06: Verify integration between deployed services

**Customer situation:** A solution combines application, managed identity, message broker, database, and monitoring resources.

**Example:** The application should read a secret, connect to the database, publish a message, and emit telemetry without storing credentials in settings.

**The test should verify that:**

- Resource endpoints and IDs are connected correctly.
- Managed identities can sign in to each dependency.
- A representative transaction crosses expected services.
- Failures produce logs and metrics in the chosen destination.
- Unintended identity or network path cannot reach services.

**Why test after deployment:** This verifies how real Azure services work together, not only predicted settings.

##### POST-07: Verify update, replacement, and idempotency behavior

**Customer situation:** A service team with a stateful service needs confidence that module updates are not destructive.

**Example:** The team deploys version A of database settings, updates to version B, and deploys version B again without changes.

**The test should verify that:**

- The update preserves stable resource IDs and data-holding resources.
- Only intended properties change.
- Redeploying same settings makes no changes.
- Azure deletes removed optional resources only when expected.
- Incompatible change provides clear warning or expected replacement.

**Why test after deployment:** Lifecycle behavior depends on existing resources and provider update behavior. One predicted snapshot cannot prove data preservation or idempotency.

##### POST-08: Verify multi-step deployments and eventual consistency

**Customer situation:** A solution must be deployed in stages because later resources depend on identities, DNS records, or provider registrations that need time.

**Example:** The team deploys an identity, waits for RBAC propagation, deploys an application that uses the identity, and then checks service access.

**The test should verify that:**

- Steps run in required order.
- Setup-step outputs can be passed to later deployments.
- Workflow retries temporary Azure failures within clear limits.
- Each wait has a reason, timeout, and diagnostic output.
- Cleanup runs in reverse order without breaking dependencies.

**Why test after deployment:** Real systems that take time to settle require coordinated retries, timeouts, cancellation, state, and cleanup.

##### POST-09: Verify an Azure-side failure is expected

**Customer situation:** The platform team needs to prove that Azure Policy or a resource provider rejects a prohibited configuration.

**Example:** Azure should deny public network access in a governed test subscription. An unsupported regional SKU should fail with a known error type.

**The test should verify that:**

- Operation fails for intended policy or provider reason.
- Sign-in, timeout, or malformed-template errors do not make the test pass.
- Results include Azure correlation and diagnostics.
- No partial resources remain after expected failure.

**Why test after deployment:** This behavior comes from live policy and provider state. PRE-11 or PRE-13 is preferred when validation provides equivalent evidence without resource creation.

##### POST-10: Verify monitoring, backup, and operational readiness

**Customer situation:** A service team must prove new infrastructure is operations-ready before handoff.

**Example:** The database must have backups and retention. Critical resources must emit diagnostics, and alerts must reach the expected action group.

**The test should verify that:**

- Diagnostic settings send logs and metrics to intended destinations.
- Representative failure or request triggers expected alert path.
- Backup policies are active and a recovery point is visible.
- Resource health and availability checks become healthy.
- Operational outputs, dashboards, and ownership details are available.

**Why test after deployment:** Predicted settings show intent, but only deployed checks prove operational connections. Full restore, disaster recovery, load, and chaos testing may remain outside the core framework.

#### Cross-cutting customer expectations

Across these scenarios, customers need the same outcomes:

- **Visible execution profile:** Tests must show whether a scenario is offline, Azure-connected no-create, or can create billable resources.
- **Real parameter reuse:** Teams need to use the same `.bicepparam` scenarios for tests and deployments when practical.
- **Useful results:** Each failure must show test, assertion or operation, source location, expected result, actual result, and diagnostics.
- **Precise expected failures:** Only intended failure should make a negative test pass.
- **Repeatable offline results:** Offline results must be stable across computers and runs.
- **Isolation:** Deployment-backed tests need dedicated scopes, unique names, and test-specific state.
- **Cleanup:** Created resources must be tracked and removed in safe order, with reports for anything left behind.
- **Safety:** Deployment-backed tests should require explicit approval and show tenant, subscription, scope, cost, and destructive actions before execution.
- **Secret handling:** Credentials and secure values must not be stored in test files or results.
- **Selective execution:** Users need filters for file, test name, tag, phase, execution profile, and changed module.
- **Controlled parallelism:** Independent scenarios should run together without shared names or state and without exceeding Azure or cost limits.
- **CI integration:** Output must be readable and available in standard test and diagnostic formats.
- **Cancellation and timeout:** Cancellation must be predictable for long deployments, retries, probes, and cleanup.

#### Representative customer journeys

##### Module pull request

A module author changes conditional resource logic in a module. CI runs static checks, offline parameter scenarios, expected-failure tests, and snapshot comparisons. The pull request remains fast and does not need Azure credentials. A smaller Azure-connected validation suite runs only for new API versions or provider behavior.

##### Module release

A release workflow publishes a registry module. The workflow runs a full pre-deployment contract suite, deploys representative setups to a dedicated subscription, checks final resources, and publishes standard test results. Cleanup is part of release gating, and leaked resources fail the run.

##### Application platform change

An application team tests a production `.bicepparam` file offline, runs what-if against staging, deploys to an isolated test resource group, runs identity and private-network health checks, and removes the environment. Production deployment proceeds only after both phases provide expected evidence.

### Terraform test gap analysis

This analysis reviews Terraform `v1.15.x` docs and compares `terraform test` with this Bicep plan. Terraform added native tests in version 1.6 and provider mocks in version 1.7.

Comparison focuses on user capability rather than syntax parity. Terraform providers, state, and HCP Terraform do not map directly to ARM, so analysis emphasizes equivalent Bicep safety and control.

Sources: [Terraform tests](https://developer.hashicorp.com/terraform/language/tests), [mocking and overrides](https://developer.hashicorp.com/terraform/language/tests/mocking), [`terraform test` command](https://developer.hashicorp.com/terraform/cli/commands/test), and [machine-readable UI output](https://developer.hashicorp.com/terraform/internals/machine-readable-ui).

#### What Terraform Supports

- **Files and discovery:** Terraform discovers `.tftest.hcl` and `.tftest.json` files in the root folder and the `tests` folder. `-test-directory` selects another test folder. `-filter` selects test files.
- **Tests and runs:** A test file can have settings for the whole file and one or more `run` blocks. Each run can set `command`, `plan_options`, `variables`, `module`, `providers`, `assert`, `expect_failures`, `state_key`, and `parallel`.
- **Plan and apply controls:** A run uses `apply` by default. It can use `plan` to avoid creating resources. Plan options control refresh, replacement, resource targeting, and normal or refresh-only mode.
- **Inputs and modules:** Variables can come from file settings, run settings, CLI arguments, variable files, or earlier run outputs. Terraform defines which source wins. Tests can use other local or registry modules for setup and checks.
- **Assertions and expected failures:** Assertions can inspect configuration, plans, state, outputs, and earlier runs. `expect_failures` only accepts named, user-defined condition failures. It does not accept unrelated syntax, type, provider, or runtime errors.
- **Mocks and overrides:** A test can mix real and mock providers. Mocks can create computed values and set defaults for resources and data sources. A file or run can override resources, data sources, and module outputs. Reusable `.tfmock.hcl` and `.tfmock.json` files can share mock data.
- **State and cleanup:** Each test file starts with its own in-memory state. `state_key` lets modules use separate or shared state. Terraform tries to destroy created resources in reverse run order and reports anything it cannot remove.
- **Order and parallel work:** Runs are sequential by default. File and run settings can allow independent runs to run at the same time. Output dependencies, shared state, and serial runs limit this. `-parallelism` limits concurrent plan and apply work.
- **Results and automation:** Terraform has human-readable output, a versioned `-json` event stream, JUnit XML, and `-verbose` plan or state details. Completed files and runs use pass, fail, error, or skip status. Separate events report cleanup failures and interruptions. The stream also reports progress and elapsed time.
- **Remote runs:** `-cloud-run` runs tests for a private registry module in HCP Terraform. This is different from a local test that uses a real cloud provider.

#### Coverage and Gaps

| Capability | Terraform today | Bicep plan | Gap |
| ---------- | --------------- | ----------- | --- |
| Test files | Uses stable `.tftest.hcl` and `.tftest.json` files. | P1 chooses `.biceptest`, `.bicep`, or both. | **Partial.** The file choice, migration path, and compatibility rules are still open. |
| Test folders and discovery | Loads tests from the root and a default or selected test folder. | P1 adds recursive discovery and include and exclude patterns. | **Partial.** Bicep does not yet define a default folder, path rules, or how to handle duplicate names. |
| Test selection | `-filter` selects test files. | P1 filters by test name and file patterns. | **Planned and may be broader.** Bicep still needs rules for filtering by file, test, tag, mode, or changed module. |
| Scenarios and runs | A file has named runs, file defaults, and run overrides. | P0 has named tests. P2 adds multiple steps. | **Partial.** The plan does not say whether each step can be selected, checked, and reported on its own. |
| Plan and apply modes | Each run uses `plan` or `apply`. `apply` is the default. | P1 defines offline `snapshot` and online `deploy`. P2 adds online runs. | **Broadly covered.** Bicep snapshot evaluates locally without calling Azure. Terraform plan may call a provider. |
| Detailed plan controls | `plan_options` controls refresh, replacement, targeting, and normal or refresh-only mode. | No matching snapshot, what-if, update, replacement, or target controls are planned. | **Gap or intentional omission.** Bicep must decide which controls make sense for ARM. |
| Parameter sources and order | Terraform defines how file variables, run variables, CLI values, environment values, `.tfvars`, automatic files, and earlier outputs override each other. | P0 supports inline parameters and plans `.bicepparam` reuse. | **Partial.** The plan does not define suite defaults, step overrides, source order, external inputs, secrets, or redaction. |
| Providers and deployment targets | Tests choose real or mock providers, aliases, and provider mappings for each run. | P0 adds deployment-context mocks. P2 adds Azure deployment tests. | **Different model, with a gap.** Bicep does not use provider plugins, but it still needs per-test rules for tenant, subscription, scope, identity, credentials, and cloud environment. |
| Assertions | Assertions can check configuration, outputs, plan or state, and earlier run outputs. | Target modules already have Boolean assertions. P0 improves failure messages. | **Partial.** The plan does not define test-local assertions for one scenario, one step, or values from earlier steps. |
| Expected failures | `expect_failures` names allowed condition failures and follows plan and apply timing rules. | P0 adds expected failures and optional diagnostic matching. | **Broadly covered.** Bicep still needs exact failure types, phases, and rules that reject unrelated failures. |
| Setup and check modules | A run can use another local or registry module for setup or checks. | P2 steps can run modules. | **Partial.** The plan does not define setup, subject, check, and cleanup roles or module source rules. |
| Data between runs | Later runs can use earlier outputs. These references also set run order. | P2 shows ordered steps but no syntax for passing outputs. | **Gap.** Steps need typed outputs, input binding, missing-value behavior, and dependency rules. |
| State and ownership | Each file starts with separate in-memory state. `state_key` controls state sharing. | Offline snapshots have no state. P2 does not define ownership for online tests. | **Gap for deploy mode.** Bicep needs deployment tracking, isolation, and sharing rules even though ARM does not use Terraform state. |
| Automatic cleanup | Terraform tries to destroy resources in reverse run order and reports resources left behind. | P2 only mentions warnings and safety checks for destructive or billable work. | **Major gap.** The plan does not require automatic cleanup, cleanup after failure, leak reports, or a cleanup-failed status. |
| Mock providers and generated values | Schema-aware mocks create computed values and can be mixed with real providers in one run. | P0 adds fixed deployment-context values. P2 may mock functions, existing resources, and outputs. | **Partial, with a large gap.** The plan does not define generated values, real and mock boundaries, step-level choices, or unknown values. |
| Targeted and reusable mocks | File or run overrides replace resources, data sources, or module outputs. Mock files share defaults and define which value wins. | P2 includes richer mocks but does not design them. | **Gap.** Bicep needs target syntax, scope, source order, reusable fixtures, and shape checks. |
| Ordered steps | Runs are ordered by default and can form setup, run, and check flows. | P2 adds multiple steps, waits, and retries. | **Planned, with extra Bicep features.** Wait and retry help with Azure delays, but need timeout, backoff, and failure rules. |
| Parallel runs | File and run settings allow parallel work. Data dependencies and shared state limit it. `-parallelism` sets a maximum. | No parallel model is planned. | **Gap.** Bicep needs limits and safety rules for Azure throttling, cost, names, and isolation. |
| Human and CI reports | Includes human-readable output, JUnit XML, and detailed assertion errors. | P0 adds clear messages and SARIF. P1 adds JUnit. | **Covered, with a Bicep addition.** SARIF adds source-based diagnostics that Terraform test does not provide directly. |
| Live result events | A versioned JSON stream reports discovery, progress, details, status, cleanup, summary, and interruption. | P2 and P3 refer to CLI or RPC results but do not define events or versions. | **Gap.** Test Explorer and external runners need one stable result and progress contract. This does not require a JSON file format. |
| Plan or state details | `-verbose` prints the plan for plan runs and state for apply runs. | Snapshot shows predicted resources. Online test details are not defined. | **Partial.** Online tests need deployment results, outputs, resource ownership, and redaction rules. |
| Stop and interruption behavior | Events show pending work and resources that may remain. Skip, interruption, and cleanup results are distinct. | No timeout, cancellation, interruption, or recovery rules are planned. | **Gap.** Online tests need these rules to be safe. |
| Managed remote runs | HCP Terraform can run tests remotely for a private registry module. | P3 adds external runners and libraries, not a managed Bicep test service. | **Gap or intentional omission.** Deploying to Azure is not the same as running the test service remotely. |
| Snapshot baselines | Tests can check and print plans, but Terraform does not define a golden-file update and review flow. | P1 makes snapshot testing a test mode and notes possible baseline improvements. | **Bicep addition.** Bicep still needs rules for storage, stable output, updates, and review. |
| Editor tools | Terraform's CLI supports integrations, but these docs do not define Test Explorer or authoring fixes. | P2 adds VS Code Test Explorer, diagnostics, completions, and code actions. | **Bicep addition.** The editor should use the same runner contract as the CLI. |

### References

- [Terraform tests](https://developer.hashicorp.com/terraform/language/tests)
- [Terraform test mocking and overrides](https://developer.hashicorp.com/terraform/language/tests/mocking)
- [Terraform test command](https://developer.hashicorp.com/terraform/cli/commands/test)
- [Machine-readable UI output](https://developer.hashicorp.com/terraform/internals/machine-readable-ui)
- [Bicep snapshot command](docs/experimental/snapshot-command.md)
- [Bicep Testing Framework (#11966)](https://github.com/Azure/bicep/issues/11966)
- [Bicep Experimental Test Framework (#11967)](https://github.com/Azure/bicep/issues/11967)
- [Anthony Martin's bicep-test framework](https://github.com/anthony-c-martin/bicep-test)

## Detailed design

Design choice: Bicep is the primary and official test authoring language, and `bicep test` is the built-in runner. Optional frameworks in other languages may invoke this runner through stable results and API contracts later, but this REP does not define a second official test model.

### Authoring and execution model

Named tests are authored in Bicep syntax and point to target modules with inline parameters or `.bicepparam` reuse. Assertions and expected failures are first-class test concepts, with expected-failure behavior constrained so unrelated infrastructure or tooling failures do not incorrectly pass a test.

The same file can include multiple named tests. Discovery and filtering are planned for repository-scale execution, with filtering by test name and file patterns in P1 and broader selectors under design.

Representative syntax:

```bicep
test dev './main.bicep' = {
  params: './dev.bicepparam'
}

test invalidName './main.bicep' = {
  params: {
    name: 'INVALID_NAME'
  }
  expect: {
    failure: true
    diagnostics: [
      'BCP###'
    ]
  }
}
```

### Offline and online modes

The mode contract is explicit:

- `snapshot` is offline and evaluates predicted resources and outputs locally.
- `deploy` is online and can create real Azure resources.

P0 and P1 are client-side and offline-first. P2 introduces deployment-backed execution and post-deployment checks for scenarios that require live Azure state or runtime behavior.

Mocks start with deterministic deployment context values (`resourceGroup`, `subscription`, `tenant`, deployment name, location) to make offline runs reproducible. Richer mock boundaries are deferred to P2 and later.

Representative syntax:

```bicep
test westus './main.bicep' = {
  params: './dev.bicepparam'
  mode: 'snapshot'
  mocks: {
    resourceGroup: {
      name: 'rg-test'
      location: 'westus'
    }
    subscription: {
      subscriptionId: '00000000-0000-0000-0000-000000000000'
    }
  }
}
```

### Test results and reporting

Results must separate compile, evaluation, expected-failure, assertion, deployment, timeout, skip, and cleanup outcomes with source mapping where applicable.

Reporting format sequence:

- SARIF first for source-linked diagnostics.
- JUnit next for CI test case aggregation.
- Stable structured runner contracts for editor and external integration follow.

Representative commands:

```sh
bicep test ./tests.bicep --format sarif
bicep test ./tests.bicep --format junit
```

### Client side changes

Client-side work includes language, CLI, and tooling capabilities:

- Test declarations, expected failures, parameter reuse, mode declarations, and context mocks.
- Discovery and filtering (`--recursive`, `--filter`, include/exclude pattern support).
- Snapshot/deploy mode handling and staged orchestration concepts (`steps`, `wait`, `retry`).
- Clear diagnostics, value-aware assertion failure output, and skip vs error separation.
- VS Code integration through Test Explorer and language-server diagnostics/actions after runner contracts stabilize.

Representative orchestration syntax:

```bicep
test delayedDeployment = {
  steps: [
    {
      module: './identity.bicep'
      params: './identity.bicepparam'
    }
    {
      wait: 'PT2M'
      reason: 'Allow identity and role assignment propagation'
    }
    {
      module: './app.bicep'
      params: './app.bicepparam'
      retry: {
        maxAttempts: 5
        wait: 'PT30S'
      }
    }
  ]
}
```

### Server side changes

No dedicated Bicep service-side protocol change is proposed in P0 or P1. These phases are client-side and offline-first.

P2 depends on Azure control-plane behavior for deployment-backed scenarios. If deployment-backed testing exposes missing service contracts, the REP must be updated before stabilization to define contract and compatibility implications.

### Microsoft.Resources/deployments API changes

No new `Microsoft.Resources/deployments` API contract is currently proposed.

P2 relies on existing Azure deployment and control-plane behaviors. Any requirement for a new or changed deployments API contract discovered during implementation must be documented in an REP update before implementation is considered stable.

### Examples

Representative CLI usage for discovery and integration:

```sh
bicep test --recursive
bicep test --filter dev
bicep test ./main.biceptest --format sarif
bicep test ./main.biceptest --format junit
```

Representative mode declaration:

```bicep
test prod './main.bicep' = {
  params: './prod.bicepparam'
  mode: 'snapshot'
}
```

## Tradeoffs

1. Bicep-native testing increases compiler, language-server, and CLI complexity; mitigation is phased rollout with offline foundations before online expansion.
2. A declarative language may be limited for rich post-deployment probes (for example authenticated HTTP and protocol-specific validation); mitigation is explicit support for external framework composition through `bicep test` contracts.
3. Mock-driven offline tests can diverge from Azure runtime behavior; mitigation is explicit mode labeling and P2 deployment-backed coverage for live validation.
4. Online deployment tests introduce cost, safety, cleanup, and interruption risks; mitigation is explicit approval/safety gates, ownership tracking, and cleanup design requirements.
5. Stable reporting and integration contracts require long-term compatibility commitments; mitigation is SARIF first, JUnit second, then versioned structured contracts.

## Alternatives

### Another general-purpose language as the primary authoring language

Tests could be authored primarily in Go, Python, TypeScript, C#, or PowerShell, with Bicep invoked through API, RPC, or CLI wrappers.

Why not selected as primary model:

- Introduces a second required language and toolchain for module authors.
- Fragments discovery, authoring, and diagnostics across runners.
- Delays closure of the native Bicep test gap.

### Several equally first-class authoring languages and libraries from the start

Multiple official libraries and idiomatic frameworks could be delivered in parallel.

Why not selected for initial design:

- Increases surface area and long-term maintenance cost before core test semantics stabilize.
- Makes compatibility and behavioral consistency harder across language ecosystems.
- Slows delivery of the built-in experience needed by most module authors.

### No native Bicep framework

Teams could continue with existing external tools and custom scripts around `build`, `lint`, `what-if`, and `deploy`.

Why not selected:

- Leaves no consistent native model for expected failures, offline assertions, and predictable test reporting.
- Preserves duplicated and ad hoc infrastructure test patterns across repositories.

### Chosen direction

Bicep-first is selected to provide one consistent authoring and execution experience and to close the native test gap. Optional external framework integrations and language-specific libraries are deferred to P3 and may be deprioritized if community demand does not justify maintenance cost.

## Rollout plan

### Priority definitions

| Priority | Meaning |
| -------- | -------- |
| P0       | Build first. Early users, module authors, and CI need these features. |
| P1       | Add next for a strong preview or GA. These features build on P0. |
| P2       | Add after the core design is stable. These features improve online tests and tools. |
| P3       | Consider later based on user demand, adoption, and maintenance cost. |

### Priority summary

| Priority | Execution Scope | Feature List |
| -------- | --------------- | ------------ |
| P0 | Offline | Reuse `.bicepparam` files; tests that should fail; SARIF output; clear failure messages; basic deployment context mocks; docs and examples; experimental feature cleanup |
| P1 | Offline, plus design for later online tests | `.biceptest` file model; JUnit output; test discovery and filtering; snapshot and deploy modes |
| P2 | Online tests, shared tools, and better offline mocks | Deployment and post-deployment tests; test orchestration; VS Code Test Explorer support; diagnostics and code actions for test files; richer mocks |
| P3 | Online and offline | External test runner support; language-specific test libraries |

### P0 features

P0 runs without Azure and does not create resources. It provides usable repository workflows with `.bicepparam` reuse, expected failures, SARIF, clear errors, basic context mocks, and onboarding documentation.

#### Reuse `.bicepparam` Files

Execution scope: **Offline in P0**. Reused by online tests in P2.

Work:

- Allow `params` in test declarations to point to `.bicepparam` files.

Why P0:

- Enables testing of the same parameter files used in deployment pipelines.

#### Tests That Should Fail

Execution scope: **Offline in P0**. Same tests can run online in P2.

Work:

- Add expected-failure support for invalid inputs, `fail()` expressions, custom validation, and diagnostic matching.

Why P0:

- Module authors need confidence that invalid input paths fail for intended reasons.

#### SARIF Output

Execution scope: **Offline in P0**, and both modes from P2.

Work:

- Support SARIF 2.1.0 output from `bicep test`.

Why P0:

- Provides source-linked diagnostic reporting for CI and code-scanning systems.

#### Clear Failure Messages

Execution scope: **Offline in P0**. Same message model applies online in P2.

Work:

- Include test name, assertion name, source location, diagnostic details, and failed Boolean values where possible.
- Clearly separate skipped tests from evaluation errors.

Why P0:

- Failures must be actionable to support adoption.

#### Basic Deployment Context Mocks

Execution scope: **Offline in P0**.

Work:

- Support fixed resource group, subscription, tenant, deployment name, and location values in tests.

Why P0:

- Stabilizes offline results across local and CI runs.

#### Docs and Examples

Execution scope: **Both modes**, with P0 docs focused on current offline behavior.

Work:

- Document `test` and `assert` behavior, inline params, `.bicepparam` reuse, expected failures, and CI outputs.
- Clarify boundaries between `build`, `lint`, `test`, `snapshot`, `what-if`, `deploy`, and policy workflows.

Why P0:

- Experimental features require clear onboarding and expectation setting.

#### Experimental Feature Cleanup

Execution scope: **Both modes**.

Work:

- Decide whether `testFramework` enables assertions or both `testFramework` and `assertions` flags remain required.
- Update config schema and docs accordingly.

Why P0:

- Feature gating is part of first-use experience and compatibility signaling.

### P1 features

P1 strengthens offline capabilities and defines contracts needed before online expansion.

#### `.biceptest` File Model

Execution scope: **Both modes, starting offline**.

Work:

- Decide `.biceptest`, `.bicep`, or both.
- Define migration path, parser and language-server handling, CLI behavior, and file/folder conventions.

Why P1:

- Discovery and editor tooling depend on a stable file model.

#### JUnit Output

Execution scope: **Offline in P1**, both modes from P2.

Work:

- Map Bicep tests to JUnit test cases with status, duration, source mapping, and aggregate counts.

Why P1:

- Complements SARIF with mainstream CI test-result ingestion.

#### Test Discovery and Filtering

Execution scope: **Offline in P1**, both modes from P2.

Work:

- Add recursive discovery.
- Add include/exclude patterns and test-name filtering.

Why P1:

- Repository-scale CI requires discoverability and selective execution.

#### Snapshot and Deploy Modes

Execution scope: **Offline in P1**, with deploy contract designed but not executed until P2.

Work:

- Define behavior contracts for `snapshot` and `deploy` modes.

Why P1:

- Mode semantics must stabilize before deployment-backed execution.

### P2 features

P2 introduces Azure deployment-backed testing and post-deployment checks, plus shared tool improvements and richer offline mocks.

#### Deployment and Post-Deployment Tests

Execution scope: **Online**.

Work:

- Add deploy mode execution against Azure.
- Add safeguards for destructive and billable actions.

Why P2:

- Online testing introduces higher cost and operational risk and should follow offline stabilization.

#### Test Orchestration

Execution scope: **Both modes from P2**.

Work:

- Define multi-step tests and support staged execution, including deployment steps, waits, and retries.

Why P2:

- End-to-end and eventual consistency scenarios need ordered execution and retry controls.

#### VS Code Test Explorer Support

Execution scope: **Both modes**.

Work:

- Surface tests in Test Explorer.
- Add per-test run actions and inline results.
- Reuse stable CLI or RPC result contracts.

Why P2:

- Editor experiences depend on stable discovery and reporting contracts.

#### Diagnostics and Code Actions for Test Files

Execution scope: **Both modes**.

Work:

- Report unsupported properties, invalid parameter references, invalid modes, and invalid mocks.
- Add code actions and completions for `params`, `expect`, `mode`, `mocks`, and `steps`.

Why P2:

- Authoring assistance should build on settled test semantics.

#### Richer Mocks

Execution scope: **Offline**.

Work:

- Evaluate targeted mocks for selected functions, existing resources, and deployment outputs.

Why P2:

- Extend offline coverage only after basic context mocks are stable.

### P3 features

P3 adds optional ecosystem integration around the built-in Bicep model.

#### External Test Runner Support

Execution scope: **Both modes**.

Work:

- Define external-runner usage guidance for stable SARIF, JUnit, and structured result contracts.
- Provide examples for TypeScript, Python, Go, Bash, PowerShell, and C# orchestration.

Why P3:

- Depends on stable output and contract guarantees delivered in earlier phases.

#### Language-Specific Test Libraries

Execution scope: **Both modes**.

Work:

- Optionally provide language helpers for discovery, process control, cleanup, typed results, and assertion helpers.
- Host frameworks continue to manage `describe`, `it`, fixtures, watch mode, parallelism, and report plugins.

Why P3:

- Official libraries create ongoing maintenance cost and should follow demonstrated demand.

## Unresolved questions

### Questions that require REP-level resolution before stabilization

1. Which values can be determined reliably offline, and how should unresolved values appear in assertions and snapshots?
2. What is the exact `.biceptest` file model, including coexistence and migration rules with `.bicep` test authoring?
3. What is the test-local assertion and step data model, including how step outputs flow into later steps?
4. How should parameter precedence work across inline values, `.bicepparam`, CLI inputs, environment values, and secrets redaction?
5. How should online target identity and scope be selected and validated per test (tenant, subscription, resource group, credentials, cloud environment)?
6. How should deployment-backed tests define ownership, naming, isolation, cleanup, and interruption recovery?
7. What stable result and progress contract is required for CLI, VS Code Test Explorer, and external frameworks?
8. Which boundaries can be mocked or overridden without masking too much real module behavior?
9. What parallelism model and safety limits are required for Azure throttling, cost control, naming collisions, and shared state?
10. Which post-deployment scenarios should be built in, and which should rely on external frameworks such as Jest, pytest, Pester, or xUnit?

### Questions that can be finalized during implementation if REP contracts stay unchanged

1. Exact default folder and duplicate-name handling rules for discovery.
2. Final CLI switch naming for filtering dimensions beyond test name and file patterns.
3. Specific timeout/backoff defaults for retry and wait orchestration.
4. Exact diagnostic taxonomy wording and report field names for skip/error subcategories.

## Out of scope

- Redesigning static validation behavior in `bicep build` and `bicep lint`.
- Turning Azure deployment validation or what-if into built-in test modes in this REP.
- Replacing Azure Policy or dedicated security/compliance scanners.
- Building performance, load, resilience, chaos, disaster recovery, or drift tooling.
- Building a managed remote Bicep test execution service.
- Defining full protocol-specific post-deployment probe libraries directly in Bicep.

External tools and frameworks may compose with `bicep test`, but they are not part of the core built-in framework defined by this REP.
