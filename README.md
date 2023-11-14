# Bicep REPs

A Bicep REP (Request for Enhancement Proposal) is a formal way to suggest significant changes to Bicep. It offers the Bicep team and the community an organized platform to share feedback, ensuring consensus on the project's direction.

## When to Follow the Process

Consider employing this process if you intend to make substantial changes to the Bicep programming language that would affect user experience. Example where the use of a REP is warranted include:
- *Introduction of New Features*: If the proposed change involves the addition of a new syntax kind or a new ARM template function, particularly if it necessitates the use of a feature flag.
- *Breaking Changes to Non-Experimental Features*: Any alterations that may potentially disrupt existing functionality of non-experimental features.
- *Feature Deprecation*: When contemplating the replacement of a feature that has already been shipped with a new feature.

It's important to note that numerous other changes, such as bug fixes and minor feature enhancements, can be implemented through the regular issue triage and pull request review process within the primary Bicep repository.

## Who Should Create Bicep REPs

While members of the Bicep community are allowed to contribute by authoring new REPs and providing feedback, it's important to note that, in practice, Bicep REPs are typically submitted by the Bicep core maintainers. This submission follows an exclusive design and discussion process among the maintainers. The author of a Bicep REP is not only responsible for the initial design but also for implementing the proposed feature and overseeing the REP's lifecycle.

Therefore, it is generally not recommended for community members to independently initiate the creation of REPs. Instead, we encourage active collaboration and engagement within the main Bicep repository. This involves participating in issue discussions and upvoting feature requests, which can prompt the Bicep core maintainers to initiate the creation of Bicep REPs.

## Bicep REPs Process

Bicep REPs progress through the following stages:

### Draft

This is the initial stage of a Bicep REP, signaling it as a work-in-progress. To create a draft REP, follow these steps:
- Copy `0000-rep-template.md` to `active/0000-my-proposed-feature` (replace `my-proposed-feature` with the actual feature name). Avoid assigning an REP number at this point.
- Complete the REP template.
    - Optionally, If the REP is an enhancement of an existing REP that does not introduce any breaking changes, the author should fill in the `Enhances` field to reference the REP that is being improved.
    - Optionally, if the REP deprecates an existing REP, the author should fill in the `Deprecates` field to reference the REP that is being deprecated.
- Submit a pull request. During the pull request phase, the REP will undergo design feedback from both Bicep core maintainers and the broader community. The author should be prepared to make revisions based on this feedback.
- Build consensus and integrate feedback.
- When the Bicep core maintainers feel confident about the design and have reached a consensus, they should proceed to vote on whether to accept or reject the REP.

### Active

Once a REP is reviewed and approved, it is considered active. The author should take the following steps to maintain the active state of the REP:
- Replace `0000` in the REP file name with the PR number.
_ Merge the PR so that the REP file will appear in the active folder in the main branch.
- Proceed with the implementation of the proposed feature.
- Optionally, consider putting the feature behind a preview feature flag. This would give the author the opportunity to easily make revisions and breaking changes.
- As the author works towards a final implementation, submit PRs to the bicep-reps repository to keep the active REP in sync with the implementation.

### Final

REPs in the Final state are considered fully complete and implemented without any preview feature flag. Any proposed changes should be made through a new REP which enhances or deprecates the final REP. To convert an active REP to a final, follow these steps:
- Ensure that the active REP is fully implemented and well tested, and there will be no breaking changes to the customer facing API contract.
- Submit a PR to move the REP from the active folder to the final folder.
    - Optionally, as part of the PR change, if the REP marked as final enhances an existing final REP, include in the existing REP a `Enhanced By` field to reference the new REP.

### Inactive

An active REP should be moved inactive if the feature does not receive enough positive user feedback. To do so, the author should send a PR to move the REP file from active to inactive.

### Deprecated

A final REP should marked as deprecated if a new REP improves it in a non-backward compatible way. Follow the steps below:
- When the new REP reaches the final state, as part of the PR change, move the old REP to the deprecated folder.
- Within the deprecated REP file, add a `Deprecated By` attribute and reference the new REP.

### Rejected

A rejected REP will lead to a closed PR. In practice, this occurrence should be rare.

### Bicep REP Lifecycle Visualization

Refer to the state diagram below for a visual representation of the Bicep REP lifecycle.

```mermaid
stateDiagram-v2
    state Review <<choice>>
    [*] --> Draft: Open a PR
    Draft --> Review: Review the PR
    Review --> Active: PR accepted
    Review --> Rejected: PR rejected
    Active --> Active: Create new PRs to\n make revision
    Active --> Final: Feature fully implemented
    Active --> Inactive: Not enough feedback\n has been received to\n finalize the feature
    Final --> Deprecated: REP superceded by a new REP
    Rejected --> [*]
```
