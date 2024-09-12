---
REP Number: 0008
Author: kalbert312 (Kyle Albert)
Start Date: 2024-01-25
Feature Status: Public Preview
---

# Extensible Resource Deployment Observability Improvements

## Summary

This REP outlines API enhancements for [`Microsoft.Resources/deployments`](https://learn.microsoft.com/en-us/rest/api/resources/deployments/create-or-update?view=rest-resources-2021-04-01&tabs=HTTP) and [`Microsoft.Resources/deployments/operations`](https://learn.microsoft.com/en-us/rest/api/resources/deployment-operations/get-at-scope?view=rest-resources-2021-04-01) to empower users to effectively monitor the progress of extensible resource deployments through Azure CLI, Azure Powershell, and Azure Portal deployments blade, and other relavant Azure Resource Manager interfaces.

## Terms and definitions

### Bicep Extension

An API abstraction for the following operations:

- Creating or updating Azure Resources or extensible resources declared in Bicep files or ARM templates;
- Performing resource-level or extension-level queries that have no side effects through functions in Bicep files or ARM templates, e.g., list secrets of a resource (currently, this is only supported by the az extension).

### Extensible Resource

An Azure data-plane resource or non-Azure resource declared in a Bicep file or an ARM template. Extensible resources are deployed through Bicep extensions.

## Motivation

Users currently encounter a major challenge when deploying extensible resources, as there is inadequate information available when querying the deployments or deployment operation APIs. Key details, such as resource IDs, are not applicable to extensible resources, making it difficult for users to monitor the progress of their deployments. For instance, Azure Portal users attempting to view extensible resource deployment on the deployment details blade will only see an infinite loading animation without any details being displayed. Similarly, Azure PowerShell users tracking deployment progress using the deployment cmdlet and the `-Verbose` switch will receive logs like `VERBOSE: 00:00:00 - Resource '' provisioning status is succeeded`.

To address this issue, the deployments and deployment operations APIs must be updated to include additional information, such as:

- Deployment resource ID
- Bicep extension alias
- Bicep extension name
- Bicep extension version
- Resource symbolic name
- Resource type
- Resource API version
- Resource identifiers

This additional data would enable users to identify a deployed extensible resource and map it to the corresponding resource definition in a Bicep file or an ARM template, providing users with comprehensive insights.

## Detailed design

### API change for `GET /Microsoft.Resources/deployments/{deploymentId}`

#### Adding `extensions`

The `Microsoft.Resources/deployments` API will be updated to include an `extensions` array in the GET response body. This modification is intended to incorporate information about extensions that are involved in executing the parent deployment and nested deployments.

```diff
{
  ...
  "type": "Microsoft.Resources/deployments",
  "properties": {
    ...
+   "extensions": [
+     {
        // deploymentId is used to uniquely identify an extension since a nested deployment could import the same extension with the same alias.
+       "deploymentId": "/subscriptions/00000000-0000-0000-0000-000000000001/resourceGroups/my-resource-group/providers/Microsoft.Resources/deployments/my-deployment",
+       "alias": "az",
+       "name": "AzureResourceManager",
+       "version": "1.0.0"
+     },
+     {
+       "deploymentId": "/subscriptions/00000000-0000-0000-0000-000000000001/resourceGroups/my-resource-group/providers/Microsoft.Resources/deployments/my-deployment",
+       "alias": "k8s",
+       "name": "Kubernetes",
+       "version": "1.27.8",
+       "config": {
+         // Out of the scope of this REP, but it is required to enable Deployment Stacks integration.  
+       }
+     }
    ]
  }
}
```

#### Updating the `outputResources` to include more identifiable information

To improve the current structure where each `outputResources` object only has a resource `id`, it's essential to add more properties for identifying extensible resources, which lack resource IDs. These additions should include the resource symbolic name, resource type, resource API version, resource identifier, the resource ID of the deployment containing the resource, and the Bicep extension alias.

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
+       "deploymentId": "/subscriptions/00000000-0000-0000-0000-000000000001/resourceGroups/my-resource-group/providers/Microsoft.Resources/deployments/my-deployment",
+       "extensionAlias": "k8s",
+       "symbolicName": "myService",
+       "resourceType": "core/Service",
+       "apiVersion": "v1",
+       "identifiers": {
+         "metadata": {
+           "namespace": "default",
+           "name": "myService",
+         },
+         "serverHostHash": "60fd32871cbe255d5793ce9e6ebf628beba25224cde7104e6a302f474f2f656e"
+       }
      }
    ]
  }
}
```

### API changes for `GET /Microsoft.Resources/deployments/operations/{operationId}` and `GET /Microsoft.Resources/deployments/operations`

#### Updating `targetResource` to include extensible resource and extension information

To align with modifications made to the `Microsoft.Resources/deployments` API, the response body now incorporates the addition of `symbolicName`, `resourceType`, `apiVersion`, `identifiers`, and `extension`.

```diff
{
  ...
  "properties": {
    "provisioningOperation": "Create",
    "provisioningState": "Succeeded",
    "targetResource": {
      "resourceType": "core/Service",
+     "apiVersion": "v1",
+     "symbolicName": "myService",
+     "identifiers": {
+       "metadata": {
+         "namespace": "default",
+         "name": "myService",
+       },
+       "serverHostHash": "60fd32871cbe255d5793ce9e6ebf628beba25224cde7104e6a302f474f2f656e"
+     },
+     "extension": {
+       "alias": "k8s",
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

### [Resolved] Provide a hash of extension resource identifiers

The identifiers object of the extension resource object has an unpredictable schema as it's decided by the extension.
This complicates assigning a key to the resource because equality contracts (with hashing) need to be established on 
the key. If a string/hash were to be supplied to the user in the API, it would need be sourced from a serialized JSON string
where the properties are recursively ordered with a stable sort. The JSON object would contain the identifiers object
supplied by the extension and any additional context needed to prevent possibility of collision.

```diff
"outputResources": [
  // Extensible resource
  {
    "deploymentId": "/subscriptions/00000000-0000-0000-0000-000000000001/resourceGroups/my-resource-group/providers/Microsoft.Resources/deployments/my-deployment",
    "extensionAlias": "k8s",
    "symbolicName": "myService",
    "resourceType": "core/Service",
    "apiVersion": "v1",
    "identifiers": {
      "metadata": {
        "namespace": "default",
        "name": "myService",
      },
      "serverHostHash": "60fd32871cbe255d5793ce9e6ebf628beba25224cde7104e6a302f474f2f656e"
    },
+   "identifiersHash": "string/hash that can be used as a key for the extension resource"
  }
```

> ✅ Decision was made to leave it up to the client.

### [Resolved] Consistent schema V.S. Minimum backend changes

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
+       "apiVersion": "2020-01-01",
+       "deploymentId": "/subscriptions/00000000-0000-0000-0000-000000000001/resourceGroups/my-resource-group/providers/Microsoft.Resources/deployments/my-deployment",
+       "extensionAlias": "az",
+       "extension": "AzureResourceManager"
      }
  }
}
```

Implementing this enhancement involves comprehensive modifications across various deployment engine components, including but not limited to `AzureDeploymentEngine`, `ResourceProxyDefinition`, `DeploymentOutputResourceDefinition`, and numerous deployment jobs. Such widespread changes carry a significant risk due to the complexity of the deployment engine and the potential for unforeseen impacts on its functionality, which would require careful consideration and discussion.

> ✅ The decision is to not include Azure resources considering the risks.

### [Resolved] Should the az extension always be returned?

Building on the previous discussion, if the extension details are not included within `outputResources` for Azure resources, should we consistently list the `az` extension in `extensions` for sake of consistentcy and clarity?

> ✅ The az extension should be returned as long as it is present in the ARM template.

### Should `outputResources` include `existing` resources as well?

Currently, `outputResources` only contains created or updated resources. Should we expand the property to include `existing` resources?

> ✅ No, we should not change the semantics of the `outputResources` property. If there is a need for returning referenced resources in the future, a new property like `referencedResources` can be added.

### [Resolved] Better name for `deploymentProviders`

> ℹ️ The `extensions` property added to the deployments API response body was originally named `deploymentProviders` in the initial design.

Can we come up with some better names?

- `importedProviders`
- `infraProviders`
- `managementProviders`
- `apiProviders`
- `platformProviders`
- `platforms`
- `apis`
- `extensions`
- `plugins`
- `vendors`
- `protocols`
- `services`
- `workloads`

> ✅ We decided to use `extensions` (or "Bicep extensions" and "ARM template extensions" for added clarity) as suggested by @anthony-c-martin for these reasons:
> - The term `providers` is considered to be highly ambiguous, particularly within the ARM context where it typically denotes Azure resource providers, which could confuse discussions around Bicep's extensibility features. Although qualifiers can help clarify the context, finding an appropriate and precise qualifier is challenging. Additionally, the introduction of a qualifier results in the formation of a compound noun, which can be cumbersome and less user-friendly.
> - The term `extensions` is deemed to be less prone to misunderstanding. The potential for confusion between Bicep / ARM template deployment extensions and Azure resource providers is minimal. Although the term is also used in the context of Azure extension resources, it is a less common usage. Notably, the Graph team has already adopted the term extensions over providers internally for clearer internal communication.
> - The concept of "extensibility" has been a recurring theme in our public documentation. Adopting the term `extensions` aligns closely with this existing terminology, making it a more intuitive choice.
> - 💪 "Bicep extension" resonates well with our established naming conventions for Bicep-related features.


## Out of scope

### Portal UI design

The update to the Portal deployments blade is necessary to display the symbolic name and Bicep extension information for extensible resources. Since the portal team has ownership of the deployments blade, they hold the responsibility for both the UI design and the implementation of these changes. The addition of new properties — `extensions` in the deployments API and `extensions` in the deployment operations API — provides the Portal team with the essential data to programmatically differentiate between Azure resources and extensible resources. This capability is key to enabling the team to design a UI that effectively segregates and presents Azure resources and extensible resources.

### Including extension config in `GET /Microsoft.Resources/deployments/{deploymentId}` response body

This should be designed separately within the scope of the Deployment Stacks and Bicep extensibility integration project.
