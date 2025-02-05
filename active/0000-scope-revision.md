---
REP Number: <Fill me in with a four-digit number matching the pull request number; Update AFTER PR is approved and BEFORE is merged.>
Author: anthony-c-martin (Anthony Martin)
Start Date: 2025-01-23
Feature Status: Public
---

# Standardize on handling of scope

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

A scope should have the following type signature:
```bicep
type scope = {
  
}
```

TODO

## Examples

TODO

## Unresolved questions

TODO
