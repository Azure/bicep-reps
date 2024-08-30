---
REP Number: <Fill me in with a four-digit number matching the pull request number; Update AFTER PR is approved and BEFORE is merged.>
Author: kalbert312 (Kyle Albert)
Start Date: 2024-08-27
Feature Status: Public Preview
Enhances: 0003
---

<!-- Remove this comment and the prompts (in the form of blockquotes) for each section before submitting your PR -->

# Integrating extensible resources with deployment stacks

## Summary

This document describes changes to how extensions in Bicep and ARM templates are defined in order to support managing
extension resources with Deployment stacks.

## Terms and definitions

- `Control plane`: An Azure or non-Azure API for managing the lifecycle of resources. This is the API for provisioning
  or deleting resources.
- `Extension resource`: An Azure data-plane resource or non-Azure resource declared in a Bicep file or an ARM template.
- `Extension configuration`: Data needed to connect to a non-Azure control plane to manage resources. This can contain 
secrets, such as a Kube config for AKS clusters.
- `Deployment stack`/`stack`: An atomic unit for lifecycle management of a set of resources deployed by a Bicep or ARM template 
deployment. Resources can be Azure resources or extension resources. Stacks can unmanage resources which results in
either their deletion or detachment from the stack.

## Motivation

Currently, Deployment stacks do not support managing extension resources deployed through one. Users that utilize
Deployment stacks will want to manage the lifecycle of any resource deployed through a stack regardless of its control
plane as this is a key feature of stacks. 

In order to minimally support deletion of extension resources which is the goal of this integration, changes need to 
made at both the Bicep and ARM template levels to how extension configurations are defined so the Deployment stack 
service can retrieve necessary secret data in a secure and direct way.

The current implementation of extension configurations in Bicep and ARM templates is too flexible. For example, it 
allows users to consume template parameters and use template language expressions to manipulate data. This might sound
great from a user experience perspective, but this creates a problem for the stacks service because it must be able to
reevaluate the extension configuration at a later time when resource deletion occurs. Non-stack deployments do not have 
this problem because the configuration is only needed at the time of deployment. The extension configuration can change
over time due to credential rotations or other reasons that are opaque to the Stacks service. Therefore, it is necessary
to constrain extension configurations to use constants or simple directives the stacks service can act upon.

## Detailed design

> The core of the REP. Elaborate on the design and implementation with sufficient detail for someone familiar with Bicep
to grasp. Provide detailed examples to illustrate how the feature is used and its implications on user experience.

The key design change is to move where extension configurations are defined in Bicep and ARM templates. Currently,
the extension configurations are defined within the deployment of the extension resources being deployed. This REP's 
proposal is to move these configuration definitions to a dedicated deployment input property that is separate from 
deployment parameters. See the examples section for Bicep and ARM template examples.

### Analyzing the current implementation

Why this change is necessary becomes more clear by observing what the stacks services would be required to do with the
current implementation. Consider this Bicep deployment:

```bicep
@secure()
param kubeConfigParam string

extension kubernetes with { // this will produce a diagnostic if the config is not provided
  kubeConfig: kubeConfigParam
  namespace: 'default'
}

// ... extension resources
```

In this example, the extension configuration contains a secret from a deployment parameter. A deployment parameter can
be supplied by the user or sourced from a parent deployment. In either case, it is a user secret and thus cannot be 
persisted long term by stacks for use in future resource deletion. 

In the case where it's supplied by the user and without secret persistence, the only way the stacks service could 
possibly source this data again is to prompt the user for it. It is unclear how that experience would work. Does the 
user need to supply the entire old deployment's parameters back to the stack service? How do they know which version of 
inputs to supply? What if a subset of those parameters are no longer relevant? Would they only need to supply parameters
used in configurations? How would we detect and differentiate that across an entire tree of deployments?

In the case where it's supplied by a parent deployment, the problem becomes recursive. It's either provided by the user,
is sourced from somewhere else in that parent deployment (such as another resource's outputs like a key vault or AKS 
cluster), or sourced from another parent deployment.

