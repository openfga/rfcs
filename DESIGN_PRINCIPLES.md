# OpenFGA Design Principles

OpenFGA's goal is to solve authorization for any software project, regardless of its type or scale.

## Fast, Flexible Authorization, at any scale

OpenFGA is a versatile low latency authorization database that can scale to global deployments of any size. In order to maintain this vision, it should be only be concerned with fine-grained authorization. It is not a universal datastore, it is not a secret manager, it is not a search engine. Furthermore, it should strive to be fast, but not at the expense of correctness. In other words: if a request takes too long, it can be purposely slowed down, but it should eventually be served.

## Versatility

New concepts should only be introduced to OpenFGA if an intended authorization scenario cannot be precisely expressed in terms of existing concepts. Concepts should not be highly specialized for one domain of users, or introduce tight coupling to specific technologies. Authorization should be insulated from the constant churn of the tech industry.

## Developer-first

OpenFGA should be designed and operated in a way that works with developers, not against them. Instead of inventing new workflows and tooling just for the sake of adding to the feature set, the product should embrace existing best practices and work the way its users want to work, with a low barrier to entry.

## Platform- and Operator-friendly

OpenFGA should be designed to integrate with modern software delivery tooling and supply chains. If there is already an established system for key functionality (eg, logging, analytics, user management), OpenFGA should embrace and integrate with that system rather than trying to reinvent and establish its own solution.

## Expressive Authorization

OpenFGA should provide concepts that build a strong mental model for the user's project, and this model should remain intuitive as the user's authorization requirements grow.

Concepts should precisely outline their motivation and intended workflows. Friction and complexity resulting from the imprecise application of a concept should be a cue to introduce new ideas or refactor existing ones.
