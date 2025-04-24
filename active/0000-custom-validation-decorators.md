---
REP Number: <Fill me in with a four-digit number matching the pull request number; Update AFTER PR is approved and BEFORE is merged.>
Author: jeskew (Jonathan Eskew)
Start Date: 2024-10-29
Feature Status: Public Preview
Bicep Issue Number(s): 2922
---

# Custom validation decorators

## Summary

Template authors should be able to define custom decorators using Bicep's user-defined function syntax. As a first
application of this affordance, template authors should be able to declare custom validation decorators that may be
applied anywhere a type decorator such as `@minValue(...)` is accepted. The identifier between the `@` and `(`
characters MUST refer to a user-defined function that accepts at least one argument corresponding to the targeted type
and returns a value of type `string?`.

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

Any user-defined function that is targeted by the `@validator()` decorator MAY be used by a template author as a type
decorator to enforce a custom validation gate.

### Client side changes

A new decorator function named `validator` will be added to the `sys` namespace. Like `export`, this new decorator
function will be consumed by the compiler rather than being emitted in the compiled template. The decorator may only
target user-defined functions that:

* Accept one or more arguments, and
* Have a return type of `string?`

Any user-defined function function targeted by the new decorator will be assigned the `FunctionFlags.TypeDecorator`
flag and loaded as both a function and a decorator. The decorator's attachable type will be the declared type of the
function's first argument. Any additional arguments must be supplied when the decorator is used.

```bicep
@validator()
func mustContain(toValidate string, requiredSubstring string) string? => contains(toValidate, requiredSubstring) ? null : '${toValidate} does not contain ${requiredSubstring}'

@mustContain('hello')      // <-- no error
param foo string


@mustContain('hello')      // <-- error because param type is not assignable to mustContain's first arg type
param bar int

@mustContain()             // <-- error because mustContain requires a second argument
param baz string
```

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
function invocation to determine whether the constraint was violated. If the returned value is `null`, the validation
passes and the engine reports no violation. If the returned value is a string, a constraint violation with the specified
error message is raised (i.e., the deployment will fail with an error code of "InvalidTemplate" and the supplied
message). If the return value is not a string or `null`, then an `TemplateValidationException` with a localizable
message indicating that a custom validator returned an invalid value will be thrown.

#### Expected engine behavior

The constraint validation engine will invoke the designated function for all non-null parameter values. The function
will be invoked on:

* Parameter values supplied as an input to the deployment.
* Parameter values generated from the `"defaultValue"` parameter declaration property.

The function will **not** be invoked on:

* A value of `null` supplied as an input to the deployment.
* An parameter for which no value or default value is available.

The latter two cases are only permitted when the parameter is declared with `"nullable": true`, which is not permitted
on parameter declarations with a `"defaultValue"` property.

#### Required template language version

The constraint requires the `AggregateTypeValidation` template language feature in order to be used. As of the time of
writing, this means that the custom validator would be usable in language versions:
* 1.9-experimental
* 1.10-experimental
* 2.0
* 2.1-experimental
* 2.2-experimental

#### Future evolution

If the deployments engine ever needs to iterate on this design (in order to, for example, allow multiple messages,
warning messages that log a deployment diagnostic but don't cause the deployment to fail, etc.), the engine can
determine what kind of handling to apply based on the JTokenType of the return value. For example, if the return value
has a token type of `Null` or `String`, the v1 contract (what is described in this REP) will be used, but if the return
value has a token type of `Array`, the v2 contract will be used.

### Examples

- Using a locally defined validator
```bicep
func myLocalValidator(arg string, extraArg int) string? => null

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
            "type": "string",
            "nullable": true,
            "value": null
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
            "type": "string",
            "nullable": true,
            "value": null
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

User-defined functions can only consist of a single expression, and validator functions will need to use at least one
ternary expression unless they wish to always succeed or always fail. Some form of `switch` or `match` expression would
make this feature easier to use.

## Overlap with other features

### `fail()`

There is some overlap with the `fail()` function, which works like a `throw` expression in C#. However, `fail()` must
be used within a statement and therefore typically requires an extra `var` declaration. For example,

```bicep
param foo string
var _throwaway1 = equals(foo, 'foo') ? fail('A `foo` value of \'foo\' is not allowed') : null
```

`var` statements need a unique name, which makes this usage pattern somewhat unergonomic. The template author's intent
is also somewhat unclear.

### `assert`

There is considerable overlap with the `assert` statement:

```bicep
param foo string
assert fooDoesNotEqualFoo = !equals(foo, 'foo')
```

The main reason to offer an alternative to `assert` here is that assertions are still an experimental feature, and there
is no concrete timeline for their general availability at this point. `assert` statements also create named symbols, and
it is this symbol name (rather than an error message) that is reported back to the user when an assertion fails. This
means (in Bicep, at least) that assertion failure error messages must be valid identifiers.