To complicate this further, any pathway between the secret's source and consumption can have dynamic language expressions
such as template function calls or branching. This would effectively require stacks to generate a separate deployment template
at deletion time from the user's deployment template to process these language expressions.

The stacks service cannot store user secrets, and it should not evaluate dynamic deployment template language expressions.
The stacks service needs extension configurations to be defined in a format that is interpretable at deletion time that
is transparent to the user and secure.

### The new design from the Bicep perspective

The following Bicep code illustrates the new design for extension configurations. The root deployment supplies the
extension configuration details to the child deployment, allowing direct access to ARM resources that supply secrets in 
the root deployment.

main.bicep - The root deployment
```bicep
// The ARM aksCluster resource can supply kubeConfigs for the AKS Kubernetes use case.
resource aksCluster '...' = {
  // ...
}

module extResources 'extResources.bicep' = {
  name: 'extResourcesDeploymentName'
  extensionConfigs: {
    k8s: {
      kubeConfig: aksCluster.listClusterAdminCredential().kubeconfig[0].value
      namespace: 'default'
    }
  }
  params: {
    // ...
  }
}
```

extResources.bicep - Child deployment with extension resources
```bicep
extension kubernetes as k8s // optional alias

extension microsoftGraph // No config is needed for this one. This extension uses OAuth OBO.

// ... extension resources
```

The keys of the "extensionConfigs" object are the extension aliases of the module deployment. If an alias is not provided,
it will be the extension name. The values of the object are extension configurations. Optionality of configuration is
determined by the extension. If this is compared to a typical resource group deployment, it is aligned because the
resource group being deployed to is not defined within the deployment itself, but as a parameter to the deployment,
either through the CLI parameters or as a scope for nested deployments. In this example, the deployment scope(s) are 
being defined as extension configurations. Expressions in this object can be constrained to expressions that can be 
compiled to simple directives that both the deployments engine and the stacks service can execute.

Allowed expressions:
- ARM resource access that can be compiled to a resource ID (`aksCluster` in the example).
- ARM resource API calls that can be compiled to a URI, HTTP method, and any inputs (`listClusterAdminCredential` in the example).
- JSON value accessors (`kubeconfig[0].value` in the example).
- Usage of compile-time constants (the `0` in `kubeconfig[0]`, `namespace: 'default'` in the example).

Disallowed expressions:
- Usage of deployment parameters.
- Usage of Bicep or ARM template functions (built-in or user defined).
- Usage of branched expressions (ternaries).

### The new design from the ARM template perspective

The main Bicep deployment would compile to the following ARM template following this paragraph. Only REP relevant data
is shown. The new "extensionConfigs" deployment property will be similar in style to how deployment parameters and type 
definitions are defined in deployment templates. Configuration object property values can be literal values such as strings,
booleans, and objects. Values can also be directives, such as to fetch a value from an ARM resource API. A directive
consists of one or more evaluation steps. The Bicep compiler will be responsible for converting template language 
expressions to these directives that both the deployment service can and stacks service can process without need for 
complex interpretation. The directives will not contain template language expressions to simplify any validation needed
for acceptable directives.

main.json - The root deployment
```json
{
   "resources": {
      "extResources": {
         "type": "Microsoft.Resources/deployments",
         "apiVersion": "2022-09-01",
         "name": "extResourcesDeploymentName",
         "properties": {
            "extensionConfigs": {
               "k8s": {
                  "namespace": {
                     "type": "string",
                     "value": "default"
                  },
                  "kubeConfig": {
                     "type": "directive",
                     "evaluation": [
                        {
                           "type": "ArmApiCall",
                           "resourceId": "/subscriptions/.../resourceGroups/.../Microsoft.ContainerService/managedClusters/...",
                           "method": "GET",
                           "apiVersion": "2024-02-01",
                           "apiAction": "/listClusterAdminCredential",
                           "query": "?a=1&b=2" // optional
                           "body": {} // optional
                        },
                        {
                           "type": "JSONPath",
                           "path": "kubeconfig[0].value"
                        }
                     ]
                  }
               }
            },
            "template": {
               "extensions": {
                  "k8s": {
                     "extension": "Kubernetes",
                     "version": "1.0.0"
                     // "config" is removed, it's expected to be provided by properties of the deployment
                  }
               }
            }
         }
      }
   }
}
```

