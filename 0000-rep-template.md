---
REP Number: <Fill me in with a four-digit number in double quotes (otherwise it's octal) matching the pull request number; Update AFTER PR is approved and BEFORE is merged.>
Author: <github username (First Name and Last Name)>
Start Date: <Today's Date, YYYY-MM-DD>
Feature Status: <Private Preview | Public Preview | Public>
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


