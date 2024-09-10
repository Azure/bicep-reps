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

> Include any terms, definitions, or acronyms that are used in this design document to assist the reader.

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
    - `type` - i.e. `User`, `Service Principal`
    - `tenantId` - represents the tenant where the user/service principal is managed
    - `appId` - represents the application ID (client ID) linked to the authenticated principal, or the application used for delegated authentication
- By scoping it down to these properties, this should make the implementation easier, as we can access these values via headers, and not have to make outgoing requests to MSGraph.

### Client side changes
We will expose a new function in the Bicep within the `az` namespace.
### Server side changes (if applicable)
We will implement the new function as part of the Deployment Engine on the backend.
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

## Alternatives

## Rollout plan

We will first roll out the backend changes to public & national clouds, and then thereafter we will add the function to the `az` namespace in Bicep

## Unresolved questions
### Function name
I chose `deployer()` because I think it's self-exlanatory (as a user I can easily tell that it means someone or something that deploys), but there are a few other good options discussed for naming the functions:
1. `submitter()`
1. `principal()`
    - this was most upvoted on an earlier discussion, however there are some lingering concerns with vagueness with this one, i.e. is this the deployment principal?
1. `deploymentPrincipal()`
1. `whoami()`

### Do we need tenantId property?
It is not clear which tenantId we would need (x-ms-home-tenant-id vs x-ms-client-tenant-id). In most cases these would be the same, however they would be different in cross-tenant scenarios e.g. Azure Lighthouse. In the cases where the tenantId property is required, e.g. Key Vault access policy, it makes sense for it to be the _resource's_ tenant (x-ms-home-tenant-id); it's not clear when a user's _home_ tenant (x-ms-client-tenant-id) specifically would need to be specified if the user is working in a cross-tenant scenario.  We had discussed omitting tenantId altogether to avoid confusion.

## Out of scope

### MSGraph Principal lookup
Given the [use cases](#use-cases-expressed-by-customers) highlighted above, we are not going to make calls to MS Graph. This means other use-cases such as looking up a user's email will not be exposed through this function.