### Microsoft.Resources/deployments API changes

This REP extends upon the API changes in REP 0003 by adding a configuration value to the extensions returned from
a deployment GET. This configuration object will be the same as the configuration object described in the above ARM
template design changes.

```json
{
   // ... deployment contract
   "extensions": [
      {
         "name": "Kubernetes",
         "alias": "k8s",
         "version": "1.0.0",
         "deploymentId": "/subscriptions/.../resourceGroups/.../providers/Microsoft.Resources/deployments/...",
         "config": {
            "namespace": {
               "type": "string",
               "value": "default"
            },
            "kubeConfig": {
               "type": "directive",
               "evaluation": [
                  {
                     "type": "ArmApiCall",
                     "resourceId": "/subscriptions/.../resourceGroups/.../Microsoft.ContainerService/managedClusters/...",
                     "method": "GET",
                     "apiVersion": "2024-02-01",
                     "apiAction": "/listClusterAdminCredential",
                     "query": "?a=1&b=2" // optional
                     "body": { } // optional
                  },
                  {
                     "type": "JSONPath",
                     "path": "kubeconfig[0].value" // decompose?
                  }
               ]
            }
         }
      }
   ]
}
```

With the deployments service returning a list of all extensions (across all nested deployments) with their configuration
directives and returning all extension resources deployed in the template (across all nested deployments), the stacks 
service can process each extension resource and use the "deploymentId" and "extensionAlias" properties as a composite 
key to the data in the extensions array.

## Drawbacks

The biggest drawback of this approach is that extension configuration data cannot be sourced from deployment parameters.
This design decision is driven solely to support the stacks user experience not needing to prompt for the parameter 
values of the last stack deployment as user secrets cannot be persisted long term. This has the side effect of disallowing
the use of non-secret parameters as well, such as the "namespace" in the Kubernetes extension use case. There could be
a constraint to allow only non-secure parameter usage in extension configurations, but that could end up encouraging the
user to work around that by not using secure parameters for data that should be secured. Alternatively, the secret portion
of the extension configuration could be split into from non-secret data. The secret portion would adopt the design proposed
in this REP and the non-secret portion could be kept as-is, but there would need to be a way for the extension to
provide metadata about the properties of the configuration schema.

The move of extension configuration definitions to the parent deployment properties along with the language expression 
constraints it imposes means that extension resources that require a configuration will no longer be deployable
in a root deployment. There is no mechanism at the root level to provide this data. Client side changes would need to
be implemented to accept extension configurations that conform to the expression rules. Another complication is how the 
user experience of templates would be affected. Because there's expectation that a parent deployment will supply extension
configurations, it wouldn't be possible to provide a design-time error diagnostic in the extension resource deployment
file until it is used as a root deployment. There needs to be documentation on expectations of deployment setup for 
these scenarios. One way to mitigate the problem in the UX directly is to provide a non-error diagnostic on the extension
line for extensions that require configuration so at least the user is aware at the time of authoring.

## Alternatives

### Resource deployment parameters

The reason why extension configuration is moved to the deployment input is mostly due to needing to be able to
reference ARM resources in a parent deployment as the source of truth for the configuration. When deployment parameters
as they are today are used for extension configurations, there are more layers make assertions at for the expression
constraints. In addition, it's not currently possible to define a deployment parameter as a reference to an ARM resource
in a way that can be analyzed at template evaluation time. There are draft designs for this type of feature, but it is 
not available at the start time of this document. Deployment parameters are currently limited to primitives and basic 
types that do not provide enough engine-level metadata to make assertions about the source of a parameter value (for 
example, how could it be known that a secure string parameter is always sourced from the AKS cluster admin credentials 
in the Kubernetes example?).

Resource deployment parameters, if designed in a way that enforces a certain resource type, would solve this issue, and
there wouldn't be need to move the extension configuration definition site. Having this available would also solve the
root deployment problem because the parameter would be like any other parameter, but enforced to be a string resource ID
of the defined resource type. This would also minimize the changes needed to clients.

