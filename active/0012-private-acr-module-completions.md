---
REP Number: 0012
Author: StephenWeatherford (Stephen Weatherford)
Start Date: 2024-05-01
Feature Status: Public Preview
Bicep Issue Number(s): <Fill if the REP originates from one or more Bicep issues; otherwise, delete.>
Enhances: <Fill an existing final REP number if this REP enhances it; otherwise, delete.>
Deprecates: <Fill an existing final REP number if this REP deprecates it; otherwise, delete.>
Enhanced By: <Applicable to existing final REPs only. Fill a final REP number if that REP enhances this one. Delete for new REPs.>
Deprecated By: <Applicable to existing final REPs only. Fill a final REP number if that REP deprecates this one. Delete for new REPs.>
---

<!-- Remove this comment and the prompts (in the form of blockquotes) for each section before submitting your PR -->

# [Breaking Change (delete if not applicable)] Title - replace me with feature name

## Summary

> Provide a brief paragraph explaining the feature.

## Terms and definitions

> Include any terms, definitions, or acronyms that are used in this design document to assist the reader.

## Motivation

> Outline the reasons behind implementing this feature, the use cases it supports, and the expected outcomes.

## Detailed design

> The core of the REP. Elaborate on the design and implementation with sufficient detail for someone familiar with Bicep to grasp. Provide detailed examples to illustrate how the feature is used and its implications on user experience.

### Client side changes

### Server side changes (if applicable)

### Microsoft.Resources/deployments API changes (if applicable)

### Examples

- Example 1
```bicep
```
- Example 2
```bicep
```

## Drawbacks

> Discuss any potential drawbacks associated with this approach. Why might this *not* be a suitable choice? Consider implementation costs, integration with existing and planned features, and the migration cost for Bicep users. Clearly identify if it constitutes a breaking change.

## Alternatives

> Explore alternative designs that were considered and clarify why the chosen design stands out among possible options. Provide rationale and discuss the impact of not selecting other designs.

## Rollout plan

> Describe how you will deliver your feature. Does it need to be an experimental feature? If the feature involves backend changes, dose it need to be guarded by an ARM feature flag?

## Unresolved questions

> - What aspects of the design do you expect to resolve through the REP process before merging?
> - What parts of the design do you anticipate resolving through feature implementation before stabilization?

## Out of scope

> - What related issues are considered out of scope for this REP but could be addressed independently in the future?

**NB**: This section is not intended to capture unfinished elements of a design. For draft proposals, anything that is yet to be decided but must be addressed in order for a feature to be functional or GA should be explicitly called out in the **Design** section (or the **Unresolved questions** section). This section is instead intended to identify related issues or features on which you do not wish to make definitive decisions/foreclose any possibilities. Carefully think through how your design will avoid impacting these out of scope questions -- if your design does impact anything in this section, that is a strong indication that whatever is impacted was not out of scope at all. 




































# Completions for private ACR modules

## Summary

When the user types this:
```bicep
module m 'br/public:
```
they receive automatic completion suggestions based on an index published by the AVM team containing information on all available public AVM modules, their versions, and descriptions.

This feature is to extend these completions to support completions for modules published in private container registries as well as the official public AVM modules.

## Motivation

Improve programmer efficiency and ease of use.

## Design requirements
- Scalable
  - Some registries can have hundreds of thousands of artifacts (how many of them have Bicpep modules?)
- Don't rely on Azure-specific functionality

## Performance characteristics

### Retrieving a list (catalog) of all repositories in a registry is fast

This is generally a single REST call with paging (e.g. 100 repository names returned per call)

Example:
```
https://mcr.microsoft.com/v2/_catalog
```
Result:
```
{
  "repositories": [
    "samples/blockchain-ai/0xdeca10b-demo",
    "samples/blockchain-ai/0xdeca10b-simulation",
    "azure-sentinel/solutions/sapcon",
    "acc/samples/helloworld-outproc",
    "acc/samples/attested-tls-inproc",
    "acc/samples/attested-tls-outproc",
    "acc/samples/layered-attested-tls-outproc",
    ...
```

### Determining whether a specific repository contains a bicep module (slow)

Requires individual calls to retrieve the manifests for each discovered repository

### Brute-force indexing

Therefore, retrieving an accurate list of Bicep modules in a registry requires first retrieving its catalog of repositories and then inspecting each repository individually to determine if it contains a Bicep module.  A very slow process.

