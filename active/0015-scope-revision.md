---
REP Number: 0015
Author: anthony-c-martin (Anthony Martin)
Start Date: 2025-01-23
Feature Status: Public
---

# Formalize handling of "scope" 

## Summary

Introduce a more well-defined concept of scope into the ARM Template language to simplify Bicep code generation, and allow us to tackle complex edge cases.

## Motivation

Bicep improved upon ARM Templates by offering a stronger concept of "scope", by validating many of the restrictions imposed by the deployment engine, and generating correct syntax in places where it was previously difficult. In the early days of Bicep, it was very difficult to evolve the underlying JSON language, and so much of this functionality had to be implemented in the compiler. This works well enough for simple cases, but breaks down when we have to do more complex analysis and transformation of values at compile time (see https://github.com/Azure/bicep/issues/1876 for example of an issue which is complex to fix). In addition, the complexity and number of edge cases becomes exponentially worse when layered with other similar features, and so this reduces our velocity for adding new features (for example, loops are in a very similar position, where bug fixes and feature work are extremely complex).

Now that we have a much more well-defined mechanism for evolving the Template language using language versions, I am proposing we revisit this, and introduce a more standardized "scope" capability, to simplify the code generation that Bicep has to do.

This also may simplify some of the work needed to implement https://github.com/Azure/bicep/issues/2246, because the handling of scope added complexity to this feature.

A non-exhaustive list of other issues this MAY allow us to address or make progress on:
* https://github.com/Azure/bicep/issues/2246
* https://github.com/Azure/bicep/issues/1876
* https://github.com/Azure/bicep/issues/3542
* https://github.com/Azure/bicep/issues/947
* https://github.com/Azure/bicep/issues/788
* https://github.com/Azure/bicep/issues/3734
* https://github.com/Azure/bicep/issues/4698
* https://github.com/Azure/bicep/issues/8783
* https://github.com/Azure/bicep/issues/7637
* https://github.com/Azure/bicep/issues/6182
* https://github.com/Azure/bicep/issues/10475
* https://github.com/Azure/bicep/issues/10131
* https://github.com/Azure/bicep/issues/10185
* https://github.com/Azure/bicep/issues/12533
* https://github.com/Azure/bicep/issues/10885

## Detailed design

The design centers on avoiding "magic" in the compiler, and instead depend on [duck typing](https://en.wikipedia.org/wiki/Duck_typing).

We define a "scope" as having the following type signature, represented in Bicep:
```bicep
@discriminator('type')
type ResourceScope = {
  type: 'tenant'
} | {
  type: 'managementGroup'
  managementGroupId: string
} | {
  type: 'subscription'
  subscriptionId: string
} | {
  type: 'resourceGroup'
  subscriptionId: string
  resourceGroup: string
} | {
  type: 'extension'
  resourceId: string
}
```

### Changes to Template JSON
In Template JSON, we introduce a new property "@scope", which takes an object of this type. This will require changes to the deployment service to understand, but I do not propose changing any of the validation we have with this spec.

For example, a resource in JSON:
```jsonc
{
  "type": "Microsoft.App/managedEnvironments",
  "apiVersion": "2022-03-01",
  "@scope": {
    "type": "resourceGroup",
    "subscriptionId": "..."
    "resourceGroup": "..."
  }
  // other properties omitted
}
```

### Changes to functions
Changes will need to be made to the following functions to output values that are compatible with `@scope`:
* `resourceGroup()`
* `subscription()`
* `tenant()`
* `managementGroup()`
* `deployment()`

TODO detailed explanation of what needs to change

### Formalized definition of how to represent a symbolic reference
We don't have a defined way of representing a "reference to a resource" in JSON today, so we will most likely need to formalize this to support extension resources. Consider:
```bicep
resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: storageAccount
  name: 'ra'
}
```

In the generated JSON, this would need to include an expression that resolves to the following:
```jsonc
{
  "type": "Microsoft.App/roleAssignments",
  "apiVersion": "2022-04-01",
  "@scope": {
    "type": "extension",
    "resourceId": "[resourceInfo('storageAccount').id]"
  }
  // other properties omitted
}
```

TODO more detail

### Why does this solve the problem?

Depending on duck-typing means that we can just emit standard expressions to handle some more complex code-generation tasks.

The following example is extremely difficult to solve with today's code generation that depends on "magic compiler stuff":
```bicep
module mod 'mod.bicep' = if (!empty(otherResourceGroup)) {
  name: '${_deployment}-mod'
  scope: !empty(otherResourceGroup) ? resourceGroup(otherResourceGroup) : resourceGroup()
}
```

With this proposal, if we instead have clearly defined types for the `scope` property, and outputs of the `resourceGroup()` function, it is extremely easy to validate and generate code for, without having to build any complex compile-time logic.

## Examples

TODO

## Unresolved questions

- Should this spec also cover a more formalized definition of "parent" (e.g. doing something similar for `@parent`)?
- Should this be gated behind a new template language version?