The reason this approach is not chosen is due to dependency on this feature development which would delay stacks extension
support. It needs to be decided if it's worth implementing that feature first.

Here is an example of this approach:

```bicep
param aksCluster Resource<'Microsoft.ContainerService/managedClusters@2024-02-01'> // = "/subscriptions/.../resourceGroups/.../Microsoft.ContainerService/managedClusters/...

param namespace string = 'default'

extension kubernetes with {
  kubeConfig: aksCluster.listClusterAdminCredential().kubeconfig[0].value
  namespace: namespace
}

// ...
```

## Rollout plan

The milestone for the initial rollout will be the successful deletion of extension resources through stacks. Supporting
features for extensibility are considered experimental so this specific feature will be considered experimental.

Here are factors that need to be considered for the rollout:
- AKS/Kubernetes in Azure is only available in production environments and this is the only available V2 extension at
this time that can be tested with.
- New prod API versions require going through swagger review which can take a lot of time.
- SDK updates for the deployment service should be consumed by Stacks before releasing the Stacks Stable API Version.

The rollout:
1. Changes in both the Deployments service and Stacks service will be feature gated.
    - Feature gate needs to be at the subscription level to enable Canary-scoped development and testing.
    - The feature gate should apply on whatever the current stable (or preview) API version is for each service 
      (or all API versions if it easier).
1. Development iteration of the Deployment service and Stack service will take place and be tested in Canary until the 
milestone is hit.
1. Develop some synthetics to run in production that use the kubernetes extension in a Deployment stack.
1. Once the milestone implementation is sufficiently bug bashed and signed off by the team, the feature gate will be
disabled.
1. The Swagger update process will begin for at least a new Prod Preview API version.
    - Priority should be given to the deployments side due to it being a dependency.
    - Both swagger updates could be potentially be done in parallel.
    - It may be possible to start the Deployment service swagger early if there's confidence in its changes.
1. Incorporate any changes from swagger review and retest if needed using the feature flags.
1. Merge the Deployment service swagger changes and release SDK.
1. Update stacks code to consume new deployment SDK code.
1. Merge the Deployment stack swagger changes and release SDK.
1. Update client tools such as Azure CLI and Azure PowerShell to handle the new API responses.
1. Collaborate with the Portal team to update the deployments blade, ensuring that the UI reflects the API changes.

## Unresolved questions

### Key vault secret expressions in extension configurations

Whether key vault secret access should be supported in extension configurations is a tricky subject. Users will likely
want this feature, and it would work fine with a standalone deployment that isn't tethered to a stack because the secret
is only needed at the time of deployment. The problem from the stacks perspective with introducing this feature is 
that accessing a user's key vault in the future for the deletion of stack resources introduces a failure point. What if
the user did not properly maintain this secret and it's now invalid? This type of issue would block the deletion or 
update of a stack when resources that no longer managed are to be deleted and potentially result in unexpected stack
states. The errors would likely manifest as incidents for the stacks team to investigate.

### Breaking up extension configurations into secret and non-secret components

This question was arrived at in the drawbacks section. The extension would need to provide metadata about top-level 
configuration properties to enable the various layers to differentiate secret versus non-secret properties. It would
enable use of deployment parameters within non-secret portions of the extension configuration and stacks could persist
these value.

### Getting the resource dependency graph of a deployment

Deletion of resources may require sequencing to avoid failures in deletion. The stacks service will employ a brute-force
approach to resource deletion, which means it will keep retrying all resources until they succeed within reasonable limits.
If brute-force is not sufficient or there's common cases where ordering is necessary, the design may need modification to
fetch the deployment dependency graph from the Deployments service after stack template deployment has completed and 
persist it in the stack for later use.

## Out of scope

- Client support for root level deployments with extensions that require an extension configuration. A mechanism similar 
to parameters files would be needed for each deployment client, and it's not necessary for a minimal integration. There
are other efforts of similar nature in progress, such as `.bicepdeploy` files.
- Deployment stack resource locks on extension resources (in ARM, this is deny assignments that prevent deletion or 
update of resources while being managed by the stack).