## High-level options

From a high altitude view, these are the options we have considered:

### Do nothing (support completions only for public AVM modules)

### Use brute force on each client (slow)
  - Generate/cache a list of Bicep modules in a registry on first use
  - Does not scale
  - Adds unnecessary load to the registry
  - Could narrow down search path by specifying a repository prefix or pattern in bicepconfig.json
    - If bicep module repository names are predictable, this could be a reasonable solution

### Use brute force centrally or manually (slow)
1. Allow users to provide Bicep with their own custom-created index
1. Provide a bicep command to create a brute-force index and publish it for all users
  - Bicep might be able to automtically update the index when a module is published, if user has write permissions to the index

- Has concurrency issues
- Requires write permissions to the index
- **Requires read access to the index to generate completions**
  - Is this an issue?
- Is it a problem to use a fixed repository path like "/bicep-index"?
  - Could be configured in bicepconfig.json (required for each client project to get completions)

Could narrow down search path by specifying a repository prefix or pattern in bicepconfig.json

### Create/cache index on each client using only the catalog of repositories (fast)

### "Optimistic" index (only uses catalog)
1. Provide completions for all repositories in a registry (whether they're Bicep modules or not)
  - Each repo will be inspected on demand when user attempts to get completions for a module's versions. Might be removed from future completions if not a Bicep module.
  - Could narrow down search path by specifying a repository prefix or pattern in bicepconfig.json
    - If bicep module repository names are predictable, this could be an effective solution

### Use repository name to find Bicep modules (only uses catalog)
These solutions involve making it possible to accurately find all Bicep modules just from the catalog of repository names.
These scale very well to creating/caching an index on each client.

#### 1. Munge the module repository name
  * If user publishes a module to myreg.io/mypath/module1, instead of creating a repository named `myreg.io/mypath/module1`, it creates a repository named `myreg.io/mypath/module1.bicep`.

|Path|Artifact Content|
|---------------------------------------------|---|
| myreg.io/mypath/module1.bicep | module `myreg.io/mypath/module1.bicep` |

  * Support older modules by looking for `myreg.io/mypath/module1.bicep` if `myreg.io/mypath/module1.bicep` not found
##### Cons
- Users will see `myreg.io/mypath/module1.bicep` different artifact name in Azure portal - any issues with this
- Ambiguity as to whether `myreg.io/mypath/module1.bicep` or `myreg.io/mypath/module1` gets loaded if both exist.
- Breaks forwards compatibility - old Bicep versions won't see the new modules without updating

#### 2. **Create a secondary semaphore repository with each Bicep module
  * When you publish module `myreg.io/mypath/module1`, we actually publish two artifacts:

|Path|Artifact Content|
|---------------------------------------------|---|
|myreg.io/mypath/module1 | actual artifact |
|myreg.io/mypath/module1.bicep | empty artifact signaling that `myreg.io/path/module1` is a bicep module |
|myreg.io/mypath/module2 | actual artifact |
|myreg.io/mypath/module2.bicep | empty artifact signaling that `myreg.io/path/module1` is a bicep module |
       
  * To find all Bicep modules, we look for repositories with names `<myrepo>.bicep`, and if there's also a repo named `<myrepo>`, we know it's a Bicep module
##### Cons
- Users will see both `myreg.io/mypath/module1` and `myreg.io/mypath/module1.bicep` in the registry
- We cannot automatically clean up `myreg.io/mypath/module1.bicep` if `myreg.io/mypath/module1` is deleted or vice versa. But they will be listed right next to each other, so should be easy to find.
  - Referrers only work within the same repo name.
 
#### 3. Same as above, but store the second artifact in a known separate location, such as "myreg.io/bicep-index/mypath/module1"

|Path|Artifact Content|
|-------------------------------------------------|---|
| myreg.io/mypath/module1 | actual artifact |
| myreg.io/mypath/module2 | actual artifact |
| myreg.io/**bicep-index**/mypath/module1.$bicep | empty repo signalling that `myreg.io/path/module1` is a bicep module |
| myreg.io/**bicep-index**/mypath/module2.$bicep | empty repo signalling that `myreg.io/path/module1` is a bicep module |       

##### Cons
- Will require write permissions to the index location when publishing
- Requires read access to the index location to generate completions
