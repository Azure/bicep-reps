---
REP Number: 0010
Author: jeskew (Jonathan Eskew)
Start Date: 2024-10-29
Feature Status: Public Preview
Bicep Issue Number(s): 2922
---

# Custom validation decorators

## Summary

Template authors should be able to define custom decorators using Bicep's lambda syntax. Bicep will add a `@validate()`
decorator function that may be applied anywhere a type decorator such as `@minValue(...)` is accepted. The decorator
will accept a single lambda argument. This lambda must itself accept a single argument whose type is compatible with the
target of the decorator and must return a boolean value. A second variant of this function will accept an even number of
arguments, where each odd-numbered argument is a lambda as described above, and each even-numbered argument is a string
to use as the error message if the immediately preceding lambda returns `false`.

At deploy time, the ARM engine will invoke the lambda with the value to be validated. A return value of `true` indicates
that the validation passed, and a return value of `false` indicates that the validation failed.

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

A `@validate()` decorator MAY be used by a template author as a type decorator to enforce a custom validation gate.

### Client side changes

A new decorator function named `validate` will be added to the `sys` namespace and assigned the
`FunctionFlags.TypeDecorator` flag. The `validate` function has one required argument and one optional argument. The
first argument is a lambda of type `T => bool`, where `T` is the declared type of the statement targeted by the
decorator. The second argument is a string that contains the error message that will be emitted if the validation lambda
returns `false`. The function will accept any number of addtional `T => bool, string` argument pairs. The decorator's attachable type will be the declared type of the
function's first argument. Any additional arguments must be supplied when the decorator is used.

```bicep
// usage with error message omitted
@validate(x => contains(x, 'foo'))
param foo string

// usage with error message supplied
@validate(x => contains(x, 'foo'), 'bar must contain the substring \'foo\'')
param bar int

// usage with multiple validation lambdas
@validate(
  x => startsWith(x, 'fo'), 'baz must start with the letters \'fo\'',
  x => endsWith(x, 'd'), 'baz must end with the letter \'d\'',
  x => length(x) == 4, 'baz be exactly 4 letters')
param baz string
```

When the compiler encounters the `validate` decorator, it will generate the `"validate"` constraint ARM JSON syntax
described in the next section.

The Bicep compiler SHOULD NOT make any attempt to refine or narrow a type based on the user-defined validation
decorator. Lambdas are too flexible to allow refining inferences to be drawn, and combining such inferences would be a
difficult task that could never be declared finished.

#### Applicability

As a type decorator, `@validate()` may target the following kinds of statements:

1. `type` statements
1. `param` statements
1. `output` statements
1. `var` statements with a declared type
1. Type property declarations within any of the above

### Server side changes

ARM will add a new constraint named `validate` whose value is an array. This array MUST have at least one argument, and
MAY have any even, non-zero number of arguments.

Each odd numbered argument MUST be a string containing a function expression. The function invoked must be `"lambda"`.
The function MUST be called with exactly two arguments:
1. A string declaring the name of the single lambda variable corresponding to the value being validated.
1. An evaluable expression.

Each even-numbered argument MUST be a non-evaluable string.

If the function invoked with an invalid number of arguments or arguments that do not match the descriptions above, then
a `TemplateValidationException` will be thrown.

The constraint validation engine will invoke each supplied lambda, passing the value being validated the single function
argument. The engine will inspect the value returned by the invocation to determine whether the constraint was violated.
If the returned value is `true`, the validation passes and the engine reports no violation. If the returned value is
`false`, a constraint violation will be raised (i.e., the deployment will fail with an error code of "InvalidTemplate").
If the return value is not a boolean, then an `TemplateValidationException` with a localizable message indicating that a
custom validator returned an invalid value will be thrown.

#### Constraint violation message

For a value with a template author-supplied error string, the message will be, "The provided value for the
`value type` `value path` is not valid. The value was rejected by a custom validation predicate with the following
message: '`message`.'".

For a value with no template author-supplied error string, the message will instead be, "The provided value for the
`value type` `value path` is not valid. The value was rejected by a custom validation predicate."

Both message variants will be localized.

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

While under development, the constraint will also require the `UserDefinedConstraints` template language feature. This
feature will only be added to language versions `2.1-experimental` and `2.2-experimental`.

#### Future evolution

If the deployments engine ever needs to iterate on this design (in order to, for example, allow multiple messages,
warning messages that log a deployment diagnostic but don't cause the deployment to fail, etc.), the engine can
determine what kind of handling to apply based on the JTokenType of the return value. For example, if the return value
has a token type of `Null` or `String`, the v1 contract (what is described in this REP) will be used, but if the return
value has a token type of `Array`, the v2 contract will be used.

### Examples

- Applied to a parameter
```bicep
@validate(x => true)
param p string
```

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "languageVersion": "2.0",
  "parameters": {
    "p": {
      "type": "string",
      "validate": [
        "[lambda('x', true())]"
      ]
    }
  },
  "resources": {}
}
```

- Applied to a type
```bicep
@validate(x => contains(x, 'x'))
type myType = string

param p myType
```

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "languageVersion": "2.0",
  "definitions": {
    "myType": {
      "type": "string",
      "validate": [
        "[lambda('x', contains(lambdaVariables('x'), 'x'))]"
      ]
    }
  },
  "parameters": {
    "p": {
      "$ref": "#/definitions/myType"
    }
  },
  "resources": {}
}
```

## Drawbacks

* Custom validation constraints cannot be directly exported and imported, although types with custom validators attached
may be.
* Unlike `@min/maxValue()`, `@min/maxLength()`, and `@allowed()`, we wouldn't be able to use a custom validator to
refine the type of a parameter. In this sense, custom validators are more like the proposed `@pattern()` validator than
they are like existing validators.

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
