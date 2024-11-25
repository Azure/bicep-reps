---
REP Number: 0006
Author: kalbert312 (Kyle Albert)
Start Date: 2024-08-27
Feature Status: Public Preview
Enhances: 0008
---

# Integrating extensible resources with deployment stacks

## Summary

This document describes changes to how extensions in Bicep and ARM templates are defined in order to support managing
extension resources with Deployment stacks.

## Terms and definitions

- `Control plane`: An Azure or non-Azure API for managing the lifecycle of resources. This is the API for provisioning
  or deleting resources.
- `Extension resource`: An Azure data-plane resource or non-Azure resource declared in a Bicep file or an ARM template.
- `Extension configuration`: Data needed to connect to a non-Azure control plane to manage resources. This can contain
  secrets, such as a Kube config for AKS clusters.
- `Deployment stack`/`stack`: An atomic unit for lifecycle management of a set of resources deployed by a Bicep or ARM
  template deployment. Resources can be Azure resources or extension resources. Stacks can unmanage resources which
  results in either their deletion or detachment from the stack.
- `Secure`/`secret`/`auth`: These terms may be used interchangeably to indicate secure portions of data for extension
  configurations in particular.
- `Non-secure`: Any data that is not considered a secret that can be persisted or exposed without risk.

## Motivation

Currently, Deployment stacks do not support managing extension resources deployed through one. Users that utilize
Deployment stacks will want to manage the lifecycle of any resource deployed through a stack regardless of its control
plane as this is a key feature of stacks.

In order to minimally support deletion of extension resources which is the goal of this integration, changes need to be
made at both the Bicep and ARM template levels to how extension configurations are defined so the Deployment stack
service can retrieve necessary secret data in a secure and direct way.

The current implementation of extension configurations in Bicep and ARM templates is too flexible. For example, it
allows users to consume template parameters and use template language expressions to manipulate data. This might sound
great from a user experience perspective, but this creates a problem for the stacks service because it must be able to
reevaluate the extension configuration at a later time when resource deletion occurs. Non-stack deployments do not have
this problem because the configuration is only needed at the time of deployment. The extension configuration can change
over time due to credential rotations or other reasons that are opaque to the Stacks service. Therefore, it is necessary
to constrain extension configurations to use constants or simple directives the stacks service can act upon.

## Detailed design

The key design change is to move where extension configurations are defined in Bicep and ARM templates. Currently, the
extension configurations are defined within the deployment of the extension resources being deployed. This REP's
proposal is to move these configuration definitions to a dedicated deployment input property that is separate from
deployment parameters. See the examples section for Bicep and ARM template examples.

### Analyzing the current implementation

Why this change is necessary becomes more clear by observing what the stacks services would be required to do with the
current implementation. Consider this Bicep deployment:

```bicep
@secure()
param kubeConfigParam string

extension kubernetes with { // this will produce a diagnostic if the config is not provided
  kubeConfig: kubeConfigParam
  namespace: 'default'
}

// ... extension resources
```

In this example, the extension configuration contains a secret from a deployment parameter. A deployment parameter can
be supplied by the user or sourced from a parent deployment. In either case, it is a user secret and thus cannot be
persisted long term by stacks for use in future resource deletion.

In the case where it's supplied by the user and without secret persistence, the only way the stacks service could
possibly source this data again is to prompt the user for it. It is unclear how that experience would work. Does the
user need to supply the entire old deployment's parameters back to the stack service? How do they know which version of
inputs to supply? What if a subset of those parameters are no longer relevant? Would they only need to supply parameters
used in configurations? How would we detect and differentiate that across an entire tree of deployments?

In the case where it's supplied by a parent deployment, the problem becomes recursive. It's either provided by the user,
is sourced from somewhere else in that parent deployment (such as another resource's outputs like a key vault or AKS
cluster), or sourced from another parent deployment.

To complicate this further, any pathway between the secret's source and consumption can have dynamic language
expressions such as template function calls or branching. This would effectively require stacks to generate a separate
deployment template at deletion time from the user's deployment template to process these language expressions.

The stacks service cannot store user secrets, and it should not evaluate dynamic deployment template language
expressions. The stacks service needs extension configurations to be defined in a format that is interpretable at
deletion time that is transparent to the user and secure.

### The new design from the Bicep perspective

