---
REP Number: <Fill me in with a four-digit number matching the pull request number; Update AFTER PR is approved and BEFORE is merged.>
Author: anthony-c-martin (Anthony Martin)
Start Date: 2025-01-23
Feature Status: Public
---

# Standardize on decorator code generation

## Summary

Introduce a new standardized pattern for how a decorator in the Bicep language is converted into JSON, which we can use for future decorators and possibly also consider adding for existing ones.

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

1. Introduce a top-level resource property `@decorators` to the JSON (also establishing a convention of using the `@` character to distinguish "meta" properties from actual properties that should be sent on an API call)
    ```json
    {
      "@decorators": {}
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
      "@decorators": {
        "name": [<arg1>, <arg2>]
      }
    }
    ```

## Examples

Bicep example:
```bicep
@description('hello!')
@patch() // not currently supported, but might be one day
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
  "@decorators": {
    "description": ["hello!"],
    "patch": []
  },
  "properties": {
    "patchMe": "please"
  }
}
```


## Unresolved questions

- Should the `@decorators` property be an object or an array? Object (as currently specced) is simpler, but does not allow for duplicates, and does not preserve ordering.
- Should we attempt to backfill existing decorators with this new contract (e.g. `@description`)?
- What about other meta-properties, like `scope`, `parent` and `dependsOn`? Should these become decorators in the long term?
- What about decorators on other declarations (parameters, variables, outputs)?
- We already have the `metadata` field defined - should we try and reuse this rather than creating a new `@decorators` field?
- What would we do with decorators that support expressions?