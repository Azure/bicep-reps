---
REP Number: '0018'
Author: anthony-c-martin (Anthony Martin)
Start Date: 2025-03-18
Feature Status: Public
---

# Define an API between parameters and tooling

## Summary

Introduce a new function specific to `.bicepparam` files, for the sole purpose of making it possible for external tooling to supply values after compiling to `.json` parameters.

## Motivation

When a tool like Azure CLI invokes Bicep programatically to submit a deployment, there are 2 distinct steps - the compilation from `.bicepparam` to `.json`, and the submitting of the Deployment to the API. We've been able to work around this with features like `readEnvironmentVariable` and inline parameters, but the API between tool and Bicep CLI is clunky and can lead to problems.

The challenge that this spec sets out to solve is that there is no way to (a) explicitly have Bicep request input from an external tool and (b) express unresolved data and expressions purely within a parameter file. This creates a problem for Microsoft-internal tooling in particular, where it's necessary to separate out the compilation and deployment.

> [!NOTE]
> Some of the examples given in this spec may feel overly clunky or simplistic. I'm intentionally avoiding any references to the above-mentioned internal tool, and have made up `sys.envVar` and `sys.cliArgument` to instead illustrate the mechanics.

## Detailed design

### Overview
1. Introduce a new `.bicepparam` function named `externalInput`.

1. Introduce a "externalInputs" field to the JSON parameters file.

1. Introduce a "externalInputs" property to the Deployments API, such that the above information can be captured and sent on the API.

1. Give the Deployment Engine some limited abilities to evaluate expressions inside parameters.

### externalInput Function

When a tool like Azure CLI invokes Bicep programatically to submit a deployment, there are 2 distinct steps - the compilation from `.bicepparam` to `.json`, and the submitting of the Deployment to the API. We've been able to work around this with features like `readEnvironmentVariable` and inline parameters, but the API between tool and Bicep CLI is clunky and can lead to problems.

The challenge that this spec sets out to solve is that there is no way to (a) explicitly request input from an external tool and (b) express unresolved data and expressions purely within a parameter file. This creates a problem for Microsoft-internal tooling in particular, where it's necessary to separate out the compilation and deployment.

The proposed `externalInput` function has the following structure:
```
externalInput(<input_name>[, <input_config>])
```

With the following arguments:
* Argument 1: The name of the input. This value is opaque to Bicep - it is up to the external tool to define the valid input names it understands. We recommend a standard naming convention for this to minimize the risk of conflicts (camel-case alphanumeric with `.` characters).
* Argument 2 (optional) The configuration for the input. This is also opaque to Bicep.

Both arguments must be compile-time constants.

#### Examples

Reading an env var:
```bicep
var myEnvVar = externalInput('sys.envVar', 'MY_ENV_VAR')
```

Reading an CLI argument:
```bicep
var myCliArg = externalInput('sys.cliArgument', 'cli-arg')
```

### Supporting inputs in JSON parameters file

To support this feature, we must introduce a new property "externalInputs" to the JSON parameters file schema. The structure of this property is intentionally simple for external tools to parse and understand.

The schema of this new field can be described in TypeScript as:
```ts
type externalInputs = {
  [name: string]: {
    type: string,
    config: any,
  },
};
```

Example:
```json
{
  "parameters": {
    "foo": { "value": "bar" }
  },
  "externalInputs": {
    "myEnvVar": {
      "type": "sys.envVar",
      "config": "MY_ENV_VAR"
    },
    "myCliArg": {
      "type": "sys.cliArgument",
      "config": "cli-arg"
    }
  }
}
```

### Supporting inputs in the API
The Deployments API will require 2 new fields in the "properties" section of a Deployment:

"externalInputs". This will exactly match the structure defined above in the parameters file.

"externalInputValues". This purely exists to contain the actual values of the inputs. Because of the potential for returning secure data, this field will only be supported on a PUT, and will be stripped out of the GET response.

The schema of this new field can be described in TypeScript as:
```ts
type externalInputValues = {
  [name: string]: any,
};
```

#### Example
```json
{
  "template": ...,
  "parameters": {
    "foo": { "value": "bar" }
  },
  "externalInputs": {
    "myEnvVar": {
      "type": "sys.envVar",
      "config": "MY_ENV_VAR"
    },
    "myCliArg": {
      "type": "sys.cliArgument",
      "config": "cli-arg"
    }
  }
}
```

### Other API changes
This proposal requires changes to the Deployments API such that expressions can be resolved at deployment time.

Currently, individual parameters can either contain "value" (for a concrete value) or "reference" (for a keyvault reference). I'm proposing we add a 3rd - "expression".

#### Example
```json
{
  "parameters": {
    "foo": { "expression": "[createObject('bar', 'baz')]" }
  }
}
```

I believe this functionality could also be extended to access runtime information like `environment()` or `tenant()`, which are popular asks. However, for this proposal, I'm only planning on implementing the "built-in" expression functions as well as `externalInput`.

### Putting it all together

1. User asks tool to deploy a `.bicepparam` file containing `externalInput` functions.
1. Bicep converts the `.bicepparam` file to JSON, retaining the configured inputs in the "externalInputs" section.
1. The tool reads the "externalInputs" file and processes all of the requested inputs - providing errors if it doen't know how to reolve them.
1. The tool builds the "externalInputValues" portion of the requet containing the resolved values of the inputs. It sends the template, parameters, externalInputs and externalInputValues property to the Deployments API.
1. The Deployments API evaluates language expressions for each parameter with an "expression" property defined, using the externalInputValues property to handle the `externalInput` function evaluation.
1. Deployments API removes the "expression" property and sets the "value" field for resolved parameters.

## Examples

### Simple env var reading
Bicep authoring:
```bicep
using 'main.bicep'

param foo = externalInput('sys.envVar', 'MY_FOO_VAR')
```

Generated JSON parameters file:
```json
{
  "parameters": {
    "foo": {
      "expression": "[externalInputs('0')]"
    }
  },
  "externalInputs": {
    "0": {
      "type": "sys.envVar",
      "options": "MY_FOO_VAR"
    }
  }
}
```

API Request:
```json
{
  "template": "...",
  "parameters": {
    "foo": {
      "expression": "[externalInputs('0')]"
    }
  },
  "externalInputs": {
    "0": {
      "type": "sys.envVar",
      "options": "MY_FOO_VAR"
    }
  },
  "externalInputValues": {
    "0": "my foo env var"
  }
}
```

## Other options considered
Another option is to use a statement rather than a function - e.g.:
```bicep
input <symbolic_name> '<type_name>' [<optional_type>] [with {
  // config
}]
```

This option has the following pros and cons:

Pros:
* Obvious way of attaching a type to an input.
* Fits more clearly with the JSON parameters format.
* Has a "name" that can be used to identify it in the JSON format.

Cons:
* More typing, because the `with` block would require any config to be wrapped in an object.
* Combining it with other functions feels more clunky - e.g. it would be nice to be able to write;
    ```bicep
    var myObject = json(getBinding('sys.envVar', 'MY_ENV_VAR'))
    ```