The following Bicep code illustrates the new design for extension configurations. The root deployment supplies the
extension configuration details to the child deployment, allowing direct access to ARM resources in the root deployment
that supply secrets.

main.bicep - The root deployment

```bicep
param namespace string = 'default'

// The ARM aksCluster resource can supply kubeConfigs for the AKS Kubernetes use case.
resource aksCluster '...' = {
  // ...
}

module extResources 'extResources.bicep' = {
  name: 'extResourcesDeploymentName'
  extensionConfigs: {
    k8s: {
      auth: {
        kubeConfig: aksCluster.listClusterAdminCredential().kubeconfigs[0].value
      },
      namespace: namespace
    }
  }
  params: {
    // ...
  }
}
```

extResources.bicep - Child deployment with extension resources

```bicep
extension kubernetes as k8s // optional alias

extension microsoftGraph // No config is needed for this one. This extension uses OAuth OBO.

// ... extension resources
```

The keys of the "extensionConfigs" object are the extension aliases of the module deployment. If an alias is not
provided, it will be the extension name. The values of the object are extension configurations. Optionality of
configuration is determined by the extension. If this is compared to a typical resource group deployment, it is aligned
because the resource group being deployed to is not defined within the deployment itself, but as a parameter to the
deployment, either through the CLI parameters or as a scope for nested deployments. In this example, the deployment
scope(s) are being defined as extension configurations. Expressions in this object can be constrained to expressions
that can be compiled to simple directives that the stacks service can execute.

The "auth" property at the root level of an extension configuration is a reserved property across all extensions. If an 
extension supports non-Azure based auth that requires passing raw credentials to it from a deployment, the extension 
must define a schema for the "auth" property. The "auth" property schema must be an object and can have nested objects. 
Properties in the "auth" object follow strict validation rules when the deployment occurs as a Deployment stack deployment.
There are no specialized validation rules for non-Deployment stack deployments. 

Below are a couple tables outlining allowable expression value types for extension configuration properties. Note that
expression value type means the type of the final value of an evaluated expression. This means that `'aString'`
and `boolParam ? 'aString' : 'bString'` are the same expression value type.

Legend: 
- ✅: Allowed expression value type
- ❌: Disallowed expression value type
- 💾: Resolved expression value saved long-term for later consumption by Deployment stacks
- 🔄️: Directive to re-fetch expression value is saved long-term for later consumption by Deployment stacks

Expression types inside the "auth" object:

| Expression Value Type                  | Non-stack deployment  | Stack deployment |
|----------------------------------------|:---------------------:|:----------------:|
| Literal value                          |           ✅           |        ❌         |
| Non-secure deployment parameter        |           ✅           |        ❌         |
| Secure deployment parameter            |           ✅           |        ❌         |
| Key vault reference                    |           ✅           |      ✅ 🔄️       |
| Resource output property (reference()) |           ✅           |        ❌         |
| LIST* output property                  |           ✅           |      ✅ 🔄️       |
| All other expression value types       |           ✅           |        ❌         |

Expression types NOT inside the "auth" object:

| Expression Value Type                  | Non-stack deployment | Stack deployment |
|----------------------------------------|:--------------------:|:----------------:|
| Literal value                          |          ✅           |       ✅ 💾       | 
| Non-secure deployment parameter        |          ✅           |       ✅ 💾       |
| Secure deployment parameter            |          ✅           |        ❌         |
| Key vault reference                    |          ✅           | ❌ <sup>(1)</sup> |
| Resource output property (reference()) |          ✅           |       ✅ 💾       |
| LIST* output property                  |          ✅           | ❌ <sup>(2)</sup> |
| All other expression value types       |          ✅           |       ✅ 💾       |

<sup>(1)</sup> Stacks cannot persist secrets which is what Key vaults are designed to hold, so we'd need to persist 
the directive. For the time being, Stacks will only support re-fetching "auth" properties because of this and that
secrets are more likely to change over time. This encourages an anti-pattern for key vaults by using them for
non-secret data, but this is probably not something the Deployments team needs to be concerned about. This could be
permissible but there should be a way for the user to differentiate saving resolved values and saving the directive to
re-fetch the value. Example: `namespace: kv.getSecret('xyz')` vs `namespace: () => kv.getSecret('xyz')`. In the 
first case, the reference is resolved and the resolved value is persisted. In the second example, the reference is 
persisted.

