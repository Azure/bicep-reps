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
param kubeConfigParam string // prone to error: what if it's not secured.

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
is transparent to the user.

### The new design

The following Bicep code illustrates the new design for extension configurations. The root deployment supplies the
extension configuration details to the child deployment, allowing direct access to ARM resources that supply secrets in 
the root deployment.

extResources.bicep - Child deployment with extension resources

```bicep
extension kubernetes as k8s // optional alias

extension microsoftGraph // No config is needed for this one. This extension uses OAuth OBO.

// ... extension resources
```

main.bicep - The root deployment
```bicep
// The ARM aksCluster resource can supply kubeConfigs for the AKS Kubernetes use case.
resource aksCluster '...' = {
  // ...
}

module extResources 'extResources.bicep' = {
  name: 'k8sResourcesDeploymentName'
  extensions: {
    // The keys are extension aliases (or extension name if no alias is set)
    // The values are extension configs
    // Optionality of extension config is decided by the extension
    // Expressions in this scope are limited to what we can compile to a simple directive the stacks service can execute (ARM Resource API call, JSON path). 
    // No deployment parameters or other dynamic expressions that cannot be compiled to simple directives.
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

The keys of the "extensions" object are the extension aliases of the module deployment. If an alias is not provided, it
will be the extension name. The values of the object are extension configurations. Optionality of configuration is
determined by the extension. Expressions in this object can be constrained to simple expressions that can be compiled
to simple directives that both the deployments engine and the stacks service can execute.

Allowed expressions:
- ARM resource access that can be compiled to a resource ID (`aksCluster` in the example).
- ARM resource API calls that can be compiled to a URI, HTTP method and any inputs (`listClusterAdminCredential` in the example).
- JSON value accessors (`kubeconfig[0].value` in the example).
- Usage of compile-time constants (the `0` in `kubeconfig[0]`, `namespace: 'default'` in the example).

Disallowed expressions:
- TBD.
- Usage of deployment parameters.
- Usage of Bicep or ARM template functions (built-in or user defined).

### Server side changes

### Microsoft.Resources/deployments API changes (if applicable)

### Examples

## Drawbacks

> Discuss any potential drawbacks associated with this approach. Why might this *not* be a suitable choice? Consider
implementation costs, integration with existing and planned features, and the migration cost for Bicep users. Clearly
identify if it constitutes a breaking change.

The move of extension configuration definitions to the parent deployment properties along with the language expression 
constraints it imposes means that extension resources that require a configuration will no longer be deployable
as a root deployment. There is no mechanism at the root level to provide this data. Client side changes would need to
be implemented to accept extension configurations that conform to the expression rules. Another complication is how the 
user experience of templates would be affected. Because there's expectation that a parent deployment will supply extension
configurations, it wouldn't be possible to provide a design-time error diagnostic in the extension resource deployment
file until it is used as a root deployment.

## Alternatives

> Explore alternative designs that were considered and clarify why the chosen design stands out among possible options.
Provide rationale and discuss the impact of not selecting other designs.

## Rollout plan

> Describe how you will deliver your feature. Does it need to be an experimental feature? If the feature involves
backend changes, dose it need to be guarded by an ARM feature flag?

## Unresolved questions

> - What aspects of the design do you expect to resolve through the REP process before merging?
> - What parts of the design do you anticipate resolving through feature implementation before stabilization?

### Key vault secret expressions in extension configurations

Whether key vault secret access should be supported in extension configurations is a tricky subject. Users will likely
want this feature, and it would work fine with a standalone deployment that isn't tethered to a stack because the secret
is only needed at the time of deployment. The problem from the stacks perspective with introducing this feature is 
that accessing a user's key vault in the future for the deletion of stack resources introduces a failure point. What if
the user did not properly maintain this secret and it's now invalid? This type of issue would block the deletion or 
update of a stack when resources that no longer managed are to be deleted and potentially result in unexpected stack
states. The errors would likely manifest as incidents for the stacks team to investigate.

## Out of scope

> - What related issues are considered out of scope for this REP but could be addressed independently in the future?

- Client support for root level deployments with extensions that require an extension configuration.
- Deployment stack resource locks on extension resources (in ARM, this is deny assignments that prevent deletion or 
update of resources while being managed by the stack).

**NB**: This section is not intended to capture unfinished elements of a design. For draft proposals, anything that is
yet to be decided but must be addressed in order for a feature to be functional or GA should be explicitly called out in
the **Design** section (or the **Unresolved questions** section). This section is instead intended to identify related
issues or features on which you do not wish to make definitive decisions/foreclose any possibilities. Carefully think
through how your design will avoid impacting these out of scope questions -- if your design does impact anything in this
section, that is a strong indication that whatever is impacted was not out of scope at all. 


