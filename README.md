# OpenFGA RFCs

Want to suggest a change to the OpenFGA project? That's great!

We follow an RFC (Request for Comments) process for substantial changes to the project.

## RFC Process

### Proposal
To get a proposal into OpenFGA, an RFC first needs to be merged into the RFC repo. Once an RFC is merged, it's considered 'active' and may be implemented to be included in the project. These steps will get an RFC to be considered:

1. Fork the RFC repo: <https://github.com/openfga/rfcs>
1. Copy `proposal-template.md` to `0000-my-feature/proposal.md` (where 'my-feature' is descriptive). Don't assign an RFC number yet.
1. Fill in RFC template. 
    1. Any section can be marked as "N/A" if not applicable.
    1. Consult the OpenFGA [design principles](https://github.com/openfga/rfcs/blob/main/DESIGN_PRINCIPLES.md) to guide your design.
1. Submit a pull request. The pull request is the time to get review of the proposal from the core team and larger community.
    1. Keep the description light; your proposal should contain all relevant information. Feel free to link to any relevant GitHub issues, since that helps with networking.
1. Build consensus and integrate feedback. RFCs that have broad support are much more likely to make progress than those that don't receive any comments.

### Development
Once a pull request is opened, the RFC is now in development and the following will happen:

1. The following labels will be applied as appropriate:
 * `label1/<sublabel>`
 * `label2/<sublabel>`
1. The core team will discuss as much as possible in the RFC pull request directly. Any outside discussion will be summarized in the comment thread.
1. When deemed "ready", a core team member will propose a "motion for final comment period (FCP)" along with a disposition of the outcome (merge, close, or postpone). This is a step taken when enough discussion of the tradeoffs have taken place and the team is in a position to make a decision. Before entering FCP, super majority of the core team must sign off.

### Final Comment Period
When a pull request enters FCP the following will happen:
1. A core team member will apply the "Final Comment Period" label.
1. The FCP will last 7 days. If there's unanimous agreement among the team the FCP can close early.
1. For voting, the binding votes are comprised of the core team (and subteam maintainers if labeled as a subteam RFC). Acceptance requires super majority of binding votes in favor. The voting options are the following: 
    * Affirmative,
    * Negative, and 
    * Abstain. 
    
    Non-binding votes are of course welcome. Super majority means 2/3 or greater and no single company can have more than 50% of countable votes.
1. If no substantial new arguments or ideas are raised, the FCP will follow the outcome decided. If there are substantial new arguments, then the RFC will go back into development.

### Merge
Once an RFC has been accepted, the sub-team maintainers should:
1. Create issues [referencing](https://docs.github.com/en/github/writing-on-github/autolinked-references-and-urls#issues-and-pull-requests) the RFC PR.
1. Label the PR with `issues-created/<sub-team>` after issues have been created or if zero issues necessary for a given sub-team.

Once an `issues-created/<sub-team>` label has been created for each sub-team, the RFC is ready to merge. The team member who merges the pull request should do the following:

1. Assign an id based off the pull request number.
1. Rename the file based off the ID inside `text/`.
1. Fill in the remaining metadata at the top.
1. Commit everything.
1. Update issues with RFC ID and a link to the text file.
1. Update any links in PR description to point at the committed file.
1. Remove the "Final Comment Period" label.