<sup>(2)</sup> Disallowed because of the save vs re-fetch workflow difference between "auth" and non-"auth". There 
should be a way to differentiate these behaviors by the Bicep/ARM template author. See <sup>(1)</sup>. Alternatively
this can be enabled to allow sourcing non-secret data and if secret data is used, it is user error.

The reasoning behind the disallowed expressions for stack deployments is because the extension configuration properties
will be evaluated by the deployment engine and the evaluation results will be returned in the deployment GET response.
The stacks service will persist this data long-term and use it for the next stack update or deletion that requires
extension resource deletions. These constraints will make it required that auth properties ("kubeConfig" in the
Kubernetes example) are provided in way where a directive to re-fetch can be computed and saved. It also prohibits 
usage of Bicep functions such as `loadFileContent` to inline load secret content.

The stacks service already sends a stack resource ID HTTP request header to the deployments service. This will be used
to enable additional validation in the Deployment engine to enforce these expression rules. It will also enable the
return of extension configurations in the Deployment GET response for API versions that support it. This header is only
needed for the initial deployment PUT, as its value will be saved on the deployment metadata and this will be used to
drive the API behavior. For deployments that are not initiated by the stack service, the configuration portion of the
extension details will not be returned, as no additional validation on language expressions is done, which means we
cannot guarantee compliance to the ARM RP RPC rule that a resource GET cannot return secret data.

Examples of expression allowance for the `kubeConfig` case:

```bicep
@secure()
param secureParam string

param nonSecureParam string

param boolParam bool

resource kv 'Microsoft.KeyVaults/keyVault@...' = {
  // ...
}

resource aksCluster '...' = {
  // ...
}

module extResources 'extResources.bicep' = {
  name: 'extResourcesDeploymentName'
  extensionConfigs: {
    k8s: {
      auth: {
        // NOTE: ❌'s and ❔'s only apply to stacks based deployments. Regular deployments should be able to use any 
        // valid Bicep expression.
        ✅ kubeConfig: aksCluster.listClusterAdminCredential().kubeconfigs[0].value
        ✅ kubeConfig: kv.getSecret(...)
        ❌ kubeConfig: loadFileContent('kubeconfig.txt')
        ❌ kubeConfig: secureParam
        ❌ kubeConfig: nonSecureParam
        ✅ kubeConfig: boolParam ? ... : aksCluster.listClusterAdminCredential().kubeconfigs[0].value // allowable, so 
        long as the branches that can be travelled down after short-circuiting are an allowable expression
        ❌ kubeConfig: userDefinedFunctionCall()
      }
    }
  }
  params: {
    // ...
  }
}
```

The expressions in question depend on how difficult it would be to detect usages of invalid expressions recursively. The
deployments engine could evaluate these expressions out to values, but it would need to be able to determine the source
of any data used to arrive at the final value to ensure secrets are not leaked.

##### Limitations for extension config expressions

**TODO(kylealbert): rework this for language expression based extension config**

The top-level object for an extension configuration must be expressed as an object literal (with brackets). No
conditional expressions, function calls, or other disruptive expressions are allowed. This allows for the ARM template
code generation to explicitly write out a top-level object with the properties of the extension config. The property
names must be known at authoring time. Object spreading in the object literal only supports spreading another extension.
For an example of this, see the section that addresses passing extension configs down to nested deployments.

#### Stacks mode for Bicep files

Because the stack service will enable additional validation and features in the deployment service, Bicep template
authors need to see diagnostics that flag disallowed expressions ahead of time. With the current Bicep file
architecture, this cannot be determined automatically. Bicep files can be used with or without stacks and modules bind
together different Bicep files to create a deployment hierarchy. To enable the additional diagnostics, they will need to
be explicitly enabled by the template author. Here is the design for how to enable this in a Bicep file via the Bicep
linter:

```json5
{
  "rules": {
    "stacks-extensibility-compatibility": { // break this down into smaller rules as needed
      "level": "warning",
      // The default level across all files. Up to user to configure as an error or to turn off.
      // Alternative to file overrides is to stick with in-file/line overrides.
      "fileOverrides": {
        "*.stack.bicep": "error",
        "<glob>": "error"
      }
    }
  }
}
```

