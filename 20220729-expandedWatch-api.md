# Meta
[meta]: #meta
- **Name**: ExpandedWatch
- **Start Date**: 2022-07-29
- **Author(s)**: jon-whit
- **Status**: Draft <!-- Acceptable values: Draft, Approved, On Hold, Superseded -->
- RFC Pull Request: (leave blank)
- **Relevant Issues**:
  <!-- List relevant Github issues here -->
- **Supersedes**: (put "N/A" unless this replaces an existing RFC, then link to that RFC)

# Summary
[summary]: #summary

The `ExpandedWatch` API will provide a solution to [Search with Permissions (Option 2)][1]. More specifically, it will give clients the ability to consume a fully expanded (or flattened) change set of one or more relationships for one or more object types, and they can use this change set to build an access aware index alongside the data they want to filter (based on search and permission criteria).

# Definitions
[definitions]: #definitions

# Motivation
[motivation]: #motivation

The purpose of the ExpandedWatch API is to assist in solving two primary problems:

- “Search with Permissions” (e.g. access aware indexes) - Take the intersection between a search filter and a permissions filter on arbitrarily large datasets. This is outlined in more detail in [Search with Permissions (Option 2)][1]. 

- Nested group optimization - Flatten user-to-group and group-to-group memberships to optimize nested group lookups when evaluating Check, Expand, and other queries.

There are many use cases where a client of OpenFGA may need to filter/sort data in their application but also apply an access aware filter to their dataset(s). ExpandedWatch should provide clients an API to build this index externally in the client's database for Search with Permissions use cases.

# What it is
[what-it-is]: #what-it-is

The following query demonstrates a query that a client might use to apply an access aware index to a query matching some search criteria (in this case any document whose name starts with ‘example’):

```
SELECT id, name
FROM   documents
       INNER JOIN permissions
               ON documents.id = permissions.object_id
WHERE  documents.name LIKE ‘example%’
       AND permissions.relation = ‘viewer’
       AND permissions.user = ‘jon’
       AND permissions.allowed=true
```

If the following data existed in the client application’s database:

```
postgres=# select * from documents;

  id  |   name
------+----------
 doc1 | exampleA
 doc2 | exampleB
 doc3 | somedoc

postgres=# select * from permissions;

 object_id | relation | user | allowed
-----------+----------+------+--------
 doc1      | viewer   | jon  | true
 doc2      | viewer   | jon  | false
 doc3      | viewer   | jon  | true
``` 

Then the query above would return the result:

```
  id  |   name
------+----------
 doc1 | exampleA
 ```

This is because ‘exampleA’ and ‘exampleB’ are the only two matching results that match the search criteria but of those two documents user ‘jon’ only has access to ‘doc1’ (exampleA).

ExpandedWatch allows a client to build, for example, the ‘permissions’ table demonstrated in the example above. If this table can be constructed by the client using the ExpandedWatch API, then the client can use their database to perform database native joins on the permission table and get highly performant (and scalable) permission aware filtering on any arbitrary dataset they may have.

---
Let’s consider the following authorization model and relationship tuples exist in a particular OpenFGA store:
```
type document
  relations
    define parent as self
    define editor as self
    define viewer as self or editor or viewer from parent

type folder
  relations
    define viewer as self
    
type group
  relations
    define member as self
```
| object            | relation | user                     |
|-------------------|----------|--------------------------|
| folder:folder1    | viewer   | group:engineering#member |
| document:docX     | parent   | folder:folder1           |
| document:docY     | parent   | folder:folder1           |
| document:docY     | viewer   | jon                      |
| group:engineering | member   | group:openfga#member     |
| group:engineering | member   | alberto                  |
| group:openfga     | member   | jon                      |

If an OpenFGA client called:
```
results := openfgaClient.ExpandedWatch({
    StoreID: "mystore",
    Type: "document",
    Relation: "viewer",
})
```
followed by, for example:
```
openfgaClient.Write({
    Deletes: {"folder:folder1#viewer@group:engineering#member"}
})
```
then `results`, for example, would look like the following:
```
print(results)
{
    "relationship_updates": [
      {
        "relationship_status": "NO_RELATIONSHIP",
        "object": {
          "type": "document",
          "id": "docX"
        },
        "relation": "viewer",
        "user_id": "alberto"
      },
      {
        "relationship_status": "NO_RELATIONSHIP",
        "object": {
          "type": "document",
          "id": "docX"
        },
        "relation": "viewer",
        "user_id": "jon"
      },
      {
        "relationship_status": "NO_RELATIONSHIP",
        "object": {
          "type": "document",
          "id": "docY"
        },
        "relation": "viewer",
        "user_id": "alberto"
      },
      {
        "relationship_status": "HAS_RELATIONSHIP",
        "object": {
          "type": "document",
          "id": "docY"
        },
        "relation": "viewer",
        "user_id": "jon"
      }
    ]
}
```
These `relationship_updates` can continuously be consumed by the client and written into their local `permissions` table (as demonstrated above) so that they can perform permission aware filtering to any arbitrary dataset.

