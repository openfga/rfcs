# Meta
[meta]: #meta
- **Name**: ListObjects API
- **Start Date**: 2022-07-14
- **Author(s)**: jon-whit
- **Status**: Approved <!-- Acceptable values: Draft, Approved, On Hold, Superseded -->
- **RFC Pull Request**: https://github.com/openfga/rfcs/pull/3
- **Relevant Issues**:
  - https://github.com/openfga/openfga/issues/101
  - https://github.com/openfga/api/issues/10
- **Supersedes**: N/A

# Summary
[summary]: #summary

In some scenarios like UI filtering or auditing, developers need to answer queries like 

> list all documents that user:X can read

More specifically, given an object type, relation, and user, list all of the object ids of the given object type the user has the provided relationship with.

Depending on the use case, this problem can be solved with the existing Check API, as described in our [Search with Permissions](https://openfga.dev/docs/interacting/search-with-permissions) documentation, but in some cases it is more practical to have a dedicated API to serve this functionality.

The ListObjects API addresses [Search with Permissions (Option 3)](https://openfga.dev/docs/interacting/search-with-permissions#option-3-build-a-list-of-ids-then-search). It is specifically targeting the Search with Permissions use case for small object collections (on the order of thousands). For Search with Permissions on larger collections (tens of thousands and beyond) an OpenFGA client should build an external index via the ReadChanges (e.g. Watch) API or, in the future, via the OpenFGA Indexer.

# Definitions
[definitions]: #definitions
For more information/background on commonly used terms, please see the official [OpenFGA Concepts](https://openfga.dev/docs/concepts) page.

* [Search with Permissions](https://openfga.dev/docs/interacting/search-with-permissions) - Given a particular search filter (and optional sort order), what objects can the user acces?
* [Authorization Model](https://openfga.dev/docs/concepts#what-is-an-authorization-model)
* [Relationship Tuples](https://openfga.dev/docs/concepts#what-is-a-relationship-tuple)
* [gRPC Server Streaming RPC](https://grpc.io/docs/what-is-grpc/core-concepts/#server-streaming-rpc)
* [gRPC Unary RPC](https://grpc.io/docs/what-is-grpc/core-concepts/#unary-rpc)

# Motivation
[motivation]: #motivation

- **Why should we do this?**
  It simplifies client integrations with OpenFGA for Search with Permissions and it addresses an ask from the community for an endpoint to list the objects a user has permissions for.

- **What use cases does it support?**
  Search with Permissions where the number of objects of a particular type the user could have access to is low (~ thousands), and the percentage of the total objects that the user can have access to is also low.

- **What is the expected outcome?**
  OpenFGA clients use the ListObjects API to return a list of object ids and then the client uses the list to filter and order their object list in their applications (pagination on the filtered list is the clients responsibility). The expectation is that the client consumes the whole ListObjects response before filtering client side, and does not try to partially filter based on a subset of the response from ListObjects. Hence, it is not the goal of ListObjects to be able to continue the ListObjects process from a prior invocation.

# What it is
[what-it-is]: #what-it-is

The ListObjects API will provide clients with the ability to query the object ids that a user has a particular relationship with. The list of object ids the ListObjects API returns can then be used by a client to filter an object collection to only the objects that a user has a particular relationship with by taking the intersection of it with any client provided filters/ordering of the underlying dataset. This is described in more detail with [Search with Permission (Option 3)](https://openfga.dev/docs/interacting/search-with-permissions#option-3-build-a-list-of-ids-then-search) document.

It will be provided as two different API endpoints: one will provide a streaming endpoint and the other will implement the endpoint without streaming. These two different endpoints could be used interchangably, but clients can choose which one may be better for their integration environment.

Consider a developer/client integrating with OpenFGA with the following authorization model:
```
type folder
    relations
        define viewer as self

type document
    relations
        define viewer as self or editor or viewer from parent
        define editor as self
        define parent as self
```
and the following relationship tuples:

| object         | relation | user           |
|----------------|----------|----------------|
| document:doc1  | viewer   | bob            |
| document:doc2  | editor   | bob            |
| document:doc3  | parent   | folder:folder1 |
| folder:folder1 | viewer   | bob            |

## StreamedListObjects
StreamedListObjects will implement a grpc [server streaming RPC](https://grpc.io/docs/what-is-grpc/core-concepts/#server-streaming-rpc) endpoint.

The endpoint will stream each object id the target user has the given relationship with as it is evaluated, and the server will continuing streaming results until either the list has been exhausted, the `listObjects-max-results` has been reached, or the `listObjects-deadline` has been reached (see [Configuration Changes](#configuration-changes)).

Sample grpcurl command:
```
grpcurl -plaintext \
  -d '{
           "store_id":"<store>", 
           "authorization_model_id":"<modelID>",
           "type":"document", 
           "relation":"viewer", 
           "user":"bob"
      }' \
  localhost:8081 openfga.v1.OpenFGAService/StreamedListObjects

{
    "object_id": "doc1",
}
{
    "object_id": "doc2"
}
{
    "object_id": "doc3"
}
```

Sample curl command:
```
curl --request POST 'http://localhost:8080/stores/<storeID>/streamedListObjects' \
--header 'Content-Type: application/json' \
--data-raw '{
    "type": "document",
    "relation": "viewer",
    "user": "bob",
    "authorization_model_id": "<modelID>"
}'

{"result":{"object_id":"doc1"}}
{"result":{"object_id":"doc2"}}
{"result":{"object_id":"doc3"}}
```

## ListObjects
ListObjects will implement a grpc [unary RPC](https://grpc.io/docs/what-is-grpc/core-concepts/#unary-rpc) endpoint.

The endpoint will return a list of the object ids the target user has the given relationship with. The size of the list that is returned will be determined by the `listObjects-max-results` config (see [Configuration Changes](#configuration-changes)). If no maximum is provided by that config, then ListObjects will attempt to accumulate all of the object ids before returning them. 
> ⚠️ Without proper configuration of, and use of the ListObjects endpoint, the server's memory footprint could be profoundly impacted, because the list is accumulated in memory before it is returned.

Sample grpcurl command:
```
grpcurl -plaintext \
  -d '{
        "store_id":"<store>",
        "authorization_model_id":"<modelID>",
        "type":"document",
        "relation":"viewer",
        "user":"bob",
        "contextual_tuples": {
          "tuple_keys': [
            {
              "object": "document:doc4",
              "relation": "viewer",
              "user": "bob"
            }
          ]
        }
      }' \
  localhost:8081 openfga.v1.OpenFGAService/ListObjects

{
    "object_ids": [
        "doc1",
        "doc2",
        "doc3",
        "doc4
    ]
}
```

Sample curl command:
```
curl --request POST 'http://localhost:8080/stores/<storeID>/listObjects' \
--header 'Content-Type: application/json' \
--data-raw '{
    "authorization_model_id": "<modelID>"
    "type": "document",
    "relation": "viewer",
    "user": "bob",
    "contextual_tuples": {
      "tuple_keys': [
        {
          "object": "document:doc4",
          "relation": "viewer",
          "user": "bob"
        }
      ]
    }
}'

{
    "object_ids": [
        "doc1",
        "doc2",
        "doc3",
        "doc4"
    ]
}
```

The results from the ListObjects query can then be used to filter a collection by taking the intersection of these object ids with objects returned from the clients own query.

# How it Works
[how-it-works]: #how-it-works

The `StreamedListObjects` endpoint will be (gRPC server stream) will be mapped to it's HTTP/json equivalent through the mapping provided by the implementation of the grpc-gateway project, which converts the stream into an [http/1.1 chunked transfer encoding](https://en.wikipedia.org/wiki/Chunked_transfer_encoding). Consumers of the streaming HTTP endpoint will be able to consume it using chunked-transfer encoding.

The `ListObjects` endpoint (unary gRPC endpoint) will be mapped to its HTTP/json equivalent through the mapping provided by the implementation of the [grpc-gateway](https://grpc-ecosystem.github.io/grpc-gateway/) project.

## API Changes
The following is the proposed patch diff of the protobuf API changes to [openfga/api](https://github.com/openfga/api):
```
diff --git a/openfga/v1/openfga_service.proto b/openfga/v1/openfga_service.proto
index 4140d56..d0056cb 100644
--- a/openfga/v1/openfga_service.proto
+++ b/openfga/v1/openfga_service.proto
@@ -661,6 +661,48 @@ service OpenFGAService {
       description: "Returns a paginated list of OpenFGA stores."
     };
   }
+
+  // StreamedListObjects is a streaming variation of the ListObjects API (see ListObjects for more info).
+  rpc StreamedListObjects(ListObjectsRequest) returns (stream StreamedListObjectsResponse) {
+    option (google.api.http) = {
+      post: "/stores/{store_id}/streamedListObjects",
+      body: "*"
+    };
+  }
+
+  // ListObjects lists all of the object ids for objects of the provided type that the given user
+  // has a specific relation with.
+  rpc ListObjects(ListObjectsRequest) returns (ListObjectsResponse) {
+    option (google.api.http) = {
+      post: "/stores/{store_id}/listObjects",
+      body: "*"
+    }
+  }
+}
+
+message ListObjectsRequest {
+    string store_id  = 1 [
+      json_name = "store_id"
+    ];
+
+    string authorization_model_id = 2 [
+      json_name = "authorization_model_id"
+    ];
+
+    string type = 3 [
+      json_name = "type"
+    ];
+
+    string relation = 4;
+    string user = 5;
+
+    openfga.v1.ContextualTupleKeys contextual_tuples = 6 [
+      json_name = "contextual_tuples"
+    ];
+}
+
+message ListObjectsResponse {
+  repeated string object_ids = 1;
+}
+
+message StreamedListObjectsResponse {
+  string object_id = 1;
+}
```

`store_id`, `type`, `relation`, and `user` will be required fields. If the `authorization_model_id` is not provided it will default to the latest model.

The streaming variant accepts the same request body but differs in its response structure. For the streaming RPC a single `StreamedListObjectsResponse` is streamed back per result, whereas with the unary variant an array of object ids is accumulated and then returned.

## Configuration Changes
- Add a `ListObjectsMaxResults` config param (`--listObjects-max-results` flag)

  This will be used to define the maximum number of results that will be returned by the streaming or unary ListObjects API endpoints. If this config is omitted then the server
  will not limit the response to a finite number of results.

- Add a `ListObjectsDeadline` config param (`--listObjects-deadline` flag)

  This will be used to define the maximum amount of time to accumulate ListObjects results until the server responds. If this config is omitted then the server will
  wait indefinitely to resolve the list of all objects.

If both the `ListObjectsMaxResults` and `ListObjectsDeadline` configs are provided, then whichever condition is met first wins. For example, if the deadline is 5s and the limit is
10, then if 10 results are available before 5s the server will respond at that moment. If the deadline is 5s and the limit is 10, then if only 5 results are available when
the deadline is reached then the server will respond at that moment.

For production usage we should document recommendations for these configuration limits so as to protect the server from memory exhaustion or costly ListObjects queries.

## Server Changes
* Add implementations of the API changes described in [Changes to the OpenFGA API](#changes-to-the-openfga-api) in the server. This includes implementations:
```
func (s *Server) StreamedListObjects(
    req *openfgapb.ListObjectsRequest,
    srv openfgapb.OpenFGAService_StreamedListObjectsServer,
) error {...}

func (s *Server) ListObjects(
    ctx context.Context,
    req *openfgapb.ListObjectsRequest,
) (*openfgapb.ListObjectsResponse, error) {...}
```

Sample code implementing these proposed changes can be found here:
https://github.com/openfga/openfga/tree/listObjects-poc (server changes) which depends on https://github.com/openfga/api/tree/listObjects-api (protobuf API changes)

### Storage Interface Changes
The [`OpenFGADatastore`](https://github.com/openfga/openfga/blob/170a6834d057428e3b0d250cae47a01f5a61898f/storage/storage.go#L132) interface will need to be expanded or modified to support queries for all objects of a particular object type (within a store). Today our datastore/storage interface only has the [`Read`](https://github.com/openfga/openfga/blob/170a6834d057428e3b0d250cae47a01f5a61898f/storage/storage.go#L51) method, but it requires a storeID and tuple key specifying either the `object` or `user` field (or both). The ListObjects work will need a method that supports only providing the storeID and object `type`.

The [`TupleBackend`](https://github.com/openfga/openfga/blob/170a6834d057428e3b0d250cae47a01f5a61898f/storage/storage.go#L47) interface will need a new method and it's signature could look like:

```
ReadRelationshipTuples(
  ctx context.Context,
  filter ReadRelationshipTuplesFilter,
  opts ...QueryOption,
) (TupleIterator, error)

type ReadRelationshipTuplesFilter struct {
  StoreID            string
  OptionalObjectType string
  OptionalObjectID   string
  OptionalRelation   string
  OptionalUserFilter string
}

type QueryOptions struct {
  Limit    uint64
}

type QueryOption func(*QueryOptions)
```

This approach will allow us to query relationship tuples using any provided filters, and could serve as a more general replacement for the existing Read method.

# Migration
[migration]: #migration

No migrations or breaking changes are necessary to support the ListObjects API endpoints described herein.

# Drawbacks
[drawbacks]: #drawbacks

Potential arguments for not introducing the ListObjects API include:
* Without proper limitations clients could impact the whole system by issuing ListObjects on large collections.
* Increases the OpenFGA API surface for a very niche use case for Search with Permissions on small object collections.
* Potentially encourages users to use ListObjects over other methods that may be better applied for their use-case (search then check or local index).

# Alternatives
[alternatives]: #alternatives

## More Generic ListObjects API
An alternative version of ListObjects could support both generic and specific queries such as:
> list all documents that user:bob can 'read' or 'write'

For example,
```
curl --request POST 'http://localhost:8080/stores/<storeID>/listObjects' \
--header 'Content-Type: application/json' \
--data-raw '{
    "type": "document",
    "relations": ["read", "write"],
    "user": "bob",
    "authorization_model_id": "<modelID>"
}'
```
and

> list all relationships for documents that user:bob has

For example,
```
curl --request POST 'http://localhost:8080/stores/<storeID>/listObjects' \
--header 'Content-Type: application/json' \
--data-raw '{
    "type": "document",
    "user": "bob",
    "authorization_model_id": "<modelID>"
}'
```

This alternative ListObjects API definition is more generic and could be more generally applicable to specific cases where you want to know all of the objects (of a specific type) and the relationships to those objects a user has. However, the query cost of supporing a generic API endpoint like this could be quite expensive since you may have to evaluate all object and relation pairs for objects of a given type.

The stricter variant of ListObjects that requires a single relationship protects the server a little more while giving control to the adopter to choose to how rate limit and throttle invocations of the ListObjects endpoint. For these reasons it is recommended that ListObjects will only serve queries such as those constrained to a specific object type, single relation, and single user.

## Paginated ListObjects API
ListObjects could be implemented as a paginated list instead of the streaming and/or static list approach proposed. For example, following along with the prior example,

```
curl --request POST 'http://localhost:8080/stores/<storeID>/listObjects' \
--header 'Content-Type: application/json' \
--data-raw '{
    "authorization_model_id": "<modelID>"
    "type": "document",
    "relation": "viewer",
    "user": "bob",
    "contextual_tuples": {
      "tuple_keys': [
        {
          "object": "document:doc4",
          "relation": "viewer",
          "user": "bob"
        }
      ]
    },
    "page_size": 2
}'

{
    "object_ids": [
        "doc1",
        "doc2"
    ],
    "continuation_token": "YmxhaAo="
}

curl --request POST 'http://localhost:8080/stores/<storeID>/listObjects' \
--header 'Content-Type: application/json' \
--data-raw '{
    "authorization_model_id": "<modelID>"
    "type": "document",
    "relation": "viewer",
    "user": "bob",
    "contextual_tuples": {
      "tuple_keys': [
        {
          "object": "document:doc4",
          "relation": "viewer",
          "user": "bob"
        }
      ]
    },
    "page_size": 2,
    "continuation_token": "YmxhaAo="
}'

{
    "object_ids": [
        "doc3",
        "doc4"
    ],
    "continuation_token": ""
}
```

The advantage of this approach is that we already have a handful of paginated APIs in the OpenFGA API, but the challenge with this is supporting pagination with graph traversal.

The existing APIs we have that support pagination require no graph traversal to resolve the request, and are easily paginated with a database filter and sort approach. Paginating ListObjects, like Expand, is challenging because we'd have to somehow encode the path of the graph that we had already visited and continue paginating from the subpath in the graph of relationships where the last request left off. To do this we'd have to change the way we traverse the graph as well, because we'd have to traverse the graph in a deterministic (and ordered) way, which would also limit the ability to concurrently evaluate subpaths of the graph in parallel, thus significantly hurting performance. This issue may be something we discover an appropriate solution for in the future, but at this time it is not obvious how to best solve that problem with performance in mind, and an implementation of such a pattern would be challenging at this time. The complexity of such an implementation would likely hold us back from getting some quicker feedback on the usability of ListObjects in general. If the community has any recommandations/ideas on how to approach this problem we'd love to hear it!

The paginated approach described above also doesn't seem to provide much value given the semantics and limitations of the ListObjects API as described earlier. Because ListObjects is designed for Search with Permissions on small object collections, the intended usage of the API is to never exceed a couple of thousand results in the response, and thus implementing pagination ontop of it seems like overkill. What do you think?

# Prior Art
[prior-art]: #prior-art

## Other Zanzibar-inspired Implementations
* [Oso](https://www.osohq.com/) has a [`List` endpoint](https://cloud-docs.osohq.com/reference/client-apis/api-explorer#/default/post_list)
  The Oso List API requires `actor_type`, `actor_id`, `action`, and `resource_type` as inputs. So they limit the query to resources for a specific actor and action, which is similar to the primary proposal herein.

  The Oso List API does not implement pagination.

* [authzed/spicedb](https://github.com/authzed/spicedb) has a [`LookupResources` endpoint](https://app.swaggerhub.com/apis-docs/authzed/authzed/1.0#/PermissionsService/PermissionsService_LookupResources)
  The authzed/spicedb LookupResources API requires the `resourceObjectType`, `permission`, and `subject` (subject can be a userset such as 'group:group1#member'), so that is also similar to the primary proposal herein.

  The LookupResources API is a gRPC server streaming API. It does not support pagination.

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

- **What parts of the design do you expect to be resolved before this gets merged?**
  The design (not necessarily the implementation) of the changes to the API surface.

- **What parts of the design do you expect to be resolved through implementation of the feature?**
  The finer details of the changes needed for the `OpenFGADatastore` storage interface.

- **What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?**
  None at this time.