There will be no ARM template changes to drive this. The extra validation on the backend will take place if the
deployment is sent from the Stacks service. The Bicep compilation step for Stacks CLI commands should run with this mode
set to "error" regardless of user setting to prevent templates that will certainly fail from being sent to the API.

#### Passing extension configurations to nested deployments

Users will need to reuse extension configurations across multiple nested deployments. Because these are passed as
deployment properties rather than as parameters, there needs to be a way to reference this configuration in the
consuming deployment for the nested deployment.

Here are some examples of passing a configuration down another level:

```bicep
extension kubernetes // this was provided a configuration through the parent deployment/parameters file

module foo 'foo.bicep' = {
  extensionConfigs: {
    kubernetes: kubernetes // resuse its config, must be from an extension with the same name and version
  }
}

// spread inheritance
module bar 'foo.bicep' = {
  extensionConfigs: {
    kubernetes: {
       ...kubernetes, // must be an extension with the same name and version
       namespace: 'myOtherNamespace'
    }
  }
}

// piecemeal inheritance
module baz 'foo.bicep' = {
  extensionConfigs: {
    kubernetes: { 
       namespace: kuberetes.namespace
       auth: kubernetes.auth // must be for an extension with the same name and version?
       // alternatively for auth, or other objects:
       auth: {
         kubeConfig: kubernetes.auth.kubeConfig
       } 
    }
  }
}
```

#### Providing extension configurations to root deployments

Because the extension configuration will be an input property to the deployment, there needs to be a way for frontends
to supply this data to a root deployment. The `bicepparam` file type can be expanded to support defining extension
configurations with the same expression rules as extension configuration object blocks in module definitions.

Here is an example:

main.bicepparam

```bicep
using 'main.bicep'

param boolParam = true
// ...

extensionConfigs {
  k8s: {
    auth: {
      ✅ kubeConfig: az.aksKubeConfig(...) // any data needed to get it. Overloads may depend on auth workflow.
      ✅ kubeConfig: getSecret(...)
      ❌ kubeConfig: loadFileContent('...')
      ❌ kubeConfig: 'inlinedSecret'
    }
    namespace: 'default'
  }
}
```

If extensions require a configuration, there must be a parameters file (Bicep or JSON) that provide the extension
configuration. If it is not supplied, there should be appropriate error diagnostics. There will be no support for
inlining extension configurations from CLI arguments.

### The new design from the ARM template perspective

#### Providing extension configurations to root deployments

An `"extensionConfigs"` object will be added to the JSON parameters file. There will be no support for inlining of 
extension configuration by CLI, so this is the only way to provide extension configurations.

Here is an example:

main.parameters.json

```json5
{
  "schema": "...",
  "parameters": {
    "boolParam": {
      "value": "true"
    }
  },
  "extensionConfigs": {
    "k8s": {
      "auth": {
        "kubeConfig": {
          "keyVaultReference": {
            "keyVault": {
              "id": "/..."
            },
            "secretName": "myKubeConfig"
          }
        },
        // OR
        "kubeConfig": {
          "apiReference": {
            "method": "POST",
            "armResourceId": "/subscriptions/.../resourceGroups/.../Microsoft.ContainerService/managedClusters/...",
            "apiVersion": "2024-02-01",
            "action": "listClusterAdminCredentials",
            "query": "a=1&b=2",
            "responseValuePath": "kubeconfigs[0].value"
          }
        }
      },
      "namespace": {
        "value": "default"
      }
    }
  }
}
```

The above format effectively defines the signature of the `"extensionConfigs"` object accepted by the deployment PUT API.
The `"extensionConfigs"` object is a mapping of extension aliases (as defined in the deployment template) to extension
configurations. Each extension configuration is an object. The properties of the object are defined by the extension schema.
For example, the Kubernetes extension defines `auth.kubeConfig` and `namespace`. The values of the properties are an
object with 3 different possible properties, all mutually exclusive from each other:

1. `"value"`: a literal value (string, number, bool, object, etc)
1. `"keyVaultReference"`: a reference to a key vault secret
1. `"apiReference"`: a reference to ARM resource API output

#### Nested deployment extension configurations

