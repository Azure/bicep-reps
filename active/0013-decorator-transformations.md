---
REP Number: '0013'
Author: anthony-c-martin (Anthony Martin)
Start Date: 2025-01-23
Feature Status: Public
---

# Standardize on decorator code generation

## Summary

Introduce a new standardized pattern for how a decorator in the Bicep language can converted into JSON, which we can use for future decorators and possibly also consider adding for existing ones.

> [!NOTE]
> This spec focuses primarily on decorators which must be understood by the Deployment Engine (deploy-time directives), whereas many of the existing decorators we support are compile-time directives. The current motivation for this spec is for supporting deploy-time directives on resource & module declarations. A similar approach may be followed if deploy-time directives are needed on other forms of syntax in future, but it is not a requirement for compile-time directives.

## Motivation

Bicep offers the ability to augment resource (and other) declarations with [decorators](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/file#decorators), which are generally used to modify behavior of the declaration, without directly affecting the payload.

For example, the following definition instructs the deployment engine to deploy the resources sequentially instead of concurrently:
```bicep
@batchSize(1)
resource foo 'Microsoft.DataFactory/factories/pipelines@2018-06-01' = [for i in [0, 1]: {
  name: 'foo${i}'
}]
```

Currently, each decorator needs specific transformation logic to be added to Bicep, in order to pass information about what the user has configured to the deployments backend. For example, for batchSize, this impacts the value of the "copy.batchSize" property.

We've gone with this approach mostly for back-compat reasons, but it adds complexity to Bicep code generation, and code generation complexities tend to build on top of each other exponentially.

It also pollutes the resource property namespace, which hasn't been a problem thus far, but could create challenges with extensible resources.

## Detailed design

1. Introduce a top-level resource property `@options` to the JSON (also establishing a convention of using the `@` character to distinguish "meta" properties from actual properties that should be sent on an API call)
    ```json
    {
      "@options": {}
    }
    ```

1. Define a standard transformation that new decorators follow from Bicep -> JSON.

    The following Bicep:
    ```
    @<name>(<arg1>, <arg2>, ...)
    ```
    
    Would be sent on the wire as:
    ```
    {
      "@options": {
        "name": [<arg1>, <arg2>]
      }
    }
    ```

The benefit of this standardized transformation is that conversion between Bicep and JSON (both ways) can be purely mechanical - meaning it doesn't need any understanding of what each decorator is doing.

## Examples

Bicep example:
```bicep
@deployIfNotExists() // check exists proposal
@waitUntil(x => x.ProvisionStatus == 'Succeeded', 'PT20S') // wait and retry proposal
resource foo 'Microsoft.DataFactory/factories@2018-06-01' = {
  name: 'foo'
  properties: {
    patchMe: 'please'
  }
}
```

JSON conversion:
```json
{
  "type": "Microsoft.DataFactory/factories",
  "apiVersion": "2018-06-01",
  "name": "foo",
  "@options": {
    "deployIfNotExists": [],
    "waitUntil": [
      "[lambda('x', equals(lambdaVariables('x').ProvisionStatus, 'Succeeded'))]",
      "PT20S"
    ]
  },
  "properties": {
    "patchMe": "please"
  }
}
```