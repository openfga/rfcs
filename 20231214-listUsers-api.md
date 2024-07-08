# Meta
[meta]: #meta
- **Name:** ListUsers and StreamedListUsers APIs
- **Start Date:** 2023-12-14
- **Authors**: @jon-whit, @willvedd, @miparnisari
- **Status:** Approved <!-- Acceptable values: Draft, Approved, On Hold, Superseded -->
- **RFC Pull Request:** https://github.com/openfga/rfcs/pull/15
- **Relevant Issues:**
  - https://github.com/openfga/roadmap/issues/16 (main roadmap issue)
  - https://github.com/openfga/openfga/issues/406
  - https://github.com/openfga/openfga/issues/1215

- **Supersedes:** N/A

## Table of Contents 
- [Summary](#summary)
- [Definitions](#definitions)
- [Motivation](#motivation)
- [What it is](#what-it-is)
- [API Semantics](#api-semantics)
- [API and Server Configuration Changes](#api-and-server-configuration-changes)
- [How it Works](#how-it-works)
- [Algorithm](#algorithm)
- [Concurrency Control](#concurrency-control)
- [Out of Scope](#out-of-scope)
- [Pruning/Directing Expansion with Graph Edges](#pruningdirecting-expansion-with-graph-edges)
- [Migration](#migration)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Prior Art](#prior-art)
- [Open Questions](#open-questions)

## Summary
[summary]: #summary

The `ListUsers` API will provide an API that answers the question, **who are all the users that have a relationship with an object**?

More specifically, given an [object](https://openfga.dev/docs/concepts#what-is-an-object), the `ListUsers` API will return all of the concrete/terminal [user](https://openfga.dev/docs/concepts#what-is-a-user) subjects that have a [relationship](https://openfga.dev/docs/concepts#what-is-a-relation) with that object.


## Definitions
[definitions]: #definitions

* You may see references to strings formatted as **`object#relation@user`** in this document. This is short-hand notation for representing an OpenFGA Relationship Tuple. For example, `group:eng#member@user:jon` or `document:1#viewer@group:eng#member` etc. You may also see strings formatted as `objectType#relation`. This is short-hand notation for representing a specific relation defined on some object type. For example, `document#viewer` represents the `viewer` relation defined on the `document` object type.

* **Expansion** - refers to the process of iteratively traversing the graph of relationships that are realized through both an OpenFGA model AND the relationship tuples that exist in the store.

* **Forward expansion** - refers to expansion of the OpenFGA relationship graph in a forward form. That is, starting at an object and relation, walk the graph of relationships in a directed way towards a user of some specific type by following paths that would lead through some target relation.

* **Reverse expansion** - refers to expansion of the OpenFGA relationship graph in a backwards form. That is, starting at a user of a specific type, walk the graph of relationships in a directed way towards an object of some specific type by following paths that would lead through some target relation.

* **Concrete/terminal objects** - a concrete or terminal object refers to a singular or specific resource of a given object type and id. A concrete object cannot be expanded any further (see Expansion above). It can be conceptualized as a leaf node in a graph. Concrete objects differ from usersets, because usersets refer to a collection of zero or more concrete objects. For example `user:jon` is a concrete object while `group:eng#member` is a userset which may expand to multiple concrete objects such as `user:jon`, `user:andres`, etc.

* **Userset** - A userset refers to a set of zero or more concrete objects that are the subjects that a relationship expands to. See our Concepts page on [Users](https://openfga.dev/docs/concepts#what-is-a-user) for more info.


## Motivation
[motivation]: #motivation

Developers using OpenFGA want to look up all of the users that have a particular relationship with an object. Currently, the Read API is inadequate for listing users because it can only return users who have direct access to an object, but not users that have derived access through expansion.

### Use Cases

- **UIs** - Display the users that a resource has been shared with. Ex: the "Share" dialog in Google Docs.
- **Notifications** - Notify all users who relate to a specific object. Ex: email all editors of a document after it has been updated.

### Expected Outcome

Two new core APIs are added to the [OpenFGA API](https://github.com/openfga/api) and implementations of these new APIs in the [OpenFGA server](https://github.com/openfga/openfga).

## What it is
[what-it-is]: #what-it-is

Given an `object`, `relation`, and one or more user/subject provided filters, return the concrete/terminal users or sets of users matching at least one of the user filters that have that relationship with the object.

For example, given the following model and relationship tuples:

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
    user_filters: [{type: "user"}]
}) --> {
  "users": [
    {
      "type": "user",
      "id": "anne"
    },
    {
      "type": "user",
      "id": "jon"
    }
  ]
}
```

**Example 2**
Looking at the type restrictions defined in the model, `group` objects are not directly related whatsoever to `document#viewer` relationships. So this example returns no results.
```
ListUsers({
    object: "document:1",
    relation: "viewer",
    user_filters: [{type: "group"}]
}) --> {
  "users": []
}
```

**Example 3**
You can see in the model that `group#member` relationships are directly related to `document#viewer`. So this example returns the `group:eng#member` direct relationship and the `group:fga#member` relationship indirectly through `group:eng#member`.
```
ListUsers({
    object: "document:1",
    relation: "viewer",
    user_filters: [
      {
        type: "group",
        relation: "member"
      }
    ]
}) --> 
{
  "users": [
    {
      "type": "group",
      "id": "eng",
      "relation": "member"
    },
    {
      "type": "group",
      "id": "fga",
      "relation": "member"
    }
  ]
}
```

## API Semantics
Unless otherwise noted, the intent is for the `ListUsers` and `StreamedListUsers` APIs to behave similarly to the `ListObjects` and `StreamedListObjects` API. This is to encourage more uniformity in the API experience. The API and server configuration should reflect similarities, the error propagation strategy should strive to be the same, and any limits and/or deadline behaviors should strive to be identical unless there is a compelling reason to have an exception. We may find such compelling reasons as we dig into the implementation details further, but it's not obvious why/if it would have to differ at this time.

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
        repeated ListUsersFilter user_filters = 5;
        repeated TupleKey contextual_tuples = 6;
        google.protobuf.Struct context = 7;
    }
    
    // ListUsersResponse represents a unary response with
    // all user result(s) up to the maximum limit.
    message ListUsersResponse {
      repeated User users = 1;
      repeated User excluded_users = 2;
    }

    message User {
      oneof user {
        Object object = 1;
        UsersetUser userset = 2;
        TypedWildcard wildcard = 3;
      }
    }
  
    message Object {
      string type = 1;
      string id = 2;
    }

    message UsersetUser {
      string type = 1;
      string id = 2;
      string relation = 3;
    }
  
    message TypedWildcard {
      string type = 1;
      Wildcard wildcard = 2;
    }

    message ListUsersFilter {
      string type = 1;
      string relation = 2;
    }
    ```
    
    The `ListUsersFilter` is an array of type restriction definitions that will be used to control the expansion. As ListUsers expansion occurs, if we find a user/subject that meets one of the `user_filters` type restriction(s), then we include the result in the response and stop further expansion on that subproblem. Once we've found a result meeting the filter criteria we don't explore that subproblem any further, even if it would lead to more results matching the other filters in the `user_filters` (see Example 5 below for further details).

    Initially `user_filters` will be limited to 1 filter (array of size 1), but we've planned on allowing the client to filter by multiple user filters in the future (as the community sees the need).

    - **StreamedListUsers**
    
    ```go
    rpc StreamedListUsers(StreamedListUsersRequest) returns (stream StreamedListUsersResponse)
    
    message StreamedListUsersRequest {
        string store_id = 1;
        string authorization_model_id = 2;
        string object = 3;
        string relation = 4;
        repeated ListUsersFilter user_filters = 5;
        repeated TupleKey contextual_tuples = 6;
        google.protobuf.Struct context = 7;
    }
    
    // StreamedListUsersResponse represents a single streaming user result 
    // returned from the streaming endpoint.
    message StreamedListUsersResponse {        
      User user = 1;
    }
    ```
    
- New server configurations (flags/config/env):

  - `--listUsers-max-results (uint32)` - limits the maximum size of the results returned by ListUsers (if 0 all results are returned up to the deadline, otherwise the response is limited to this size)

  - `--listUsers-deadline (duration)` - the timeout deadline for serving ListUsers or StreamedListUsers requests (default 3s)

  - `--max-concurrent-reads-for-list-users (uint32)` - the maximum allowed number of concurrent datastore reads in a single ListUsers or StreamedListUsers query

### Limits and Deadlines
The `ListUsers` API is a [unary RPC](https://grpc.io/docs/what-is-grpc/core-concepts/#unary-rpc) while the `StreamedListUsers` is a [server-streaming RPC](https://grpc.io/docs/what-is-grpc/core-concepts/#server-streaming-rpc). Because of these semantic differences we have different behaviors around limits and deadlines. 

The server flag `--listUsers-deadline` (mentioned above) sets an overall upper limit on the amount of time (e.g. the deadline) for serving a `ListUsers` or `StreamedListUsers` request. It controls how long time can be spent expanding results (e.g. finding relationships in the graph). If there are very few results and the deadline is generous, then the request will respond earlier than the deadline period. However, if there are many results and returning all of them would take longer then the deadline, then only the subset that can be returned in the deadline period are returned.

The server flag `--listUsers-max-results` (mentioned above) will limit the size of the set of results return from the unary `ListUsers` endpoint. It sets a hard upper limit on how many results can be returned. If the actual result set is less than this limit then the subset *should* be promptly returned so as to avoid keeping the client waiting longer than necessary. If the actual result set is at least as large as the max results limit but the results haven't been computed before the `--listUsers-deadline` period, then only the results which have been computed up to that deadline will be returned.

### Error Handling
ListUsers and StreamedListUsers should strive to implement error handling semantics inline with the way ListObjects and StreamedListObjects do. Namely, the API  should strive to fulfill the request with its limits as much as possible. For the unary ListUsers endpoint, if and only if it cannot fulfill the requested `--listUsers-max-results` and at least one error occurred, then an error should be surfaced. For the StreamedListUsers endpoint, as errors are encountered they should be yielded over the stream.

### Typed Public Wildcard

```go
model
  schema 1.1
type user
type employee

type document
  relations
    define viewer: [user:*, employee:*]
    define blocked: [user]
    define not_blocked: [user:*] but not blocked
```

| object     | relation    | user       |
|------------|-------------|------------|
| document:1 | viewer      | user:*     |
| document:1 | viewer      | employee:* |
| document:2 | not_blocked | user:*     | 
| document:2 | blocked     | user:anne  | 

**Example 1:**

```
ListUsers({
  object: "document:1",
  relation: "viewer",
  user_filters: [
    {
      type: "user",
    }
  ]
}) --> {
  "users": [
    {
      "type": "user",
      "wildcard": {}
    }
  ]
}
```
In example 1, there are two tuples establishing a typed wildcard, one for `user` and `employee`, both with `document:1#viewer`. But while expanding the ListUsers request we only return the `user` typed public wildcard because it is the only tuple that matches the filters in the `user_filters`.


**Example 2:**

```
ListUsers({
  object: "document:1",
  relation: "viewer",
  user_filters: [
    {
      type: "user",
    }
    {
      type: "employee",
    }
  ]
}) --> {
  "users": [
    {
      "type": "user",
      "wildcard": {}
    },
    {
      "type": "employee",
      "wildcard": {}
    }
  ]
}
```
However, in example 2, both `user` and `employee` typed wildcard are returned because those user types were specified in the `user_filters` input field.

**Example 3**

```
ListUsers({
    object: "document:2",
    relation: "not_blocked",
    user_filters: [ { type: "user" } ]
}) --> 
{
  "users": [
    {
      "type": "user",
      "wildcard": {}
    }
  ],
   "excluded_users": [
    {
      "type": "user",
      "id": "anne"
    }
  ]
}
```

`document:2` is publicly accessible, but `user:anne` is specifically blocked, so both are returned.

## Migration
[migration]: #migration

* No migrations or API breaking changes should be necessary for this work. We're extending the API surface, not changing it.

## Drawbacks
[drawbacks]: #drawbacks

* Increases API surface to maintain

* Another costly graph traversal query - when we added ListObjects and StreamedListObjects we had to work through some reliability improvements to ensure these new APIs didn't exhaust the database connection pool. These same considerations will be very relevant to this work as well.

## Alternatives
[alternatives]: #alternatives

- The client calls Check for every user in the system for the given `object` and `relation`

This is *an option*, but it is not one that is conducive to performance and/or handling more queries at scale. If many OpenFGA clients used Check for this pattern, then the server would have to be scaled up pretty high during more normal operation to account for this burst load. Additionally, it is also more challenging for a client to orchestrate this many Checks since we do not have a BatchCheck mechanism at this time.

- Naive server implementation which queries all subjects/users in the system that match the target subject/user filter and for each of them call Check on the provided `object` and `relation` (e.g. server-side BatchCheck naive implementation).

A preliminary implementation spike revealed that this approach would not cater to various of the use cases we explored including the Google Share Dialog that is discussed in more depth above in [Resolved Paths](#resolved-paths). For this reason and for performance concerns (e.g. the volume of Checks that would have to occur) we do not feel this is a viable approach.

- Encourage clients to implement Expand recursively themselves

There’s nothing stopping clients from implementing ListUsers today using the existing `Expand` API, and we won’t necessarily discourage it even after we implement ListUsers. However, we want to build ListUsers to provide this API more natively in the API offering and thus reduce duplication across the community and the burden on the client. We want to provide developers in the community with a simple to use API that doesn’t require reimplementing this logic anywhere it is needed. I anticipate the community will build API endpoints providing recursive Expand for this use case, and so we may as well offer it natively in the API.

## Prior Art
[prior-art]: #prior-art

The following code implements a POC implementation of `ListUsers`. The code is not quite complete when it comes to intersection and exclusion or the [typed public wildcard](#typed-public-wildcard) behavior described above, but it demonstrates the main algorithmic composition. Intersection and exclusion support will behave similarly to how intersection and exclusion behave in ListObjects. That is, a set of users that are contained under an intersection or exclusion will require additional Checks to resolve the set intersection or exclusion directly. 

https://github.com/jon-whit/openfga/blob/168986a51aae8281499aeb1b643d818621b70a07/pkg/server/commands/listusers/list_users_rpc.go

## Open Questions
[open-questions]: #open-questions

N/A