The format of the `"extensionConfigs"` deployment property is different for nested deployments because they are part of
a deployment template. The structure is the same except extension configurations are language expressions. The actual
extension configuration is derived from the language expression, so it must resolve to a valid configuration object for
the corresponding extension. The derived root object will have an `"auth"` property for any secure properties defined
by the extension configuration schema. The rest of the root object's properties will be non-secure properties as defined
by the extension configuration schema.

Below is an example ARM template followed by specifications for each property. Only REP relevant data is shown.

main.json - The root deployment

```json5
{
  // root deployment
  "template": {
    "resources": {
      "extResources": {
        // Nested deployment
        "type": "Microsoft.Resources/deployments",
        "apiVersion": "2022-09-01",
        "name": "extResourcesDeploymentName",
        "properties": {
          "extensionConfigs": {
            // Expressions are not allowed to reference secure parameters or provide data for config secrets in ways that put the secret security at risk.
            // This will be evaluated out to the deployment PUT format during preprocessing and deployment
            "k8s": "[createObject('namespace', parameters('namespace'), 'auth', createObject('kubeConfig', listClusterAdminCredential(resourceId('Microsoft.ContainerService/managedClusters', parameters('clusterName')), '2024-02-01').kubeconfigs[0].value))"
          },
          "template": {
            "extensions": {
              "k8s": {
                "name": "Kubernetes",
                "version": "1.0.0",
                "config": {
                  "namespace": {
                    "type": "string",
                    "defaultValue": "default"
                  },
                  "kubeConfig": {
                    "type": "secureString"
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

#### Specification for "non-auth" properties

Language expressions for non-secure property values are subject to the non-secure validation rules for Deployment stack
deployments as outlined the above Bicep design section. For stack deployments, non-secure properties are always resolved
once and the resolved value is persisted.

Any data provided for non-secure properties will be exposed via the deployment GET API for Deployments stack deployments.
Any future modification to built-in language expression functions will need to be considered for validation blocks to
prevent user secrets from leaking. Keeping an allow list of specific functions will not be realistic to maintain, and a
deny list for this feature will also be difficult to keep accurate, so it may be ideal to flag built-in functions as a
certain source of secure data. Alternatively, this can be left to the responsibility of the template author.

#### Specification for the "auth" property

This property is an object that contains all the secure properties for an extension config. It can only be omitted if
the extension schema does not declare secure/auth properties. If it is present, null is always an invalid value.

For stack deployments, language expressions within this property object tree for auth property values are subject to the
secure validation rules as outlined the above Bicep design section. Additionally for stack deployments, properties
within "auth" are always persisted as directives/references to be re-fetched when necessary by other stack operations
such as when deleting extensible resources.

#### Common specification for both the "non-auth" and "auth" properties

At template preprocessing time for Deployment stack deployments, the extension configuration language expression must
evaluate out minimally to an object with all of its property names recursively available. Non-evaluable expressions (
like reference calls) within the expression will be replaced with sentinel expressions in order to make the expression
evaluable. The newer What-If partial evaluator will be used to assist with this task. This is so preprocessing
validation can validate required properties have a value, address dangling config properties, merge default values, and
validate secure/non-secure rules. This means that reference call sourced objects cannot be used for extension config
property names and cannot be spread onto extension configs. Each sentinel expression will map back to the original
expression so it can be validated.

Preprocessing will do as much validation as it can, but for language expressions that are only resolvable in the
deployment jobs, errors will surface only during deployment, such as value sourced from a reference expression that is
of the wrong type. Because replacing non-evaluable portions affects short-circuiting, expressions that involve
short-circuiting/branching that prevent deriving the object property names may need to be restricted.

Using the above template example's "extensionConfigs", the final merged extension configuration for extension "k8s"
passed to the extension during deployment will be:

```json5
{
  "auth": {
    "kubeConfig": "<secret>"
  },
  "namespace": "default"
}
```

#### Passing extension configurations to nested deployments

There is an additional challenge when it is desired to pass an extension configuration down more than 1 level of
deployments. Deeply nested deployments should be able to inherit an extension configuration from a parent deployment. To
enable this, a new built-in ARM template function `extensionConfigs(string extAlias)'` is added to access parent
deployment extension configurations. The `extAlias` parameter is the extension alias as defined in the template. This
function will be usable in any template language expression.

Here is an example:

