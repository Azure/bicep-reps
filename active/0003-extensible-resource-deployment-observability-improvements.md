---
REP Number: 0003
Author: shenglol (Shenglong Li)
Start Date: 2024-01-25
Feature Status: Public Preview
---

# Extensible Resource Deployment Observability Improvements

## Summary

This REP outlines API enhancements for [`Microsoft.Resources/deployments`](https://learn.microsoft.com/en-us/rest/api/resources/deployments/create-or-update?view=rest-resources-2021-04-01&tabs=HTTP) and [`Microsoft.Resources/deployments/operations`](https://learn.microsoft.com/en-us/rest/api/resources/deployment-operations/get-at-scope?view=rest-resources-2021-04-01) to empower users to effectively monitor the progress of extensible resource deployments through Azure CLI, Azure Powershell, and Azure Portal deployments blade, and other relavant Azure Resource Manager interfaces.

## Terms and definitions

### Deployment Provider

An API abstraction for the following operations:

- Creating or updating Azure Resources or extensible resources declared in Bicep files or an ARM templates;
- Performing resource-level or provider-level queries that have no side effects through provider functions in Bicep files or ARM templates, e.g., list secrets of a resource (currently, this is only supported by the az provider).

### Extensibility Provider

An extensibility provider refers to any deployment provider that operates outside the scope of ARM, such as the MS Graph provider and the Kubernetes provider. An extensibility provider deployes extensible resources instead of Azure resources.

### Extensible Resource

An Azure data-plane resource or non-Azure resource declared in a Bicep file or an ARM template. Extensible resources are deployed through extensibility providers.

## Motivation

Users currently encounter a major challenge when deploying extensible resources, as there is inadequate information available when querying the deployments or deployment operation APIs. Key details, such as resource IDs, are not applicable to extensible resources, making it difficult for users to monitor the progress of their deployments. For instance, Azure Portal users attempting to view extensible resource deployment on the deployment details blade will only see an infinite loading animation without any details being displayed. Similarly, Azure PowerShell users tracking deployment progress using the deployment cmdlet and the `-Verbose` switch will receive logs like `VERBOSE: 00:00:00 - Resource '' provisioning status is succeeded`.

To address this issue, the deployments and deployment operations APIs must be updated to include additional information, such as:

- Resource symbolic name
- Resource type
- Deployment provider alias
- Deployment provider name
- Deployment provider version

This additional data would enable users to identify a deployed extensible resource and map it to the corresponding resource definition in a Bicep file or an ARM template, providing users with comprehensive insights.

## Detailed design

> The core of the REP. Elaborate on the design and implementation with sufficient detail for someone familiar with Bicep to grasp. Provide detailed examples to illustrate how the feature is used and its implications on user experience.

### `Microsoft.Resources/deployments` API changes for PUT and GET

#### Adding `deploymentProviders`

The `Microsoft.Resources/deployments` API will be updated to include a `deploymentProviders` array in both the PUT and GET response bodies. This modification is intended to incorporate information about deployment providers that are involved in executing the deployment to offer a more comprehensive deployment view.

> It's worth noting that we cannot use the name `providers` as the property `providers` is already assigned to return Azure Resource Provider information. While an alternative approach could involve renaming the current `providers` property to `resourceProviders` for enhanced accuracy, the solution should be avoided as it constitutes a breaking change.

```diff
{
  ...
  "type": "Microsoft.Resources/deployments",
  "properties": {
    ...
    "providers": [
      {
        "namespace": "Microsoft.Storage",
        "resourceTypes": [
          {
            "resourceType": "storageAccounts",
            "locations": [
              "eastus"
            ]
          }
        ]
      }
    ],
+   "deploymentProviders": [
+     {
+       "alias": "az",
+       "name": "AzureResourceManager",
+       "version": "1.0.0"
+     }  ,
+     {
+       "alias": "kubernetes",
+       "name": "Kubernetes",
+       "version": "1.27.8"
+     }
    ]
  }
}
```

#### Updating the `outputResources` to include more identifiable information

To improve the current structure where each `outputResources` object only has a resource `id`, it's essential to add more properties for identifying extensible resources, which lack resource IDs. These additions should include the symbolic name, the resource type, and the deployment provider alias.

```diff
{
  ...
  "type": "Microsoft.Resources/deployments",
  "properties": {
    "outputResources": [
      // Azure resource
      {
        "id": "/subscriptions/00000000-0000-0000-0000-000000000001/resourceGroups/my-resource-group/providers/Microsoft.Storage/storageAccounts/my-storage-account"
      },
      // Extensible resource
      {
+       "symbolicName": "myService",
+       "resourceType": "core/Service@v1",
+       "deploymentProvider": "kubernetes"
      }
    ]
  }
}
```

### `Microsoft.Resources/deployments/operations` API changes for GET and LIST

#### Updating `targetResource` to include symbolic name and provider for extensible resources

To align with modifications made to the `Microsoft.Resources/deployments API`, the response body now incorporates the addition of `symbolicName` and `deploymentProvider`.

```diff
{
  ...
  "properties": {
    "provisioningOperation": "Create",
    "provisioningState": "Succeeded",
    "targetResource": {
      "resourceType": "core/Service@v1",
+     "symbolicName": "myService",
+     "deploymentProvider": {
+       "alias": "kubernetes",
+       "name": "Kubernetes",
+       "version": "1.27.8"
+     }
    }
  }
}
```

### Client side changes

### Azure CLI and Azure PowerShell

Updating the Azure CLI and Azure PowerShell to effectively communicate deployment information for extensible resources is crucial for providing users with clear and actionable insights. A practical approach to this could include enhancing the output with informative log messages that clearly summarize essential details. For instance, implementing a log message format such as:

```
VERBOSE: 00:00:00 - Extensible resource with symbolic name 'myService' of type 'core/Service@v1' has provisioning status: 'Succeeded'. The resource is provisioned by 'Kubernetes@1.27.8'.
```

## Drawbacks

### Inconsistency between Azure resources and extensible resources

The new properties are added exclusively for extensible resources, leaving Azure resources unchanged, which avoids introducing breaking changes to the API, SDK, and clients. However, this approach leads to an inconsistency in the schema between Azure resources and extensible resources. Such a disparity could potentially cause confusion, as users might expect a uniform schema across all resource kinds. Clear documentation might be required to manage user expectations and explain the rationale behind the differing schemas.

## Alternatives

### API redesign

As an alternative solution, a redesign focused on using symbolic names instead of resource IDs as the primary identifiers for deployments and deployment operations could be beneficial. This approach would simplify the API structure and make it more intuitive.

While this redesign could enhance API clarity and operational efficiency, it represents a substantial shift, introducing major breaking changes for both the API itself and the back-end implementation. These changes would have a broad impact, affecting everything from internal systems to external clients that depend on the existing API schema. Moreover, pivoting to a design that prioritizes symbolic names hinges on the widespread adoption of symbolic name-enabled ARM templates, a transition that is still in its early stages and may require extensive time and effort to achieve widespread acceptance.

Therefore, despite the potential long-term benefits of a more streamlined and user-friendly API, the immediate costs and resource demands associated with such a substantial overhaul are prohibitive. This is particularly the case in light of the ongoing DRP migration. Consequently, embarking on this redesign at the current stage would be impractical.

## Rollout plan

> According to [Azure REST API version change guide](https://github.com/Azure/azure-rest-api-specs/blob/main/documentation/Breaking%20changes%20guidelines.md#new-property-added-to-response), introducing new properties to an API response is deemed a breaking change, necessitating a new API version since these additions would be overlooked by older SDK clients.

To ensure a secure and controlled deployment of the API modifications, the rollout should follow these steps:

1. Update the backend to accommodate the API changes, ensuring they're initially concealed behind a feature flag to prevent immediate, widespread impact.
2. Roll out the updated API across all ARM regions.
3. Activate the feature flag in Canary regions only to start a controlled testing phase, minimizing risk by limiting exposure.
4. Conduct comprehensive testing in Canary regions to validate the functionality and stability of the API changes.
5. Enable the feature flag in more regions incrementally by following the preview feature rollout SDP to manage risk.
6. Do more testing and wait for a few weeks to ensure that the changes haven't introduced any regressions or unintended side effects.
7. Create a new API version to expose the API changes and remove the feature flag.
9. Update Swagger definition for the deployments and deployment operations APIs.
10. Generate versions of Azure SDKs, including the Python SDK and C# SDK.
11. Update client tools such as Azure CLI and Azure PowerShell to handle the new API responses.
12. Collaborate with the Portal team to update the deployments blade, ensuring that the UI reflects the API changes.

## Unresolved questions

### Consistent schema V.S. Minimum backend changes

To address the inconsistency in output resource schema between Azure resources and extensible resources, as highlighted in the Drawbacks section, a potential solution involves extending the schema to include additional properties for Azure resources as well, making Azure resources and extensible resources uniformly identifiable. For example:

```diff
{
  ...
  "type": "Microsoft.Resources/deployments",
  "properties": {
    "outputResources": [
      // Azure resource
      {
        "id": "/subscriptions/00000000-0000-0000-0000-000000000001/resourceGroups/my-resource-group/providers/Microsoft.Storage/storageAccounts/my-storage-account"
+       "symbolicName": "myStorageAccount",
+       "resourceType": "Microsoft.Storage/storageAccounts",
+       "deploymentProvider": "az"
      },
      // Extensible resource
      {
+       "symbolicName": "myService",
+       "resourceType": "core/Service@v1",
+       "deploymentProvider": "kubernetes"
      }
  }
}
```

Implementing this enhancement involves comprehensive modifications across various deployment engine components, including but not limited to `AzureDeploymentEngine`, `ResourceProxyDefinition`, `DeploymentOutputResourceDefinition`, and numerous deployment jobs. Such widespread changes carry a significant risk due to the complexity of the deployment engine and the potential for unforeseen impacts on its functionality, which would require careful consideration and discussion.

### Should the az provider always be returned?

Building on the previous discussion, if the deploymentProvider details are not included within `outputResources` for Azure resources, should we consistently list the `az` provider in the `deploymentProviders` for sake of consistentcy and clarity?

### Should `outputResources` include `existing` resources as well?

Currently, `outputResources` only contains created or updated resources. Should we expand the property to include `existing` resources?

## Out of scope

### Portal UI design

The update to the Portal deployments blade is necessary to display the symbolic name and deployment provider information for extensible resources. Since the portal team has ownership of the deployments blade, they hold the responsibility for both the UI design and the implementation of these changes. 

