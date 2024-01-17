# Meta
[meta]: #meta
- Name: ListUsers and StreamedListUsers APIs
- Start Date: 2023-12-14
- Author(s): @jon-whit
- Status: Draft <!-- Acceptable values: Draft, Approved, On Hold, Superseded -->
- RFC Pull Request: (leave blank)
- Relevant Issues:
  - https://github.com/openfga/roadmap/issues/16 (main roadmap issue)
  - https://github.com/openfga/openfga/issues/406
  - https://github.com/openfga/openfga/issues/1215

- Supersedes: N/A

# Summary
[summary]: #summary

The `ListUsers` and `StreamedListUsers` will provide an API that answers the question

> Who are all the users that have a relationship with an object?

More specifically, given an [object](https://openfga.dev/docs/concepts#what-is-an-object) and [relation](https://openfga.dev/docs/concepts#what-is-a-relation), these APIs will return all of the concrete/terminal [user](https://openfga.dev/docs/concepts#what-is-a-user) objects of a particular type that have that [relationship](https://openfga.dev/docs/concepts#what-is-a-relationship).


# Definitions
[definitions]: #definitions

* You may see references to strings formatted as **`object#relation@user`** in this document. This is short-hand notation for representing an OpenFGA Relationship Tuple. For example, `group:eng#member@user:jon` or `document:1#viewer@group:eng#member` etc.. You may also see strings formatted as `objectType#relation`. This is short-hand notation for representing a specific relation defined on some object type. For example, `document#viewer` represents the viewer relationship defined on the document object type.

* **Expansion** - refers to the process of iteratively traversing the graph of relationships that are realized through both an OpenFGA model AND the relationship tuples that exist in the store.

* **Forward expansion** - refers to expansion of the OpenFGA relationship graph in a forward form. That is, starting at an object and relation, walk the graph of relationships in a directed way towards a user of some specific type by following paths that would lead through some target relation.

* **Reverse expansion** - refers to expansion of the OpenFGA relationship grpah in a backwards form. That is, starting at a user of a specific type, walk the graph of relationships in a directed way towards an object of some specific type by following paths that would lead through some target relation.

* **Concrete/terminal objects** - a concrete or terminal object refers to a singular or specific resource of a given object type and id. A concrete object cannot be expanded any further (see Expansion above). It can be conceptualized as a leaf-node in a graph. Concrete objects differ from usersets, because usersets refer to a collection of zero or more concrete objects. For example `user:jon` is a concrete object while `group:eng#member` is a userset which may expand to multiple concrete objects such as `user:jon`, `user:andres`, etc..

* **Userset** - A userset refers to a set of zero or more concrete objects that are the subjects that a relationship expands to. See our Concepts page on [Users](https://openfga.dev/docs/concepts#what-is-a-user) for more info.


# Motivation
[motivation]: #motivation

- Why should we do this?

Developers using OpenFGA want to be able to do a reverse query and lookup all of the users that have a particular relationship with an object. 

- What use cases does it support?

**UIs** - display the users that a resource has been shared with. Think of the "Share" dialog in Google Docs, for example.

**todo: fill out more**


- What is the expected outcome?

Two new core APIs are added to the [OpenFGA API](https://github.com/openfga/api) and implementations of these new APIs in the [OpenFGA server](https://github.com/openfga/openfga).

# What it is
[what-it-is]: #what-it-is

Given an `object`, `relation`, some `user_object_type` and optional `user_relation`, return the concrete/terminal set of user objects of the target `user_object_type` and optional `user_relation` that have that relationship with the object.

More formally, given the following model and relationship tuples:

```
model
  schema 1.1

type user
type cat

type group
  relations
    define member: [user, group#member]

type document
  relations
    define viewer: [cat, user, group#member]
```

| object     | relation | user             | condition_name | context |
|------------|----------|------------------|----------------|---------|
| document:1 | viewer   | user:anne        | null           | null    |
| document:1 | viewer   | group:eng#member | null           | null    |
| group:eng  | member   | group:fga#member | null           | null    |
| group:fga  | member   | user:jon         | null           | null    |

ListUsers and StreamedListUsers would behave as follows:

> ℹ️ For the sake of brevity the examples below focus on the ListUsers unary RPC variant. The difference with the StreamedListUsers API is the streaming semantics of the response. Instead of returning an array, the streaming variant returns each result one at a time as they are "discovered".

**Example 1**
`user` object types are both directly and indirectly related to `document#viewer` relationships. The indirect relationships come through the direct relationships involving `group#member` (e.g. users can directly view a document or if the user is part of a group that can view the document)
```
ListUsers({
    object: "document:1",
    relation: "viewer",
    user_object_type: "user"
}) --> ["user:anne", "user:jon"]
```

**Example 2**
Looking at the type restrictions defined in the model, `group` objects are not directly related whatsoever to `document#viewer` relationships. So this example returns no results.
```
ListUsers({
    object: "document:1",
    relation: "viewer",
    user_object_type: "group"
}) --> []
```

**Example 3**
You can see in the model that `group#member` relationships are directly related to `document#viewer`. So this example returns the `group:eng#member` direct relationship and the `group:fga#member` relationship indirectly through `group:eng#member`.
```
ListUsers({
    object: "document:1",
    relation: "viewer",
    user_object_type: "group",
    user_relation: "member"
}) --> ["group:eng", "group:fga"]
```

## API Semantics
Unless otherwise noted, the intent is for the ListUsers and StreamedListUsers APIs to behave similarly with the ListObjects and StreamedListObjects API. This is to encourage more uniformity in the API experience. The API and server configuration should reflect similarities, the error propagation strategy should strive to be the same, and any limits and/or deadline behaviors should strive to be identical unless there is a compelling reason to have an exception. We may find such compelling reasons as we dig into the implementation details further, but it's not obvious why/if it would have to differ at this time.

## API and Server Configuration Changes
- Introduce the new protobuf API definitions.

  > ℹ️ The various protobuf annotations have been omitted for brevity, but assume they are identical to those existing annotations for the same or similar fields used in other endpoints.
  
    - **ListUsers**
    
    ```protobuf
    rpc ListUsers(ListUsersRequest) returns (ListUsersResponse)
    
    message ListUsersRequest {
        string store_id = 1;
        string authorization_model_id = 2;
        string object = 3;
        string relation = 4;
        string user_object_type = 5;
        string user_relation = 6;
        repeated TupleKey contextual_tuples = 7;
        google.protobuf.Struct context = 8;
    }
    
    // ListUsersResponse represents a unary response with
    // all user result(s) up to the maximum limit.
    message ListUsersResponse {
      repeated string users = 1; // e.g. document:1
    }
    ```
    
    - **StreamedListUsers**
    
    ```go
    rpc StreamedListUsers(StreamedListUsersRequest) returns (stream StreamedListUsersResponse)
    
    message StreamedListUsersRequest {
        string store_id = 1;
        string authorization_model_id = 2;
        string object = 3;
        string relation = 4;
        string user_object_type = 5;
        string user_relation = 6;
        repeated TupleKey contextual_tuples = 7;
        google.protobuf.Struct context = 8;
    }
    
    // StreamedListUsersResponse represents a single streaming user result 
    // returned from the streaming endpoint.
    message StreamedListUsersResponse {        
      string user = 1; // e.g. document:1
    }
    ```
    
- New server configurations (flags/config/env):

  - `--listUsers-max-results (uint32)` - limits the maximum size of the results returned by ListUsers (if 0 all results are returned up to the deadline, otherwise the response is limited to this size)

  - `--listUsers-deadline (duration)` - the timeout deadline for serving ListUsers or StreamedListUsers requests (default 3s)

  - `--max-concurrent-reads-for-list-objects (uint32)` - the maximum allowed number of concurrent datastore reads in a single ListUsers or StreamedListUsers query

### Limits and Deadlines
The `ListUsers` API is a [unary RPC](https://grpc.io/docs/what-is-grpc/core-concepts/#unary-rpc) while the `StreamedListUsers` is a [server-streaming RPC](https://grpc.io/docs/what-is-grpc/core-concepts/#server-streaming-rpc). Because of these semantic differences we have different behaviors around limits and deadlines. 

The server flag `--listUsers-deadline` (mentioned above) sets an overall upper limit on the amount of time (e.g. the deadline) for serving a `ListUsers` or `StreamedListUsers` request. It controls how long time can be spent expanding results (e.g. finding relationships in the graph). If there are very few results and the deadline is generous, then the request will respond earlier than the deadline period. However, if there are many results and returning all of them would take longer then the deadline, then only the subset that can be returned in the deadline period are returned.

The server flag `--listUsers-max-results` (mentioned above) will limit the size of the set of results return from the unary `ListUsers` endpoint. It sets a hard upper limit on how many results can be returned. If the actual result set is less than this limit then the subset *should* be promptly returned so as to avoid keeping the client waiting longer than necessary. If the actual result set is at least as large as the max results limit but the results haven't been computed before the `--listUsers-deadline` period, then only the results which have been computed up to that deadline will be returned.

### Error Handling


### Typed Public Wildcard
Nothing special is required when handling typed wildcards. Namely, consider this model, tuples, and request:

```go
type user

type document
  relations
    define viewer: [user:*]
```

| object     | relation | user        |
|------------|----------|-------------|
| document:1 | viewer   | user:*      |

```
ListUsers({
  object: "document:1",
  relation: "viewer",
  user_object_type: "user"
}) --> ["user:*"]
```
In this example there is a single tuple establishing a typed wildcard of type `user` with `document:1#viewer`, and while expanding the ListUsers request we yield the typed wildcard directly because it matches the desired `user_object_type`.

# How it Works
[how-it-works]: #how-it-works

## Algorithm

`ListUsers` and `StreamedListUsers` is a filtered form of recursive [Expand](https://openfga.dev/api/service#/Relationship%20Queries/Expand). We don’t want to return *all users of any type*, we only want to return all the concrete user objects *of a specified type*.

The implementation of ListUsers and StreamedListUsers will behave a lot like [ListObjects](https://openfga.dev/api/service#/Relationship%20Queries/ListObjects) and [StreamedListObjects](https://openfga.dev/api/service#/Relationship%20Queries/StreamedListObjects), but instead of starting at a user and expanding backwards (e.g. reverse expansion) we will start at a relationship with an object and recursively expand (forward expansion) any usersets we find along the way that would lead to a concrete/terminal object of the specified `user_object_type`.


> ℹ️ Most of the implementation details we spent time tuning with `ListObjects` apply to this problem as well. Namely, bounding the number of concurrent evaluation paths, bounding the number of concurrent database queries that can be inflight per request, constraining the response results and/or the streaming deadline, etc.. We should be able to move more quickly on the implementation phase as a result of the prior work and education from ListObjects.

Here’s are a few examples to demonstrate the algorithm:

### Example 1a (Direct Relation with Object)
```
model
  schema 1.1

type user

type document
  relations
    define viewer: [user]
```

| object     | relation | user        |
|------------|----------|-------------|
| document:1 | viewer   | user:jon    |
| document:1 | viewer   | user:andres |


```
ListUsers({
  object: "document:1",
  relation: "viewer",
  user_object_type: "user"
}) --> ["user:jon", "user:andres"]
```
In this case the algorithm simply must expand all direct relationships between `document:1#viewer` and `user` objects. Since these are directly related to one another, then we must only do a simple database lookup which equates to `SELECT user_object_id FROM tuple WHERE object_type="document" AND object_id="1" AND relation="viewer" AND user_object_type="user"`

### Example 1b (Direct Relation with Userset)
```
model
  schema 1.1

type user

type group
  relations
    define member: [user, group#member]

type document
  relations
    define viewer: [group#member]
```
| object         | relation | user                  |
|----------------|----------|-----------------------|
| document:1     | viewer   | group:eng#member      |
| group:eng      | member   | group:fga#member      |
| group:fga      | member   | user:andres           |
| group:fga      | member   | group:fga-core#member |
| group:fga-core | member   | user:jon              |

```
ListUsers({
  object: "document:1",
  relation: "viewer",
  user_object_type: "user"
}) --> ["user:jon"]
```
This example requires recursive expansion.
> ℹ️ The usage of `expand` from here forward refers to an internal expand function, and it should not be confused with the public Expand API, but it operates somewhat similarly.


1. Expand the set of subjects/users that relate to `document:1#viewer`. 

   `expand(document:1#viewer)` --> ["group:eng#member"]

1. Expand the new set of subjects/users that make up `group:eng#member` set.

   `expand(group:eng#member)` --> ["group:fga#member"]

1. Expand the set of subjects/users that make up the `group:fga#member` set.

   `expand(group:fga#member)` --> ["user:andres", "group:fga-core#member"]

   1. We find a terminal/concrete object of `user:andres`, which is of the target user_object_type, and so we add it to the list of items to include in the response.

1. Continue expanding the residual set of subjects/users in `group:fga-core#member`

   `expand(group:fga-core#member)` --> ["user:jon"]

   1. We find a terminal/concrete object of `user:jon`, which is of the target user_object_type, so add it to the list of items to include in the response

1. No further subjects to expand, and so we're done. Return the items we accumulated from the steps above.

Visually, the overall recursive call tree looks like the following:

```
expand(document:1#viewer) --> ["group:eng#member"]
|-> expand(group:eng#member) --> ["group:fga#member"]
|---> expand(group:fga#member) --> ["user:andres", "group:fga-core#member"]
      yield "user:andres"
|-----> expand(group:fga-core#member) --> ["user:jon"]
        yield "user:jon"

return ["user:andres", "user:jon"]
```


### Example 1c (Direct Relation with Typed Public Wildcard)
```
model
  schema 1.1

type user

type document
  relations
    define viewer: [user:*]
```

### Example 2 (Computed Relationship)
```
model
  schema 1.1

type user
type person

type document
  relations
    define editor: [user, person]
    define viewer: editor
```
| object     | relation | user       |
|------------|----------|------------|
| document:1 | editor   | user:jon   |
| document:1 | editor   | person:bob |

```
ListUsers({
  object: "document:1",
  relation: "viewer",
  user_object_type: "user"
}) --> ["user:jon"]
```
This example demonstrates a simple rewritten relation involved in the expansion. Instead of expanding `document:1#viewer` we immediately rewrite that to `document:1#editor` and expand that. Namely,

1. Rewrite document#viewer to document#editor through computed_userset.

   `expand(document:1#viewer)` --rewritten--> `expand(document:1#editor)`

1. Expand the new (rewritten) relationship.

   `expand(document:1#editor)` --> ["user:jon", "person:bob"]

   1. We find terminal/concrete objects including `user:jon` and `person:bob`. `user:jon` is of the target user_object_type, so add it to the list of items to include in the response, but we filter out/omit `person:bob`.

1. No further subjects to expand, and so we're done. Return the items we accumulated from the steps above.

Visually, the overall recursive call tree looks like the following:
```
expand(document:1#viewer) (rewritten)
|-> expand(document:1#editor) --> ["user:jon", "person:bob"]
    filter(["user:jon", "person:bob"])
    yield "user:jon"

return ["user:jon"]
```

### Example 3 (TTU Relationship)

### Example 4 (Expanding Relationships With `objectType#relation`)

```
model
  schema 1.1

type user

type group
  relations
    define member: [user]

type document
  relations
    define viewer: [group#member]
```
| object     | relation | user             |
|------------|----------|------------------|
| document:1 | viewer   | group:eng#member |
| group:eng  | member   | group:fga#member |
```
ListUsers({
  object: "document:1",
  relation: "viewer",
  user_object_type: "group"
  user_relation: "member"
}) --> ["group:eng", "group:fga"]
```
This example deviates from many of the examples above in that we expand all relationships for a specific object and relation (e.g. `document:1#viewer`) that are related to a given set of users or subject set (e.g. `group#member`).

1. Expand the set of subjects/users that relate to `document:1#viewer`.

   `expand(document:1#viewer)` --> ["group:eng#member"]

   1. We find terminal/concrete object `group:eng#member`, which is of the target user_object_type (group) and user_relation (member), so add it to the list of items in the response.

1. Expand the set of subjects/users that relate to `group:eng#member`.

   `expand(group:eng#member)` --> ["group:fga#member"]

    1. We find terminal/concrete object `group:fga#member`, which is of the target user_object_type (group) and user_relation (member), so add it to the list of items in the response.

1. Expand the set of subjects/users that relate to `group:fga#member`.

   `expand(group:fga#member)` --> []

1. No further subjects to expand, and so we're done. Return the items we accumulated from the steps above.

Visually, the overall recursive call tree looks like the following:
```
expand(document:1#viewer) --> ["group:eng#member"]
yield group:eng
|-> expand(group:eng#member) --> ["group:fga#member"]
    yield group:fga
|---> expand(group:fga#member) --> []

return ["group:eng", "group:fga"]
```


### Intersection and Exclusion
For relationships that involve an intersection (e.g. `a and b`) or exclusion (e.g. `a but not b`) we'll apply the same algorithmic approach we do in ListObjects. Namely, given the set `a and b` we'll compute `a` and then call `Check` for each of the results to resolve the set intersection through Check resolution. Likewise, for `a but not b` we'll compute `a` and then call `Check` for each of the results to resolve the set difference.

This choice is an algorithmic choice that exploits the fact that `a and b` and `a but not b` can be no larger than the `max(size(a), size(b))`, so instead of computing the results for both sets, holding the results temporarily in memory and then resolving the overlap, we simply compute the first set and then use Check resolution to resolve the residual.

## Concurrency Control
Similar to the mitigations we've implemented in Check and ListObjects, a single ListUsers subproblem could fan-out to hundreds or more of repetitive expansions. Consequently, we must limit the breadth of the number of expansions that can be dispatched at any level as well as limit the total depth of expansion. 

For widely nested sets of relationship, consider the following model and relationship tuples:

```
model
  schema 1.1

type user

type group
  relations
    define member: [user, group#member]
```
| object  | relation | user           |
|---------|----------|----------------|
| group:1 | member   | group:2#member |
| group:1 | member   | group:3#member |
| group:1 | member   | ...            |
| group:1 | member   | group:N#member |
| group:N | member   | user:jon       |

If a developer were to call 
```
ListUsers({
  object: "group:1",
  relation: "member",
  user_object_type: "user"
})
```
then, for large `N`, this would cause a high degree of expansive breadth and saturate the server CPU and memory.

Similarly, for recursively expanding deeply nested sets, consider the same model above but the following tuples:
| object  | relation | user           |
|---------|----------|----------------|
| group:1 | member   | group:2#member |
| group:2 | member   | group:3#member |
| ...     | member   | ...            |
| group:N | member   | user:jon       |

If a deveoper were to call
```
ListUsers({
  object: "group:1",
  relation: "member",
  user_object_type: "user"
})
```
then, for large `N`, this would cause a high degree of depth and saturate the server CPU and memory.

## Pruning/Directing Expansion with Graph Edges
> ℹ️ Failure to prune expansion could lead to more excessive server exhaustion and be a reliability and performance concern.

A naive implementation would blindly expand subject/user sets without using any of the type restriction information included in the model (e.g. neglecting concrete edges in the graph). A more optimized approach to the iterative expansion algorithm described above would use the type restriction information to prune the expansion space. For example, consider the following model and relationship tuples:

```
model
  schema 1.1

type person
type user

type group
  relations
    define member: [person, group#member]

type document
  relations
    define viewer: [group#member]
```

| object     | relation | user           |
|------------|----------|----------------|
| document:1 | viewer   | group:1#member |
| document:1 | viewer   | group:2#member |
| document:1 | viewer   | ...            |
| document:1 | viewer   | group:N#member |
| group:N    | member   | person:bob     |

If a developer were to call
```
ListUsers({
  object: "document:1",
  relation: "viewer",
  user_object_type: "user"
})
```
then a naive implementation would start expanding all `group#member` relationships only to find that `user` objects aren't related to any group members. This is because the type restrictions would not allow such relationships to exist in the first place. In this case, if `N` is large, this is a server concern. Consequently, we should avoiding expanding edges that would not lead to a terminal object that is the desired user_object_type. We can do so by using the relationship edges information we have available in the OpenFGA typesystem.

# Migration
[migration]: #migration

* No migrations or API breaking changes should be necessary for this work. We're extending the API surface, not changing it.

# Drawbacks
[drawbacks]: #drawbacks

* Increases API surface to maintain

* Another costly graph traversal query - when we added ListObjects and StreamedListObjects we had to work through some reliability improvements to ensure these new APIs didn't exhaust the database connection pool. These same considerations will be very relevant to this work as well.

# Alternatives
[alternatives]: #alternatives

- Encourage clients to implement Expand recursively themselves

There’s nothing stopping clients from implementing ListUsers today using the existing `Expand` API, and we won’t necessarily discourage it even after we implement ListUsers. However, we want to build ListUsers to provide this API more natively in the API offering and thus reduce duplication across the community and the burden on the client. We want to provide developers in the community with a simple to use API that doesn’t require reimplementing this logic anywhere it is needed. I anticipate the community will build API endpoints providing recursive Expand for this use case, and so we may as well offer it natively in the API.
- Call Check for every user in the system for the given `object` and `relation`

This is *an option*, but it is not one that is conducive to performance and/or handling more queries at scale. If many OpenFGA clients used Check for this pattern, then the server would have to be scaled up pretty high during more normal operation to account for this burst load. Additionally, it is also more challenging for a client to orchestrate this many Checks since we do not have a BatchCheck mechanism at this time.

# Prior Art
[prior-art]: #prior-art

The following code implements a POC implementation of `ListUsers`. The code is not quite complete when it comes to intersection and exclusion or the typed wildcard behavior described above, but it demonstrates the main algorithmic composition. Intersection and exclusion support will behave similarly to how intersection and exclusion behave in ListObjects. That is, a set of users that are contained under an intersection or exclusion will require additional Checks to resolve the set intersection or exclusion directly. 

https://github.com/jon-whit/openfga/blob/168986a51aae8281499aeb1b643d818621b70a07/pkg/server/commands/listusers/list_users_rpc.go

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to be resolved before this gets merged?
- What parts of the design do you expect to be resolved through implementation of the feature?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?