---
REP Number: 0016
Author: anthony-c-martin (Anthony Martin)
Start Date: 2025-02-28
Feature Status: Public
---

# Support resource references in lambdas

## Summary

Proposes a mechanim such that Bicep is able to allow resource referencing inside lambda syntax blocks.

## Motivation

Lambdas are currently limited in their capabilities, because of limitations in the Deployment Engine with calculating references. See #14106 for an example of some of the difficulties this creates.

The introduction of symbolic name referencing gives us an opportunity to improve on this.

## Detailed design

### Current State
When writing code like the following, Bicep will emit a `reference()` expression for the output value:
```bicep
// myRes is a resource declared in this file
output fooValue string = myRes.properties.foo
```

The JSON expression generated will look something like the following (assuming symbolic names are enabled):
```
"value": "[reference('myRes', 'Full').properties.foo]"
```

Currently, when the Deployment Engine processes a template, the first thing it attempts to do is to build a DAG for the resources in the deployment. It does this by visiting each reference statement in each resource body, and building a dependency for it.

The reference function is currently interpreted as either:
1. Refering to a resource name or id of a resource that is declared in the template.
1. Refering to a resource name or id of a resource that is NOT declared in the template.
1. Refering to a resource symbolic name of a resource that is declared in the template.

It is crucial that this visiting process happens for two purposes:
1. To insert dependencies that are not explicitly declared (e.g. if the user forgets a `dependsOn`, but includes a `reference` to a resource that's in the deployment).
1. To identify resources that are not explicitly declared in the template, but should be treated as if they were "existing" resources. This results in adding a new node to the DAG which performs a resource GET.

### Proposed State

1. Introduce a new function `resources()`, which takes a single string argument that ONLY permits a symbolic reference. Conceptually this is similar to the `variables` or `parameters` functions.
1. Skip the visiting of `resources()` functions when building the DAG. This should no longer be necessary because:
    * We can rely on Bicep code generation to satisfy the 1st need for the visiting process. It already handles the correct generation of `dependsOn` statements, and so the extra validation offered by performing this check is not necessary.
    * We do not need to satisfy the the 2nd need for the visiting process listed under "Current State" - a symbolic name must (by definition) refer to a resource that is declared in the template.

## Changes needed
1. Add support for the `resources()` function to the Deployment Engine.
1. Update Bicep to remove the block on references inside lambdas, to switch to symbolic name generation, and to emit the `resources()` function instead of `reference()`.

## Open Questions
- Should this be a new function, or just a different set of parameters for `reference`?
- How do the referencing capabilities interact with https://github.com/Azure/bicep-reps/pull/15?