# How it Works
[how-it-works]: #how-it-works

## API Changes
We’ll introduce at least two new internal APIs that will be used to compute the expanded change set that is caused by a single tuple change in the system, and public API(s) that will be used to serve the expanded change set.

### ExpandedWatch API (public)
**Summary**: The ExpandedWatch endpoint will implement a grpc server streaming RPC that behaves like a database changefeed for changes to relationships in the graph of relationships. It will use the [ReadChanges API][read-changes] and react to changes to tuples by computing the other relationships impacted by a single tuple change.

This API will start an expanded watch over changes to relationships and stream the expanded change set back to the client. The expanded change set will be limited to changes to a single relationship for a single object type.

> ⁉️ In the future we may find that we want to allow ExpandedWatch to serve the expanded change set for multiple object types and multiple relationships, but it’s not uncommon to only index a couple of relationships for a couple of types, so this is a good starting point.

The `continuation_token` in the request can be used to start processing changes from a particular point in time in the past. The ulid of the tuple changelog entry encoded in the continuation token will be updated as each change is processed (by expanding the change into the full change set impacted by it).

```
rpc ExpandedWatch(ExpandedWatchRequest) 
  returns (stream ExpandedWatchResponse)

message ExpandedWatchRequest {
    string store_id = 1;                // required
    string authorization_model_id = 2;  // defaults to 'latest' if omitted
    string type = 3;                    // required
    string relation = 4;                // required
    
    string continuation_token = 5;      // if omitted, ReadChanges from the beginning
}

message ExpandedWatchResponse {
    RelationshipStatusUpdate relationship_update = 1;
    
    string continuation_token = 5;
}

message RelationshipStatusUpdate {
    enum RelationshipStatus {
        HAS_RELATIONSHIP = 1;
        NO_RELATIONSHIP = 2;
    }
    
    Object object = 1;
    string relation = 2;
    string user_id = 3;
    
    RelationshipStatus relationship_status = 4;
}

message Object {
    string type = 1;
    string id = 2;
}
```

### ConnectedObjects API (internal)

### ExpandUsers API (internal)
**Summary**: Given a user or userset (e.g. object#relation), ExpandUsers will return all of the user ids (direct or indirect) that the userset expands to. This is a recursive form of the existing Expand API on the provided userset.

When a tuple change is received from the [ReadChanges API][read-changes], then the ExpandUsers API will be used to compute the user ids that the tuple change could impact.

For example, consider the following relationship tuples:
| object            | relation | user                 |
|-------------------|----------|----------------------|
| group:engineering | member   | group:openfga#member |
| group:engineering | member   | jim                  |
| group:openfga     | member   | alberto              |
| group:openfga     | member   | bob                  |

Calling `ExpandUsers(group:engineering#member)` would return the list `['jim', 'alberto', 'bob']`. Calling `ExpandUsers(jill)` would just return the list list `['jill']`, because a user id doesn't expand to a set of users.

# Migration
[migration]: #migration

This section should document breaks to public API and breaks in compatibility due to this RFC's proposed changes. In addition, it should document the proposed steps that one would need to take to work through these changes. Care should be give to include all applicable personas, such as application developers, authorization platform operators, DevSecOps users and end users whose lives depend on the authorization system.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

- **What other designs have been considered?**
- **Why is this proposal the best?**
- **What is the impact of not doing this?**

# Prior Art
[prior-art]: #prior-art

Discuss prior art, both the good and bad.

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

- **What parts of the design do you expect to be resolved before this gets merged?**
- **What parts of the design do you expect to be resolved through implementation of the feature?**
- **What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?**

[1]: https://openfga.dev/docs/interacting/search-with-permissions#option-2-build-a-local-index-from-changes-endpoint-search-then-check
[read-changes]: https://openfga.dev/api/service#/Relationship%20Tuples/ReadChanges