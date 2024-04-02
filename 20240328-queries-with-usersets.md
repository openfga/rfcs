# Meta
[meta]: #meta
- Name: Queries with usersets
- Start Date: 2024-03-28
- Author(s): @miparnisari
- Status: Draft
- RFC Pull Request: (leave blank)
- Relevant Issues: n/a
- Supersedes: n/a

# Summary
[summary]: #summary

This is a proposal to change the outcome of query APIs (specifically, Check and ListObjects APIs) when the target of the query is a userset. The proposal also involves adapting ListUsers API to return responses consistent with Check and ListObjects.

# Definitions
[definitions]: #definitions

- **Object**: is of the form `objectType:objectId`.
- **User**: is a specific object such as `employee:maria` or `document:1`, or a userset, or a wildcard (described below).
- **Userset**: is a set of objects that have a specific relation. Usersets are represented via this notation: `objectType:objectId#relation`. For example, `document:1#viewer` represents the set of objects that are related to `document:1` as `viewer`.
- **Wildcard user**: an object with the form `objectType:*` that denotes all objects of the given `objectType`.
- **Tuple**: is a relation in the system. Tuples are represented via this notation: `object#relation@(user|userset|wildcard)`. Writing a tuple such as `document:1#viewer@employee:*` means that every object of type `employee` has `viewer` relation to `document:1`.

# Motivation
[motivation]: #motivation

## Correctness

The userset `document:1#viewer` is defined as the set that has the `viewer` relation with `document:1`. Therefore, intuitively, `Check(object=document:1, relation=viewer, user=document:1#viewer)` should always return `{allowed=true}`.

## Consistency between APIs

OpenFGA strives to maintain consistency in the responses of every query API. That is, barring server flags that limit the size of the responses,

1. If a call to `Check(object=document:1, relation=viewer, user=document:1#viewer)` returns `{allowed:true}`, then we intuitively expect the equivalent ListObjects call `ListObjects(type=document, relation=viewer, user=document:1#viewer)` to return `[document:1]`.
2. If a call to `Check(object=document:1, relation=viewer, user=document:1#viewer)` returns `{allowed:true}`, then we intuitively expect the equivalent ListUsers call `ListUsers(object=document:1, relation=viewer, userFilter=[document#viewer])` to return `[document:1#viewer]`.

## Leopard indexing

In the Zanzibar paper they described the mechanics behind the Leopard indexing system. They mentioned building an index by taking the intersection between two sets - a Member2Group (M2G) and Group2Group (G2G) index which stored direct user to group relationships and group to group relationships, respectively.

Let's assume that we have an FGA model such as the following:
```
model
  schema 1.1
type employee
type group
  relations
    define member: [employee, group#member]
```

and the following relationship tuples:
- `group:eng#member@group:fga#member`
- `group:fga#member@employee:jon`

The M2G index would be defined as:

| object_type | object_id | relation | subject_type | subject_id | subject_relation |
|-------------|-----------|----------|--------------|------------|------------------|
| group       | fga       | member   | employee     | jon        |                  |

And the G2G index, naively, would be defined as:

| object_type | object_id | relation | subject_type | subject_id | subject_relation |
|-------------|-----------|----------|--------------|------------|------------------|
| group       | eng       | member   | group        | fga        | member           |

We take the intersection of M2G and G2G, namely

```sql
SELECT g2g.object_type, g2g.object_id, g2g.relation, m2g.subject_type, m2g.subject_id
FROM M2G m2g 
INNER JOIN G2G on m2g.object_type=g2g.object_type AND m2g.relation=g2g.relation
```
which returns

| object_type | object_id | relation | subject_type | subject_id |
|-------------|-----------|----------|--------------|------------|
| group       | eng       | member   | employee     | jon        |

Notice that we account for the `group:eng#member` relationship for `user:jon`, but the `group:fga#member` relationship is missing. This is because we naively overlooked the fact that `group:fga#member` defines itself. That is, every subject who is a `member` of `group:fga` is definitely a subject of `group:fga#member`. A subject set defines itself, because it is reflexive. So our system should represent this and reflect this natively in our APIs, and what we should have had for the G2G index as a side effect of this property is the following:

| object_type | object_id | relation | subject_type | subject_id | subject_relation |
|-------------|-----------|----------|--------------|------------|------------------|
| group       | eng       | member   | group        | eng        | member           |
| group       | eng       | member   | group        | fga        | member           |
| group       | fga       | member   | group        | fga        | member           |

Then our join would return:

