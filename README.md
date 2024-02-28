# OpenFGA RFCs

Want to suggest a change to the OpenFGA project? That's great! We follow an RFC (Request for Comments) process for substantial changes to the project. An RFC is a way to propose, communicate and coordinate on new efforts for the project. 

## 👀 Open Requests For Comments

Check out the PRs below to see and contribute to features the core team and community are currently evaluating. Let us know your feedback with emojis and comments.

| Feature                                                                                                                                          | PR                                              | Status            |
|:-------------------------------------------------------------------------------------------------------------------------------------------------|:------------------------------------------------|:------------------|
| [ExpandedWatch API](https://github.com/openfga/rfcs/blob/expanded-watch-rfc/20220729-expandedWatch-api.md)                                       | [#4](https://github.com/openfga/rfcs/pull/4)    | Draft             |
| [Add type restrictions to DSL Syntax](https://github.com/openfga/rfcs/blob/type-restriction-dsl/20221012-add-type-restrictions-to-dsl-syntax.md) | [#7](https://github.com/openfga/rfcs/pull/7)    | Open for Comments |
| [Endpoint Authorizations](https://github.com/openfga/rfcs/blob/fe5eaeffc2230d162297296d4d3388fe4d065d44/20221103-endpoint-authz.md)              | [#10](https://github.com/openfga/rfcs/pull/10)  | Open for Comments |

---

## ✅ Accepted RFCs
| Feature                                                                                                                                |   Status |
|:---------------------------------------------------------------------------------------------------------------------------------------|---------:|
| [ListObjects API](https://github.com/openfga/rfcs/blob/main/20220714-listObjects-api.md)                                               | Accepted |
| [Add Type Restrictions to the JSON Syntax](https://github.com/openfga/rfcs/blob/main/20220831-add-type-restrictions-to-json-syntax.md) | Accepted |

---

## 💡 When the RFC process is necessary

The RFC process is necessary for changes which have a substantial impact on end users, operators, or contributors. "Substantial" is subjective, but it generally includes:

* Changes to core workflow functionality (models, tuples, new concepts).
* Changes to how OpenFGA is packaged, distributed, or configured.
* Changes with significant architectural implications.
* Changes which modify or introduce officially supported interfaces (HTTP APIs, external integrations, etc).

An RFC is not necessary for changes which have narrow scope and don't leave much to be discussed:

* Bug fixes and optimizations with no semantic change.
* Small features which only impact a narrow use case and affect users in an obvious way.

The RFC process aims to prevent wasted time and effort on substantial changes that end up being sent back to the drawing board. If your change takes minimal effort, or if you don't mind potentially scrapping it and starting over, feel free to skip this process. Do note however that pull requests may be closed with a polite request to submit an RFC.

If you're not sure whether to open an RFC for a change you'd like to propose, feel free to reach out through our [community channels](https://openfga.dev/community).

---

## 📋 RFC Process

### Proposal
To get a proposal into OpenFGA, an RFC first needs to be merged into the RFC repo. Once an RFC is merged, it's considered 'active' and may be implemented to be included in the project. These steps will get an RFC to be considered:

1. Fork the RFC repo: <https://github.com/openfga/rfcs>
1. Copy `proposal-template.md` to `YYYYMMDD-my-feature.md` (where 'YYYY' is the year, 'MM' is the numerical month, 'DD' is the numerical date, and 'my-feature' is descriptive).
1. Fill in the RFC template. 
    1. Any section can be marked as "N/A" if not applicable.
    1. Feel free to leave 'Relevant' sections blank until issues or PRs have been created.
    1. Consult the OpenFGA [design principles](https://github.com/openfga/rfcs/blob/main/DESIGN_PRINCIPLES.md) to guide your design.
1. Submit a pull request. The pull request is the time to get review of the proposal from the core team and larger community.
    1. Keep the description light; your proposal should contain all relevant information. Feel free to link to any relevant GitHub issues, since that helps with networking.
1. Build consensus and integrate feedback. RFCs that have broad support are much more likely to make progress than those that don't receive any comments.

### Development
Once a pull request is opened, the RFC is now in development and the following will happen:

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
1. The following labels will be applied as appropriate:
 * `resolution/merge`: the proposal will be merged; there are no outstanding objections, and implementation can begin as soon as the RFC is merged.
 * `resolution/close`: the proposal will be closed.
 * `resolution/postpone`: resolution will be deferred until a later time when the motivating or blocking factors may have changed.

### Merge
Once an RFC has been accepted, the sub-team maintainers should:
1. Create issues [referencing](https://docs.github.com/en/github/writing-on-github/autolinked-references-and-urls#issues-and-pull-requests) the RFC PR.
1. Label the PR with `issues-created/<sub-team>` after issues have been created or if zero issues necessary for a given sub-team.

Once an `issues-created/<sub-team>` label has been created for each sub-team, the RFC is ready to merge. The team member who merges the pull request should do the following:

1. Fill in the remaining metadata at the top.
1. Commit everything.
1. Update issues with RFC ID and a link to the text file.
1. Update any links in PR description to point at the committed file.
1. Remove the "Final Comment Period" label.