```json5
{
  // root deployment
  "template": {
    "resources": {
      "extResources": {
        // nested deployment (2nd level)
        "type": "Microsoft.Resources/deployments",
        "apiVersion": "2022-09-01",
        "name": "extResourcesDeploymentName",
        "properties": {
          "extensionConfigs": {
            // what we will inherit
            "k8s": "[createObject('auth', createObject('kubeConfig', listClusterAdminCredential(resourceId('Microsoft.ContainerService/managedClusters', parameters('clusterName')), '2024-02-01').kubeconfigs[0].value, 'namespace', parameters('namespace'))]"
          },
          "template": {
            "extensions": {
              "k8s": {
                "name": "Kubernetes",
                "version": "3.0.0",
                "config": {
                  "namespace": {
                    "type": "string",
                    "defaultValue": "default"
                  },
                  "kubeConfig": {
                    "type": "secureString"
                  }
                }
              }
            },
            "resources": {
              "extResources2": {
                // nested deployment (3rd level)
                "type": "Microsoft.Resources/deployments",
                "apiVersion": "2022-09-01",
                "name": "extResourcesDeploymentName2",
                "properties": {
                  "extensionConfigs": {
                    "kubernetes": "[extensionConfigs('k8s')]",
                    // other sample expression:
                    "kubernetes": "[createObject('auth', extensionConfigs('k8s').auth, 'namespace', extensionConfigs('k8s').namespace)]"
                  },
                  "template": {
                    "extensions": {
                      "kubernetes": {
                        // assume same extension name and version as "k8s", thus same schema, so it's valid to inherit from k8s
                      },
                    }
                  }      
                }
              }
            }
          }
        }
      }
    }
  }
}
```

When the DeploymentResourceJob runs on the third level nested deployment, there will be access to the second level
deployment entity. By this time, the extension configuration for the second level deployment is already evaluated which
enables substitution to occur.

In order to conform to the validation rules laid out in the Bicep design section, usages of the `extensionConfigs` function
in non-secure property value expressions will not be allowed to access the `"auth"` property of an extension configuration.

### Microsoft.Resources/deployments API changes

This REP extends upon the API changes in REP 0008 by adding a configuration value to the extensions returned from a
deployment GET. This configuration object will be similar to the configuration object described in the above ARM
template design changes but with template language expressions evaluated and compiled to a format that can be relatively
easily interpreted. The disallowed expression types will prevent sensitive data from showing up here. The configuration
will not be provided for non-stack deployments because there are no restricted expressions.

```json5
{
  // ... deployment contract
  "extensions": [
    {
      "name": "Kubernetes",
      "alias": "k8s",
      "version": "1.0.0",
      "deploymentId": "/subscriptions/.../resourceGroups/.../providers/Microsoft.Resources/deployments/...",
      "config": {
        // the language expressions provided to the PUT have been evaluated
        "namespace": {
          "type": "string",
          "value": "default"
        },
        "kubeConfig": {
          "type": "securestring",
          // or keyVaultReference
          "apiReference": {
            "method": "POST",
            "armResourceId": "/subscriptions/.../resourceGroups/.../Microsoft.ContainerService/managedClusters/...",
            "apiVersion": "2024-02-01",
            "action": "listClusterAdminCredentials",
            "query": "a=1&b=2",
            "responseValuePath": "kubeconfigs[0].value"
          }
        }
      }
    }
  ]
}
```

With the deployments service returning a list of all extensions (across all nested deployments) with their configuration
directives and returning all extension resources deployed in the template (across all nested deployments), the stacks
service can process each extension resource and use the "deploymentId" and "extensionAlias" properties as a composite
key to the data in the extensions array.

### Directive evaluation failures

The Deployments service will need to account for failures when executing directives. A deployment error will be returned
if a failure is encountered. Failures can include invalid secrets, insufficient permissions, API errors, null reference
exceptions (during JSON path evaluation), and unsupported evaluation sequences.

The Stacks service will also need to account for failures. Failures can include invalid secrets, insufficient
permissions, transient API errors, null reference exceptions (during JSON path evaluation), and unsupported evaluation
sequences (deployments enhancements made but not yet supported by stacks).

The errors should be clear to the user that it is something they can resolve, such as a stale user-provided secret, or
if it's something the deployments team needs to look into. Errors should not leave the stack in a state the blocks the
user from recovering.

## Drawbacks