| object_type | object_id | relation | subject_type | subject_id |
|-------------|-----------|----------|--------------|------------|
| group       | eng       | member   | employee     | jon        |
| group       | fga       | member   | employee     | jon        |

# What it is
[what-it-is]: #what-it-is

We are introducing the notion of invariants in a model, i.e. facts that were never explicitly written in the system but that will be accepted as true. These facts stem from applying equivalence of sets to the authorization model.

For example, given this authorization model:

```go
model
  schema 1.1
type employee
type group
  relations
    define member: [employee]
type document
  relations
    define a: [employee]
    define b: [employee]
    define c: [group#member]
    define computed: a
    define union: a or b
    define intersection: a and b
    define difference_1: a but not b
    define difference_2: c but not a 
    define parent: [group]
    define tuple_to_userset: member from group
```

The proposal is to change the behavior as follows:

| Tuples                              | Query                                                     | Before            | After                         |
|-------------------------------------|-----------------------------------------------------------|-------------------|-------------------------------|
| -                                   | Check(document:1#a@document:1#a)                          | `{allowed=false}` | `{allowed=true}`              |
| -                                   | Check(document:1#computed@document:1#a)                   | `{allowed=false}` | `{allowed=true}`              |
| -                                   | Check(document:1#union@document:1#a)                      | `{allowed=false}` | `{allowed=true}`              |
| -                                   | Check(document:1#union@document:1#b)                      | `{allowed=false}` | `{allowed=true}`              |
| document:1#parent:group:marketing   | Check(document:1#tuple_to_userset@group:marketing#member) | `{allowed=false}` | `{allowed=true}`              |
| -                                   | Check(document:1#intersection@document:1#a)               | `{allowed=false}` | `{allowed=false}` (no change) |
| -                                   | Check(document:1#intersection@document:1#b)               | `{allowed=false}` | `{allowed=false}` (no change) |
| -                                   | Check(document:1#difference_1@document:1#a)               | `{allowed=false}` | `{allowed=false}` (no change) |
| -                                   | Check(document:1#difference_1@document:1#a)               | `{allowed=false}` | `{allowed=false}` (no change) |
| document:1#c@group:marketing#member | Check(document:1#difference_2@group:marketing#member)     | `{allowed=true}`  | `{allowed=true}`  (no change) |


https://play.fga.dev/stores/create/?id=01HT3QDCENG2D0A3KS0CNQSTAJ.

# How it Works
[how-it-works]: #how-it-works

Check API involves solving a parent problem that may contain subproblems. The implementation of this proposal will be that when solving the parent problem or each subproblem, if the problem is of the form `objectType:objectId # relation @ objectType:objectId#relation`, then we will immediately return `{allowed=true}`.

The same idea will apply to ListObjects and ListUsers APIs.

# Migration
[migration]: #migration

This proposal doesn't entail any migration of data. However, it will require a thorough review by application developers and authorization model authors to ensure that their applications are making the right queries to OpenFGA. Some developers making Check API calls with a userset in the `user` field may want to change this to be a specific user, as the application may start granting more permissions than desired.

# Drawbacks
[drawbacks]: #drawbacks

- Confusion for application developers. So far, developers have largely assumed that a Check returns `{allowed: true}` if and only if there is a tuple or tuples in the system saying so. With this proposal we are introducing the idea of invariant relationships.

# Alternatives
[alternatives]: #alternatives

N/A

# Prior Art
[prior-art]: #prior-art

N/A

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

- Given that the userset `document:1#viewer` is defined as the set that has the `viewer` relation with `document:1`, should we block application developers from writing the tuple `document:1#viewer@document:1#viewer` since the Write will become irrelevant? And if yes:
  - What error should we return?
  - What do we do with tuples like these that have already been written?
- How does this affect the [Read API](https://openfga.dev/api/service#/Relationship%20Tuples/Read)? The documentation says 
    > The Read API will return the tuples for a certain store that match a query filter specified in the body of the request. [...] it only returns relationship tuples that are stored in the system and satisfy the query.

    If a developer calls `Read(user=document:1#viewer, relation:viewer, object=document:1)` today, unless they previously wrote the tuple `document:1 # viewer @ document:1#viewer`, this query returns an empty result, even though the query is an invariant that holds true.
  - Should we update documentation to say that invariants are not returned?
- How does this affect the [Expand API](https://openfga.dev/api/service#/Relationship%20Queries/Expand)? 
  - Should we update documentation to say that invariants are not returned?
