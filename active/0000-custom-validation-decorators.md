---
REP Number: <Fill me in with a four-digit number matching the pull request number; Update AFTER PR is approved and BEFORE is merged.>
Author: jeskew (Jonathan Eskew)
Start Date: 2024-10-29
Feature Status: Public Preview
Bicep Issue Number(s): 2922
---

<!-- Remove this comment and the prompts (in the form of blockquotes) for each section before submitting your PR -->

# Custom validation decorators

## Summary

Template authors should be able to define custom decorators using Bicep's user-defined function syntax. As a first
application of this affordance, template authors should be able to declare custom validation decorators that may be
applied anywhere a type decorator such as `@minValue(...)` is accepted. The identifier between the `@` and `(`
characters MUST refer to a user-defined function that accepts at least one argument corresponding to the targeted type
and returns a value assignable to the following type:

```bicep
@discriminator('kind')
type validatorReturn =
  | { kind: 'success' }
  | { kind: 'failure', errorMessage: string }
```

At deploy time, the ARM engine will invoke the user-defined function with the value to be validated along with any
specified additional arguments and inspect the function's return value to determine whether the validation passed.

## Terms and definitions

* **Deployment boundaries** - junctures within a deployment or template that have different expression evaluation
environments on each side. In ARM, deployment boundaries exist surrounding inner-scoped nested deployments and
surrounding user-defined functions.

## Motivation

Bicep/ARM type constraints are powerful mechanisms that enforce expectations about what shape data will take at
structural boundaries within a deployment. However, they are limited in what they can enforce to a small number of
predefined constraint types. Allowing user-defined functions to be invoked would permit users to apply whatever
checks they need (so long as they can be expressed via ARM's built-in functions) before allowing a value to cross a
boundary within their deployment.

## Detailed design

Any user-defined function that accepts at least one argument and returns a value assignable to
`{ kind: 'success' } | { kind: 'failure', errorMessage: string }` MAY be used by a template author as a type decorator
to enforce a custom validation gate

### Client side changes

Any user-defined function or imported function that accepts at least one argument and returns a value assignable to
`{ kind: 'success' } | { kind: 'failure', errorMessage: string }` will be assigned the `FunctionFlags.TypeDecorator`
flag and loaded as both a function and a decorator. The decorator's attachable type will be the declared type of the
function's first argument.

When the compiler encounters a decorator that is a user-defined function, it will generate the user-defined constraint
ARM JSON syntax described in the next section.

The Bicep compiler SHOULD NOT make any attempt to refine or narrow a type based on the user-defined validation
decorator. User-defined functions are too flexible to allow refining inferences to be drawn, and combining such
inferences would be a difficult task that could never be declared finished.

### Server side changes

ARM will add a new constraint named `userDefinedConstraint` whose value is an object with three properties:
1. *namespace* -  the function namespace (**required**)
2. *name* - the function within the specified namespace (**required**)
3. *additionalArguments* - an array of additional arguments to pass to the specified function
If the namespace is not found or contains no function with the provided name, then a `TemplateValidationException` will
be thrown.

The constraint validation engine will invoke the designated function, passing the value being validated the first
function argument, followed by any specified additional arguments. The engine will inspect the value returned by the
function invocation to determine whether the constraint was violated. If the value is an object with a `kind` property
that has a value of `'success'`, the validation passes and the engine reports no violation. If the `kind` property has a
value of `'failure'`, a constraint violation with the the specified error message is thrown. If the return value is not
an object OR the `kind` property is missing or has any value other than `'success'` or `'failure'` OR if a message with
a `kind` of `'failure'` does not have a string property named `errorMessage`, then an `TemplateValidationException` with
a localizable message indicating that a custom validator returned an invalid value will be thrown.

The constraint requires the `AggregateTypeValidation` template language feature in order to be used. As of the time of
writing, this means that the custom validator would be usable in language versions:
* 1.9-experimental
* 1.10-experimental
* 2.0
* 2.1-experimental
* 2.2-experimental

### Examples

- Using a locally defined validator
```bicep
func myLocalValidator(arg string, extraArg int) object => { kind: 'success' }

@myLocalValidator(10)
param p string
```

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "languageVersion": "2.0",
  "functions": [
    {
      "namespace": "__bicep",
      "members": {
        "myLocalValidator": {
          "parameters": [
            {
              "name": "arg",
              "type": "string"
            },
            {
              "name": "extraArg",
              "type": "int"
            }
          ],
          "output": {
            "type": "object",
            "value": {
              "kind": "success"
            }
          }
        }
      }
    }
  ],
  "parameters": {
    "p": {
      "type": "string",
      "userDefinedConstraint": {
        "namespace": "__bicep",
        "name": "myLocalValidator",
        "additionalArguments": [10]
      }
    }
  },
  "resources": {}
}
```

- Using an imported validator
```bicep
import { anImportedValidator } from 'shared.bicep'

@anImportedValidator()
param p string
```

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "languageVersion": "2.0",
  "functions": [
    {
      "namespace": "__bicep",
      "members": {
        "anImportedValidator": {
          "parameters": [
            {
              "name": "arg",
              "type": "string"
            }
          ],
          "output": {
            "type": "object",
            "value": {
              "kind": "success"
            }
          }
        }
      }
    }
  ],
  "parameters": {
    "p": {
      "type": "string",
      "userDefinedConstraint": {
        "namespace": "__bicep",
        "name": "anImportedValidator"
      }
    }
  },
  "resources": {}
}
```

- Using a wildcard-imported validator
```bicep
import * as shared from 'shared.bicep'

@shared.anImportedValidator()
param p string
```

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "languageVersion": "2.0",
  "functions": [
    {
      "namespace": "_1",
      "members": {
        "anImportedValidator": {
          "parameters": [
            {
              "name": "arg",
              "type": "string"
            }
          ],
          "output": {
            "type": "object",
            "value": {
              "kind": "success"
            }
          }
        }
      }
    }
  ],
  "parameters": {
    "p": {
      "type": "string",
      "userDefinedConstraint": {
        "namespace": "_1",
        "name": "anImportedValidator"
      }
    }
  },
  "resources": {}
}
```

## Drawbacks

Users will need to declare the `validatorReturn` type or return an untyped object whenever they declare a custom
decorator.