Needing to trace value sources in configuration expressions can be complex and change over time, leading to potential of
secret data leaks. Therefore, allowed expression types should be very restrictive, at least for initial iterations.
There should also be sufficient diagnostics and validation to guard from workarounds the user might go for such as using
non-secure parameters for secret data.

## Alternatives

### Resource deployment parameters

The reason why extension configuration is moved from a template to the deployment input is mostly due to needing to be
able to reference ARM resources in a parent deployment as the source of truth for the configuration. It's not currently
possible to define a deployment parameter as a reference to an ARM resource in a way that can be analyzed at template
evaluation time. There are draft designs for this type of feature, but it is not available at the start time of this
document. Deployment parameters are currently limited to primitives and basic types that do not provide enough
engine-level metadata to make assertions about the source of a parameter value (for example, how could it be known that
a secure string parameter is always sourced from the AKS cluster admin credentials in the Kubernetes example?).

Resource deployment parameters, if designed in a way that enforces a certain resource type, would solve this issue, and
there wouldn't be need to move the extension configuration definition site. Having this available would also solve the
root deployment problem because the parameter would be like any other parameter, but enforced to be a string resource ID
of the defined resource type. This would also minimize the changes needed to clients.

The reason this approach is not chosen is due to dependency on this feature development which would delay stacks
extension support. It needs to be decided if it's worth implementing that feature first.

Here is an example of this approach:

```bicep
param aksCluster Resource<'Microsoft.ContainerService/managedClusters@2024-02-01'> // = "/subscriptions/.../resourceGroups/.../Microsoft.ContainerService/managedClusters/...

param namespace string = 'default'

extension kubernetes with {
  kubeConfig: aksCluster.listClusterAdminCredential().kubeconfigs[0].value
  namespace: namespace
}

// ...
```

### Use of existing resources

The extension configuration could be kept as-is but restricted to existing resource reference expressions. There should
not be a need to include this resource in the deployment GET response for stacks usage because it will be compiled to
the new configuration format and embedded in that.

Here is an example:

```bicep
param clusterName string
param namespace string = 'default'

// duplicates a resource definition, but leaves the extension config definition site unchanged
resource aksCluster 'Microsoft.ContainerService/managedClusters@2024-02-01' existing = {
  name: clusterName
}

extension kubernetes with {
  kubeConfig: aksCluster.listClusterAdminCredential().kubeconfigs[0].value
  namespace: namespace
}

// ... 
```

## Rollout plan

The milestone for the initial rollout will be the successful deletion of extension resources through stacks. Supporting
features for extensibility are considered experimental so this specific feature will be considered experimental.

Here are factors that need to be considered for the rollout:

- AKS/Kubernetes in Azure is only available in production environments and this is the only available V2 extension at
  this time that can be tested with.
- New prod API versions require going through swagger review which can take a lot of time.
- SDK updates for the deployment service should be consumed by Stacks before releasing the Stacks Stable API Version.

The rollout:

1. Changes in both the Deployments service and Stacks service will be feature gated.
    - Feature gate needs to be at the subscription level to enable Canary-scoped development and testing.
    - The feature gate should apply on whatever the current stable (or preview) API version is for each service
      (or all API versions if it easier).
1. Development iteration of the Deployment service and Stack service will take place and be tested in Canary until the
   milestone is hit.
1. Develop some synthetics to run in production that use the kubernetes extension in a Deployment stack.
1. Once the milestone implementation is sufficiently bug bashed and signed off by the team, the feature gate will be
   disabled.
1. The Swagger update process will begin for at least a new Prod Preview API version.
    - Priority should be given to the deployments side due to it being a dependency.
    - Both swagger updates could be potentially be done in parallel.
    - It may be possible to start the Deployment service swagger early if there's confidence in its changes.
1. Incorporate any changes from swagger review and retest if needed using the feature flags.
1. Merge the Deployment service swagger changes and release SDK.
1. Update stacks code to consume new deployment SDK code.
1. Merge the Deployment stack swagger changes and release SDK.
1. Update client tools such as Azure CLI and Azure PowerShell to handle the new API responses.
1. Collaborate with the Portal team to update the deployments blade, ensuring that the UI reflects the API changes.

## Unresolved questions

### Expression differentiation for persisting the value versus persisting the directive/reference

