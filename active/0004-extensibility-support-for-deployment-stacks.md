---
REP Number: <Fill me in with a four-digit number matching the pull request number; Update AFTER PR is approved and BEFORE is merged.>
Author: kalbert312 (Kyle Albert)
Start Date: 2024-08-27
Feature Status: <Private Preview | Public Preview | Public>
Enhances: 0003
---

<!-- Remove this comment and the prompts (in the form of blockquotes) for each section before submitting your PR -->

# [Breaking Change (delete if not applicable)] Title - replace me with feature name

## Summary

This document describes changes to how extensions in Bicep and ARM templates are defined in order to support managing
extension resources with Deployment stacks.

## Terms and definitions

> Include any terms, definitions, or acronyms that are used in this design document to assist the reader.

- `Control plane`: An Azure or non-Azure API for managing the lifecycle of resources. This is the API for provisioning
  or deleting resources.
- `Extension resource`: An Azure data-plane resource or non-Azure resource declared in a Bicep file or an ARM template.
- `Extension configuration`: Data needed to connect to a non-Azure control plane to manage resources. This can contain 
secrets, such as a Kube config for AKS clusters.
- `Deployment stack`/`stack`: An atomic unit for lifecycle management of a set of resources deployed by a Bicep or ARM template 
deployment. Resources can be Azure resources or extension resources. Stacks can unmanage resources which results in
either their deletion or detachment from the stack.

## Motivation

> Outline the reasons behind implementing this feature, the use cases it supports, and the expected outcomes.

Currently, Deployment stacks do not support managing extension resources deployed through one. Users that utilize
Deployment stacks will want to manage the lifecycle of any resource deployed through a stack regardless of its control
plane as this is a key feature of stacks. 

In order to minimally support deletion of extension resources, changes need to made at both the Bicep and ARM template
levels to how extension configurations are defined so the Deployment stack service can retrieve necessary secret data in
a secure and direct way.

The current implementation of extension configurations in Bicep and ARM templates is too flexible. For example, it 
allows users to consume template parameters and use template language expressions to manipulate data. This creates a
problem for the stacks service because it must be able to reevaluate the extension configuration at a later time when 
resource deletion occurs. Non-stack deployments do not have this problem because the configuration is only needed at
the time of deployment. The extension configuration can change over time due to credential changes, rotations, or other
reasons that are opaque to the Stacks service. Therefore, it is necessary to constrain extension configurations to
simple directives the stacks service can act upon.

## Detailed design

> The core of the REP. Elaborate on the design and implementation with sufficient detail for someone familiar with Bicep
to grasp. Provide detailed examples to illustrate how the feature is used and its implications on user experience.

### Client side changes

### Server side changes (if applicable)

### Microsoft.Resources/deployments API changes (if applicable)

### Examples

- Example 1

```bicep
```

- Example 2

```bicep
```

## Drawbacks

> Discuss any potential drawbacks associated with this approach. Why might this *not* be a suitable choice? Consider
implementation costs, integration with existing and planned features, and the migration cost for Bicep users. Clearly
identify if it constitutes a breaking change.

## Alternatives

> Explore alternative designs that were considered and clarify why the chosen design stands out among possible options.
Provide rationale and discuss the impact of not selecting other designs.

## Rollout plan

> Describe how you will deliver your feature. Does it need to be an experimental feature? If the feature involves
backend changes, dose it need to be guarded by an ARM feature flag?

## Unresolved questions

> - What aspects of the design do you expect to resolve through the REP process before merging?
> - What parts of the design do you anticipate resolving through feature implementation before stabilization?

## Out of scope

> - What related issues are considered out of scope for this REP but could be addressed independently in the future?

**NB**: This section is not intended to capture unfinished elements of a design. For draft proposals, anything that is
yet to be decided but must be addressed in order for a feature to be functional or GA should be explicitly called out in
the **Design** section (or the **Unresolved questions** section). This section is instead intended to identify related
issues or features on which you do not wish to make definitive decisions/foreclose any possibilities. Carefully think
through how your design will avoid impacting these out of scope questions -- if your design does impact anything in this
section, that is a strong indication that whatever is impacted was not out of scope at all. 


