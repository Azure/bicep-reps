---
REP Number: <Fill me in with a four-digit number in double quotes (otherwise it's octal) matching the pull request number; Update AFTER PR is approved and BEFORE is merged.>
Author: jeskew (Jonathan Eskew)
Start Date: 2025-07-16
Feature Status: Public
Bicep Issue Number(s): 13399
---

# Any-typed parameters

## Summary

Add a new type named `any` for typed symbols that do not have a type or whose type is not a subset of one of the ARM primitive types.

## Terms and definitions

**ARM Primitive Type** - One of five types defined by the Deployments engine (`string`, `int`, `bool`, `array`, or `object`). These correspond roughly to the kinds of values defined in [the JSON grammar](https://www.crockford.com/mckeeman.html).
**Template schema node** - A JSON object that expresses type constraints in ARM. For example, `{"type": "string", "minLength": 3}`. Schema nodes may also use references, e.g., `{"$ref": "#/definitions/myType"}`.

## Motivation

Resource providers define their APIs using TypeSpec or Swagger, both of which allow types that do not fit neatly into any ARM primitive type. For example, most properties of the [Microsoft.DataFactory/factories/datasets@2018-06-01](https://learn.microsoft.com/en-us/azure/templates/microsoft.datafactory/2018-06-01/factories/datasets?pivots=deployment-language-bicep) resource are untyped, and CPU allocations for containers and virtual machines can be specified as a string (`"0.25"`) or a number (`0.25`). Bicep template authors are currently blocked from using such properties in resource-derived types, because such types must fit with one ARM primitive type. Template authors also cannot accurately model such resources with user-defined type syntax.

## Detailed design

### Client side changes

Bicep will define a new ambient type symbol named `any` with a type of `LanguageConstants.Any`. This symbol may be used wherever type syntax is expected.

When converting type syntax to a template schema node during compilation, any type that does not fit into a single ARM primitive type will result in the `type` constraint being omitted.

### Server side changes

Type definitions in ARM are expressed as a set of constraints to apply to values. The type grammar is similar to (and was indeed largely based on) JSON schema. Currently, the `"type"` constraint is required for all schema nodes that do not have a `"$ref"` property. Under this proposal, the `"type"` constraint would become optional like other constraints in ARM schema nodes. This would follow the example of JSON schema, where `"type"` may be omitted, in which case a value may be of any type.

### Examples

- Example using `any` symbol
```bicep
param foo any

func getUnlessOmitted(container object, key string, defaultValue any) any => contains(container, key) ? container[key] : defaultValue
```

```json
{
    ...
    "parameters": {
        "foo": {}
    },
    "functions": [
        {
            "namespace": "__bicep",
            "members": {
                "getUnlessOmitted": {
                    "parameters": [
                        {
                            "type": "object",
                            "name": "container"
                        },
                        {
                            "type": "string",
                            "name": "key"
                        },
                        {
                            "name": "defaultValue"
                        }
                    ],
                    "output": {
                        "value": "[if(contains(parameters('container'), parameters('key')), parameters('container')[parameters('key')], parameters('defaultValue'))]"
                    }
                }
            }
        }
    ]
}

- Example using resource-derived types
```bicep
type foo = {
  metadataProperty: resourceInput<'Microsoft.Authorization/policyAssignments@2025-01-01'>.properties.metadata
}
```

```json
{
    ...
    "definitions": {
        "foo": {
            "type": "object",
            "properties": {
                "metadataProperty": {
                    "metadata": {
                        "__bicep_resource_derived_type!": {
                            "source": "Microsoft.Authorization/policyAssignments@2025-01-01#/properties/metadata"
                        }
                    }
                }
            }
        }
    }
}
```

## Drawbacks

### Frankentypes

Because many constraints in ARM can only be applied to certain kinds of values, this proposal would allow the creation of odd schemata, such as:

```json
{
  ...
  "parameters": {
    "foo": {
      "minValue": 0,
      "maxValue": 99,
      "minLength": 1,
      "maxLength": 10,
      "additionalProperties": {
        "type": "string"
      }
    }
  }
}
```

`minValue`/`maxValue` will only be enforced against whole numbers, `minLength`/`maxLength` will only be enforced against strings and arrays, and `additionalProperties` will only be enforced against objects. This is [an intentional feature of JSON schema](https://github.com/json-schema/json-schema/issues/172#issuecomment-114076650), but it's important to note because it might be surprising.

### Backdoor floats

It is currently impossible to pass a non-integral number (e.g., `0.75`) as a parameter or output because such a value does not fit into any ARM primitive type, although it is possible to pass such a value wrapped in an object (which, without a `properties` or `additionalProperties` constraint, is equivalent to `{*: any}`) or an array (which, without an `items` or `prefixItems` constraint, is equivalent to `any[]`). We should decide whether/how we want existing constraints to apply to float values before making `"type"` optional, as a naive implementation will allow `minValue`/`maxValue` to target float values with no effect.

## Alternatives

ARM JSON template could use an explicit `"type": "any"` instead of merely omitting the `"type"` constraint. This could make the "frankentypes" discussed above easier to understand but is more verbose and deviates from JSON schema's example.

## Rollout plan

The backend change involves no changes beyond removing a validation check. I would propose that this not be gated under a feature flag or minimum template language version.

## Unresolved questions

> - Should `minValue`/`maxValue` be updated to apply to floats?