This was highlighted in a footnote of the allowable expressions inside of extension configurations. Should Bicep offer a
way to differentiate the workflow of resolving a reference and persisting the resolved value versus the workflow of
persisting the reference and using it to re-fetch later? For a non-stack Deployment, there is no semantic difference
between the two, but there is a distinction to be made for stacks. This would also tie into Bicep lint diagnostics. See
the below table for examples.

| Stack behavior after deployment GET         | Bicep Expression                       |
|---------------------------------------------|----------------------------------------|
| Persisting the resolved value               | `namespace: kv.getSecret('xyz')`       |
| Persisting the reference and re-fetch later | `namespace: () => kv.getSecret('xyz')` |

### [Resolved] Key vault secret expressions in extension configurations and other expression limitations driven by stacks

Whether key vault secret access should be supported in extension configurations is a tricky subject. Users will likely
want this feature, and it would work fine with a standalone deployment that isn't tethered to a stack because the secret
is only needed at the time of deployment. The problem from the stacks perspective with introducing this feature is that
accessing a user's key vault in the future for the deletion of stack resources introduces a failure point. What if the
user did not properly maintain this secret and it's now invalid? This type of issue would block the deletion or update
of a stack when resources that no longer managed are to be deleted and potentially result in unexpected stack states.
The errors would likely manifest as incidents for the stacks team to investigate.

✅ Decision: Allow key vault sources and other ARM resource sources. Provide error detail to user if secret is invalid.

### [Resolved] Breaking up extension configurations into secret and non-secret components

This question was arrived at in the drawbacks section. The extension would need to provide metadata about top-level
configuration properties to enable the various layers to differentiate secret versus non-secret properties. It would
enable use of deployment parameters within non-secret portions of the extension configuration and stacks could persist
these values.

✅ The configuration schema will need to outline this to enable validation at both the Bicep and ARM template engine
level to prevent secure leaks and allow use of non-secure deployment parameters. The extension config schema in Bicep
appears hard coded for Kubernetes which can be changed. There does not appear to be special validation of the Kubernetes
plugin configuration at the ARM template engine level which will require some special validation added. Future source of
schemas will be the extension registry?

### [Resolved] Getting the resource dependency graph of a deployment

Deletion of resources may require sequencing to avoid failures in deletion. The stacks service will employ a brute-force
approach to resource deletion, which means it will keep retrying all resources until they succeed within reasonable
limits. If brute-force is not sufficient or there's common cases where ordering is necessary, the design may need
modification to fetch the deployment dependency graph from the Deployments service after stack template deployment has
completed and persist it in the stack for later use. However, stacks service will need some type of brute force
algorithm to account for problems opaque to the service.

It's important to note that deletion dependencies and Azure Deployment dependencies are two different concepts.
Deployment dependencies define create-or-update dependencies. This means that resources can depend on each other simply
by sourcing resource properties from other resource outputs, but this does not imply there is a container relationship
between the resources. Deletion dependencies would be determined by container relationships. A container relationship in
the context of deletion means that if the parent resource is deleted, the child resource is either deleted beforehand or
will be an invalid state. An example of a container relationship is between Azure resource groups and resources.
Deleting a resource group will delete the resources inside of it.

There are two types of container relationships for extension resources:

1. `ARM resource -> Extension resource` (example: AKS cluster for AKS resources)
2. `Extension resource -> Extension resource`

The ARM resource container case may be detectable if the ARM resource is used as input to the extension configuration.
The extension resource container case is harder to detect. In the Kubernetes example, the extension resources reference
each other with selectors which are defined in Bicep as a string field. There is no hard link between the resources at
the template level.

✅ Decision: This is out of scope for now. Stacks will employ a brute force approach and will adapt as needed.

### [Resolved] Limitations imposed on deployments due to stacks

Deployments service does not need to evaluate extension configurations at a later time like stacks service does. This
means standalone deployments could have constraints lifted. Should constraints necessary for stacks be made generally or
only for stack deployments? If only for stacks, how would the API payloads change and how would the end user UX inform
users of stacks limitations?

✅ Answer: Introduce a stacks mode for Bicep files that enables additional validation that is in-sync with backend
validation. Only persist and display extension configurations for stacks deployments that have been successfully
validated.

## Out of scope

- Deployment stack resource locks on extension resources (in ARM, this is deny assignments that prevent deletion or
  update of resources while being managed by the stack).
