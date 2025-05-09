﻿---
REP Number: 0006
Author: kalbert312 (Kyle Albert)
Start Date: 2024-08-27
Feature Status: Public Preview
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
to constrain extension configurations to use constants or simple references the stacks service can act upon.

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

// A sample key vault
resource kv '...' = { } 

// The ARM aksCluster resource can supply kubeConfigs for the AKS Kubernetes use case.
resource aksCluster '...' = {
  // ...
}

module extResources 'extResources.bicep' = {
  name: 'extResourcesDeploymentName'
  extensionConfigs: {
    k8s: {
      namespace: namespace  // sourced from a deployment parameter
      
      // BEGIN clusterType = 'Managed'
      
      clusterType: 'Managed'
      managedClusterId: aksCluster.id
      credentialType: 'Admin'  // 'Admin' | 'User'
      
      // END clusterType = 'Managed
      
      // BEGIN clusterType = 'Custom' w/ key vault
      
      clusterType: 'Custom'
      kubeConfig: kv.getSecret(...)  // Only allowable expression type for secure properties for stack deployments. Other ARM resource sourced credentials must be supported by the extension via config properties.
      
      // END clusterType = 'Custom' w/ key vault
      
      // BEGIN clusterType = 'Custom' w/ custom expression (NOT allowed for stack deployments)
      
      clusterType: 'Custom'
      kubeConfig: aksCluster.listAdminClusterCredential().kubeconfigs[0].value
      // OR
      kubeConfig: someDeploymentParam ? ... : ...
      
      // END clusterType = 'Custom' w/ custom expression (NOT allowed for stack deployments)
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
provided, it will be the extension name. The values of the object are extension configurations and the schema of the
object is determined by the extension. Optionality of configuration is determined by the extension. The example shown
above is how configurations are represented in Bicep and this representation will be transformed into the actual
extension configuration in the Deployments backend. There is a difference because to support stack deployments,
instructions on how to fetch secure portions of the configuration must be persisted.

As for the extension configuration declaration site change, if this is compared to a typical resource group deployment,
it is aligned because the resource group being deployed to is not defined within the deployment template itself, but as a
parameter to the deployment, either through the CLI parameters or as a scope for nested deployments. In this example,
the deployment scope(s) for extensible resources are the extension configurations.

The non-secure values of the configuration are supplied with Bicep expressions. Secure parameters cannot be used for their
value. 

The secure values of the configuration ("kubeConfig" in the example) can be supplied with custom Bicep expressions
for non-stack deployments. For stack deployments, the only allowable expression type is a key vault reference. Secure 
property values can be resolved from ARM resources for stack deployments but the extension must support this declaratively
with its extension properties. The extension can supply the deployment engine detail on how to resolve the value based
on the config values or the deployment engine can hardcode this for specific use cases. A future REP will cover how 
extensions will provide instructions to fetch from an ARM resource. In short, this will be a pure function on the extension
based on what the user passed to the config. Allowing state (such as a resource's state or deploying user) to alter the
directions the extension provides would increase the complexity of the workflow.

Legend:
- ✅: Allowed
- ❌: Not allowed (error)

| Source    | Description                                                                                          | Stack deployments                             |
|-----------|------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| Key Vault | The secret is retrieved from an Azure Key Vault via `kv.getSecret(...)`                              | ✅                                             |
| Extension | The secret is retrieved from an Azure resource through its REST API or through some external source. | ✅ the extension must support it declaratively |
| Custom    | The secret value is sourced directly from the custom Bicep expression provided.                      | ❌                                             |

**For stack deployments, any language expressions present in extension configurations must be constrained to prevent
persisting user secrets long-term and causing security risks. There are no restrictions for non-stack deployments.**
Secure properties will be re-fetched on demand for stack deployments. Non-secure property expressions will be resolved
at deployment time and the value of the expression will be persisted long-term for stack deployments.

For stack deployments, Bicep expressions prohibited anywhere inside extension configurations:
- Secure deployment parameters

For stack deployments, additional prohibited expressions for non-secure properties:
- Value access to inherited extension config secure properties

The reasoning behind the disallowed expressions for stack deployments is because the extension configuration properties
will be evaluated by the deployment engine and the evaluation results will be returned in the deployment GET response.
The stacks service will persist this data long-term and use it for the next stack update or deletion that requires
extension resource deletions. **The deployments backend will need to ensure future sources of certain secure data are 
blocked from being used for stack deployment extension configurations.**

The stacks service already sends a stack resource ID HTTP request header to the deployments service. This will be used
to enable additional validation in the Deployment engine to enforce these expression rules. It will also enable the
return of extension configurations in the Deployment GET response for API versions that support it. This header is only
needed for the initial deployment PUT, as its value will be saved on the deployment metadata and this will be used to
drive the API behavior. For deployments that are not initiated by the stack service, the configuration portion of the
extension details will not be returned, as no additional validation on language expressions is done, which means we
cannot guarantee compliance to the ARM RP RPC rule that a resource GET cannot return secret data.

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

// ✅ whole inheritance
module foo 'foo.bicep' = {
  extensionConfigs: {
    kubernetes: kubernetes // resuse its config, must be from an extension with the same name and version
  }
}

// ✅ piecemeal inheritance
module baz 'foo.bicep' = {
  extensionConfigs: {
    kubernetes: { 
      kubeConfig: kubernetes.kubeConfig // types must match
      namespace: kubernetes.namespace // types must match
    }
  }
}

// ❌ spread inheritance CANNOT be used
module bar 'foo.bicep' = {
  extensionConfigs: {
    kubernetes: {
      ...kubernetes
      namespace: 'myOtherNamespace'
    }
  }
}

// ❌ secure properties cannot be used as values to non-secure properties
module fooy 'foo.bicep' = {
  extensionConfigs: {
    kubernetes: { 
      kubeConfig: kubernetes.kubeConfig // ok
      namespace: kubernetes.kubeConfig // error
    }
  }
}
```

##### Limitations

Because secure property values must be provided in reference format, there are several limitations on allowable Bicep
expressions inside of extension configurations for any deployment:

1. No object spreads within an expression scope that is not represented as a language expression in the ARM template.
  This includes the top-level extension configuration object and secure property reference objects.
1. Extension configuration inheritance is limited to whole inheritance or explicit piecemeal inheritance (each property
  must be explicitly written out to inherit from some value).

#### Providing extension configurations to root deployments

Because extension configurations will be an input property to the deployment, there needs to be a way for frontends
to supply this data to a root deployment. The `bicepparam` file type can be expanded to support defining extension
configurations. Each extension can be configured with a `extension <alias> with { <config object> }` declaration.

Here is an example:

main.bicepparam

```bicep
using 'main.bicep'

param boolParam = true
// ...

extension k8s with {
  namespace: 'default'
  
  // BEGIN clusterType = 'Managed'
  
  clusterType: 'Managed'
  managedClusterId: '/subscriptions/.../resourceGroups/.../Microsoft.ContainerService/managedClusters/...'
  credentialType: 'Admin' // 'Admin' | 'User'
  
  // END clusterType = 'Managed'
  
  // BEGIN clusterType = 'Custom'
  
  clusterType: 'Custom'
  kubeConfig: getSecret(...) // This is the ONLY supported expression type for secure properties for stack deployments. Resource sourced credentials that the extension helps point to must be supported by the extension via a config discriminator such as 'clusterType' in this example.
  
  // END clusterType = 'Custom'
  
  // BEGIN clusterType = 'Custom' & non-stack deployment
  
  clusterType: 'Custom'
  kubeConfig: loadFileContent(...)
  // OR
  kubeConfig: 'inlined'
  
  // END clusterType = 'Custom' & non-stack deployment
}

extension otherExt with { ... }
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
      "namespace": {
        "value": "default"
      },
      "clusterType": {
        "value": "Managed", // "Managed" | "Custom"
      },
      "managedClusterId": {
        "value": "/...", // ARM resource id
      },
      "credentialType": {
        "value": "Admin", // "Admin" | "User"
      },
      "kubeConfig": {
        "keyVaultReference": {
          "keyVault": {
            "id": "/subscriptions/.../resourceGroups/.../Microsoft.KeyVault/vaults/.."
          },
          "secretName": "myKubeConfig"
        }
      },
      // OR
      "kubeConfig": { // ❌ NOT allowed for stack deployments, ✅ non-stack deployment
        "value": "Some literal value"
      }
    }
  }
}
```

The above format defines the signature of the `"extensionConfigs"` object accepted by the deployment PUT API. The
`"extensionConfigs"` object is a mapping of extension aliases (as defined in the deployment template) to extension
configurations. Each extension configuration is an object. The properties of the object are defined by the extension
schema. For example, the Kubernetes extension defines `clusterType`, `kubeConfig`, and `namespace`. Extensions can
define their own config object discriminators for different ways of resolving config data or instructing the deployments
engine on how to fetch the data. Non-secure properties values are provided as an object with a `"value"` property for
the actual value. Secure properties are provided either as an object with either a value or key vault reference.

#### Nested deployment extension configurations

The format of the `"extensionConfigs"` deployment property is different for nested deployments because they are part of
a deployment template. The structure is similar to the parameters file format except that non-key vault based values are
provided either literally or as a language expression directly to the property. This format will be transformed on the
backend to match the PUT format. The reason this is necessary is support language expressions that can resolve to key
vault references.

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
            // Expressions are not allowed to reference secure parameters.
            // This will be evaluated out to the deployment PUT format during preprocessing and deployment
            "k8s": {
              "namespace": {
                "value": "[parameters('namespace')]"
              },
              "clusterType": {
                "value": "Managed"
              },
              "managedClusterId": {
                "value": "[resourceId('Microsoft.ContainerService/managedClusters', parameters('clusterName')),'2024-02-01')]"
              },
              "credentialType": {
                "value": "Admin"
              },
              // key vault example
              "kubeConfig": {
                "keyVaultReference": {
                  "keyVault": {
                    "id": "[...]"
                  },
                  "secretName": "[...]"
                }
              },
              // custom example for secure property (NOT allowed for stack deployments)
              "kubeConfig": {
                "value": "[...]"
              }
            }
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
            "k8s": {
              "namespace": {
                "value": "[parameters('namespace')]"
              },
              "clusterType": {
                "value": "Managed"
              },
              "managedClusterId": { 
                "value": "[resourceId('Microsoft.ContainerService/managedClusters', parameters('clusterName')), '2024-02-01')]"
              },
              "credentialType": {
                "value": "Admin"
              },
              // key vault example
              "kubeConfig": {
                "keyVaultReference": {
                  "keyVault": {
                    "id": "[...]"
                  },
                  "secretName": "[...]"
                }
              }
            }
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
                    // whole inheritance:
                    "kubernetes": "k8s", // ✅ extension name + version must match.
                    //  OR:
                    "kubernetes": "[createObject(...)]", // ❌ language expression not allowed here.
                    // piece-meal inheritance:
                    "kubernetes": {
                      "namespace": "[extensionConfigs('k8s').namespace]", // ✅ ( ❌ [extensionConfigs('k8s').kubeConfig] )
                      "kubeConfig": "[extensionConfigs('k8s').kubeConfig]" // ✅
                    },
                    // partial inheritance:
                    "kubernetes": {
                      "namespace": "[createObject('value', 'myOtherNamespace')]",
                      "kubeConfig": "[extensionConfigs('k8s').kubeConfig]", // ✅ inherit a KV reference for stack deployment. For stack deployments, the language expression must resolve from an extension config secure property, otherwise it's invalid
                       // OR:
                      "kubeConfig": "[createObject('value', listClusterAdminCredential(...).kubeconfigs[0].value)]" // ❌ NOT allowed for stack deployments
                    },
                    //  OR:
                    "kubernetes": {
                      "namespace": "[extensionConfigs('k8s').namespace]", // ✅
                      "kubeConfig": {
                        "keyVaultReference": { // ✅ different key vault reference
                          "keyVault": {
                            "id": "[...]"
                          },
                          "secretName": "[parameters('myOtherSecretName')]"
                        }
                      },
                    }
                  },
                  "template": {
                    "extensions": {
                      "kubernetes": {
                        // assume same extension name and version as "k8s"
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

The language expression rules listed in the Bicep design discussion apply here as well, such as prohibition of secure
deployment parameters.

The `extensionConfigs()` function will return the deployment engine's representation of the configuration. That means
for each top-level property, an object wrapper is returned rather than the actual value of that property. This wrapper
object has the following properties:

- `value`: The resolved value of the extension config property.
- `keyVaultReference`: The key vault reference used to resolve the extension config property if one was used, otherwise null.

For example, to output the values of extension configurations using deployment outputs:

```json5
{
  "outputs": {
    "kubeConfigUsed": {
      "type": "securestring",
      "value": "[extensionConfigs('k8s').kubeConfig.value]"
    },
    "kubeConfigKeyVaultReference": {
      "type": "object",
      "value": "[extensionConfigs('k8s').kubeConfig.keyVaultReference]"
    }
  }
}
```

This object wrapper will be automatically created in Bicep's codegen as needed to match the expected nested deployment
extension config format. When manually authoring an ARM template, the author must match the expected format as shown in
the above examples.

Here are some examples of Bicep to ARM template translations:

Bicep:
```bicep
extensionConfigs: {
  k8s: {
    str: strParam
    obj: objParam
    secureStr: secureStrParam
    secureStrExt: k8s.kubeConfig
    secureObjExt: k8s.kubeObject
    ifBothExt: boolParam ? k8s.kubeConfig : k8s2.kubeConfig
    ifMixed: boolParam ? k8s.kubeConfig : strParam
  }
}
```

ARM template:
```json5
{
  "extensionConfigs": {
    "k8s": {
      "str": "[createObject('value', parameters('strParam'))]",
      "obj": "[createObject('value', parameters('objParam'))]",
      "secureStr": "[createObject('value, parameters('secureStrParam'))]",
      "secureStrExt": "[extensionConfigs('k8s').kubeConfig]",
      "secureObjExt": "[extensionConfigs('k8s').kubeObject]",
      "ifBothExt": "[if(parameters('boolParam'), extensionConfigs('k8s').kubeConfig, extensionConfigs('k8s2').kubeConfig)]",
      "ifMixed": "[if(parameters('boolParam'), extensionConfigs('k8s').kubeConfig, createObject('value', parameters('strParam')))]"
    }
  }
}
```

### Microsoft.Resources/deployments API changes

To support stacks extensibility, the deployments API needs to return additional data about extensible resources.
It needs to return extensions with their configurations and the resources deployed to those extensions. Extensible resources
need to be linked to extensions. This can be done with the key: extension name, extension version, and config ID. It
is possible the key may match to multiple extensions with differing config instructions, but this is an edge case.

Deployment GET:
```json5
{
  // ... deployment contract
  "extensions": [ // NEW: "Extensions" must be returned for any deployment terminal state (Succeeded, Failed, etc)
    {
      "name": "Kubernetes",
      "version": "1.0.0",
      "configId": "<ID from extension>", // Provided by the extension in the CREATE/UPDATE return payload.
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
  ],
  "outputResources": [ // "outputResources" is only populated when the deployment is successful. Stacks uses operations to peel through failed deployment resources.
    { // NEW: Extensible resource
      "extension": {
        "name": "Kubernetes",
        "version": "1.0.0",
        "configId": "<ID from extension>",
      },
      "resourceType": "core/Service",
      "apiVersion": "v1",
      "identifiers": {
        "metadata": {
          "namespace": "default",
          "name": "myService",
        }
      }
    },
    { // ARM resource
      "id": "..."
    }
  ]
}
```

Operation GET for extensible resource:
```diff
{
  ...
  "properties": {
    ...
    "provisioningOperation": "Create",
    "provisioningState": "Succeeded",
    "targetResource": {
+     "resourceType": "core/Service",
+     "apiVersion": "v1",
+     "symbolicName": "myService",
+     "identifiers": {
+       "metadata": {
+         "namespace": "default",
+         "name": "myService",
+       },
+     },
+     "extension": {
+       "alias": "k8s",
+       "name": "Kubernetes",
+       "version": "1.27.8",
+       "configId": "<ID from extension>"
+     }
    }
  }
}
```

Operation GET for ARM resource:

```diff
{
  ...
  "properties": {
    ...
    "provisioningOperation": "Create",
    "provisioningState": "Succeeded",
    "targetResource": {
      "id": "/subscriptions/...",
+     "resourceType": "Microsoft.ContainerService/cluster",
+     "apiVersion": "2024-04-01"
    }
  }
}
```

With the deployments service returning a list of all extensions (across all nested deployments) with their configuration
s and returning all extension resources deployed in the template (across all nested deployments), the stacks service can
process each logical extension resource and track them between 2 stack deployments.

#### Stacks service considerations

Stacks will track a logical resource by its extension name, extension version, and config ID. The extension alias and
deployment ID cannot be used in identifying resources or extensions. Non-concurrent deployments with colliding IDs and
aliases can point to different extensions. The same resource can also be deployed in deployments with differing IDs.
Additionally, these 2 fields can be changed by the user or are generated between deployment stack PUTs. For each 
extensible resource, stacks will persist the extension name, extension version, configuration ID.

The extension, if it accepts a configuration, will be required to provide back a configuration ID. It is the
responsibility of the extension to secure this ID. The extension should return the same ID for the same
configuration passed to it and should not collide between unique configurations. This allows stacks to uniquely identify
the logical resource without persisting secure data. The extension will accept this ID back when stacks tries to
delete extensible resources. At the time of deletion, the extension will compare the ID it receives with the ID of
the configuration passed to it at that time. If the IDs do not match, the extension will error the deletion.

Stacks will also persist the extensions returned by the deployment GET. Each extensible resource should map back to 1
extension for the most part, but there are edge cases where it can map back to multiple extensions. For example, the
user might declare 2 Kubernetes extensions in their templates, each with a different key vault reference. It's possible
that the 2 different secrets could resolve to the same secret, leading to matching config IDs but differing
instructions. Stacks can resolve this by resolving each configuration. If both configurations match, the request to delete
can be deduped and be queued. If extension configurations mismatch, stacks can throw an error to the user and require them
to manually intervene. Stacks can also try to delete with both configurations. The ID check match implemented in 
extensions should cause all but 1 of the requests to fail.

### Reference evaluation failures

The Deployments service will need to account for failures when executing references. A deployment error will be returned
if a failure is encountered. Failures can include invalid secrets, insufficient permissions, API errors (transient or fatal), and mismatched
runtime types.

The Stacks service will also need to account for failures. It inherits all of the scenarios the deployments service has.

The errors should be clear to the user that it is something they can resolve, such as a stale user-provided secret, or
if it's something the deployments team needs to look into. Errors should not leave the stack in a state the blocks the
user from recovering.

## Alternatives

### Pure language expression extension configurations

Design iterations and prototypes of the approach where extension configurations are represented entirely as language
expressions proved to be challenging to validate and maintain. The validation logic depended on knowing which property
in the extension configuration was being processed. It offered a lot of flexibility but with it the complexity of
analysis and the question if the flexibility was justified. There are currently only a few extensions, many with no
configuration or very little configuration needed, so it was decided to pivot to the secure property reference object
approach.

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

### [Resolved] Expression differentiation for persisting the value versus persisting the reference/reference

This was highlighted in a footnote of the allowable expressions inside of extension configurations. Should Bicep offer a
way to differentiate the workflow of resolving a reference and persisting the resolved value versus the workflow of
persisting the reference and using it to re-fetch later? For a non-stack Deployment, there is no semantic difference
between the two, but there is a distinction to be made for stacks. This would also tie into Bicep lint diagnostics. See
the below table for examples.

| Stack behavior after deployment GET         | Bicep Expression                       |
|---------------------------------------------|----------------------------------------|
| Persisting the resolved value               | `namespace: kv.getSecret('xyz')`       |
| Persisting the reference and re-fetch later | `namespace: () => kv.getSecret('xyz')` |

✅ Not doing this at the current time.

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
- Azure auth for kube extension.
