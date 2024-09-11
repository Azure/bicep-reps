---
REP Number: <Fill me in with a four-digit number matching the pull request number; Update AFTER PR is approved and BEFORE is merged.>
Author: levimatheri (Levi Muriuki)
Start Date: 2024-09-09
Feature Status: Private Preview
Bicep Issue Number(s): [#645](https://github.com/Azure/bicep/issues/645), [#4959](https://github.com/Azure/bicep/issues/4959), [#9969](https://github.com/Azure/bicep/discussions/9969)
---

# Title - New deployment principal function

## Summary

Introduces a new function to get the principal that is submitting the current deployment. This can be a user or an application (service principal)

## Terms and definitions

## Motivation

Customers have expressed that having a function that can output properties about the current deployment principal would be helpful for [use-cases](#use-cases-expressed-by-customers) that require certain fields such as `objectId` (`principalId`). The current workarounds to achieve this are not desirable as these workarounds include running a prerequisite `az` command step to obtain the current principal's fields and then passing them into the Bicep template as parameters, e.g.

```sh
az login
let principalId = az rest --method GET --uri https://graph.microsoft.com/v1.0/me | from json | get id
az deployment group create -g mygroup -f my.bicep -p $"principalId=($principalId)"
```

```bicep
param principalId string
...

resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  ...
  properties: {
    ...
    principalId: principalId
    principalType: 'User'
  }
}
```

### Use-cases expressed by customers
- Assigning the current principal to RBAC roles to various resources within the same template
    - [API definition](https://learn.microsoft.com/en-us/azure/templates/microsoft.authorization/roleassignments?pivots=deployment-language-bicep)
- Adding current user to a Entra ID group created using Bicep extensibility 
    - requires member's `objectId`
- Setting the default admin of a SQL server
    - [API definition](https://learn.microsoft.com/en-us/azure/templates/microsoft.sql/servers/administrators?pivots=deployment-language-bicep)
- Assigning Key Vault access policies
    - [API definition](https://learn.microsoft.com/en-us/azure/templates/microsoft.keyvault/vaults/accesspolicies?pivots=deployment-language-bicep#accesspolicyentry)

## Detailed design
- We plan to expose a minimal set of properties to satisfy the above use-cases, namely:
    - `objectId` - represents the unique identifier of the principal within a tenant
    - `type` - i.e. `User`, `ServicePrincipal`
    - `tenantId` - represents the tenant where the user/service principal is managed
- By scoping it down to these properties, this should make the implementation easier, as we can access these values via headers, and not have to make outgoing requests to MSGraph.

### Client side changes
We will expose a new function in the Bicep within the `az` namespace.
### Server side changes (if applicable)
We will implement the new function as part of the Deployment Engine on the backend. 
Notably, there already exists a [DeploymentUserIdentifier](https://msazure.visualstudio.com/One/_git/AzureUX-Deployments?path=/src/Engine/Host/Azure/AzureDeploymentEngine.cs&version=GBmaster&line=671&lineEnd=672&lineStartColumn=1&lineEndColumn=1&lineStyle=plain&_a=contents), which is currently used for the secure outputs feature to check access permissions. We should reuse this class if possible, as it seems to represent what we need here. In that case, a refactor of the class and its usages may be needed to accommodate the new function.

### Examples

- Assigning RBAC role
```bicep
resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  ...
  properties: {
    ...
    principalId: deployer().objectId
    principalType: deployer().type
  }
}
```

- Assigning Key Vault access policy
```bicep
resource keyvault 'Microsoft.KeyVault/vaults@2019-09-01' = {
  properties: {
    tenantId: deployer().tenantId
    accessPolicies: [
      {
        objectId: deployer().objectId
        tenantId: deployer().tenantId
      }
    ]
  }
}
```

## Drawbacks
The minimal set of properties
## Alternatives

## Rollout plan

We will first roll out the backend changes to public & national clouds, and then thereafter we will add the function to the `az` namespace in Bicep

## Unresolved questions
### [Resolved] Function name

I chose `deployer()` because I think it's self-exlanatory (as a user I can easily tell that it means someone or something that deploys), but there are a few other good options discussed for naming the functions:
1. `submitter()`
   - this works, but I think we can achieve more clarity about what we are _submitting_ exactly with a better name
1. `principal()`
    - this was most upvoted on an earlier discussion, however there are some lingering concerns with vagueness with this one, i.e. is this the deployment principal or some other principal?
1. `deploymentPrincipal()`
   - this works too. It's just slightly longer than `deployer()` which seems to achieve the same meaning
1. `whoami()`
   - This works too; I can't think of any downside to using this?

✅ Decision: Use `deployer()`

### [Resolved] Do we need tenantId property?
It is not clear which tenantId we would need (x-ms-home-tenant-id vs x-ms-client-tenant-id. These are documented [here](https://github.com/Azure/azure-resource-manager-rpc/blob/master/v1.0/common-api-details.md#proxy-request-header-modifications)). In most cases these would be the same, however they would be different in cross-tenant scenarios e.g. Azure Lighthouse. In the cases where the tenantId property is required, e.g. Key Vault access policy, it makes sense for it to be the _resource's_ tenant (x-ms-home-tenant-id), in which case `subscription().tenantId` would suffice. It's not clear when a _user's_ tenant (x-ms-client-tenant-id) specifically would need to be specified if the user is working in a cross-tenant scenario.  We had discussed omitting tenantId altogether to avoid confusion.

✅ Decision: Yes, use the user's home tenant. The fact that the user's & resource's tenant could be the same, does not negate the fact that the user's tenantId is an independent attribute that identifies that user. Therefore, it still makes sense to expose the `tenantId` property to indicate the user's tenant. Point of correction: The Key Vault tenantId allows one to specify a different tenantId than the resource subscription's tenant, so technically it is allowable to supply the user's tenantId. See [this article](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/service/key-vault) for more info about multitenancy in Key Vault.

### [Resolved] Do we need appId property?
On one hand, it probably doesn't hurt to include it, but on the other hand, it's only needed in the [Application-and-user](https://learn.microsoft.com/en-us/azure/key-vault/general/security-features#key-vault-authentication-options) auth option for KV, and I'm struggling to see how relevant the function is to this scenario, since it will return either a user or an application's properties (not both), for example:

```bicep
resource keyvault 'Microsoft.KeyVault/vaults@2019-09-01' = {
  properties: {
  ...
    accessPolicies: [
      {
        objectId: deployer().objectId // this needs to be the user's objectId
        applicationId: deployer().appId // this needs to be the 3P application ID
      }
    ]
  }
}
```

✅ Decision: Omit the `appId` property for now. The Application-and-user auth option scenario seems like a niche one that has not been asked for by customers, and doesn't really fit usage of the new function. The `appId` can easily be added in the future if the need arises.

### How do we expose the principal type?
Within ARM, there exists [functionality](https://msazure.visualstudio.com/One/_git/AzureUX-ARM?path=/src/frontdoor/Roles/Frontdoor.Common/Extensions/RequestIdentityExtensions.cs&version=GBmaster&line=452&lineEnd=453&lineStartColumn=1&lineEndColumn=1&lineStyle=plain&_a=contents) to retrieve the principal type. This relies on JWT token claims from the user. However, from DRP's perspective, these claims are not available within the JWT token that we get from ARM. 
A potential solution pulled from an [internal stack overflow answer](https://stackoverflow.microsoft.com/a/249012/172688) is to rely on [`x-ms-client-principal-id` header](https://github.com/cloud-and-ai-microsoft/resource-provider-contract/blob/master/v1.0/common-api-details.md#proxy-request-header-modifications); this header is not set if the client is a Service principal. However, the documentation mentions it's only added _when available_, and it is sourced from legacy PUID, that makes it sound like it wouldn't be a reliable header to use for this.
An alternative could be to leverage the OBO token from ARM, provided we have visibility into the original user's claims in the token. 

## Out of scope

### MSGraph Principal lookup
Given the [use cases](#use-cases-expressed-by-customers) highlighted above, we are not going to make calls to MS Graph. This means other use-cases such as looking up a user's email will not be exposed through this function.

